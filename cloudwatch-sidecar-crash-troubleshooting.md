# 🧨 CloudWatch Agentの「静かな崩壊」によりECSタスクが定期的に再起動される問題

## 🎯 実際の状況

私たちはECS Fargate上でPython Djangoバックエンドシステムを運用しており、NGINXを前面に配置してルーティングとキャッシュを行っています。最近、システムの可観測性（observability）を向上させるため、サイドカーコンテナcwagentを通じてCloudWatch Application Monitoringを統合し、同時にadot-auto-instrumentation-pythonによる自動計装を適用してSLI/SLO指標のトレースと監視を行いました。

しかし、本番環境にデプロイした後、かなり厄介な動作に遭遇し始めました：

- ECSタスクが実行開始から6〜7時間後に突然再作成される (定期的ではない)
- cwagentコンテナが素早く再起動する → ALBでECSはタスクをhealthyとマークしたまま
- アラートが発動しない — 手動でdeployment infoを確認するまで問題を発見できなかった

## 🔍 原因調査

各コンテナからのログを詳しく調査した結果、cwagentから深刻なエラーメッセージを発見しました：

```go
fatal error: concurrent map writes
```

### 😱 Goの古典的なエラー

これはGoで書かれたアプリケーション（cwagentもその一つ）で非常に一般的なランタイムパニックです。複数のgoroutineが同期（sync.Mapやlock）なしに同じmapに書き込むと発生します。Goはメモリ破損（memory corruption）を避けるため、プログラム全体をkillします。

### 📈 主な原因

- アプリケーションが特に自動計装を有効にした際に、過度に多くのログ/トレースを送信
- cwagentが大量のログ/メトリクスを同時に処理する必要がある → レースコンディションが発生
- ログにInternalOperationの行が大量に記録され、3分間に9,000リクエストに達するが、システムにそのような外部トラフィックはない

### OTEL設定からの追跡

ECSタスクの設定を確認した際、システムが以下のサンプラーを使用していることがわかりました：

```json
{
  "name": "OTEL_TRACES_SAMPLER",
  "value": "xray"
},
{
  "name": "OTEL_TRACES_SAMPLER_ARG",
  "value": "endpoint=http://localhost:2000"
}
```

xrayサンプラーはほぼすべてのリクエストをトレースします — 以下を含む：

- 内部ジョブ（cron、queue worker、healthcheck）
- フレームワークの内部リトライ
- コネクションプール操作

これらのトレースすべてがcwagentに送信され、オーバーロードを引き起こし、エージェント内部でパニックを引き起こします。

## 解決策：より合理的なサンプリング

### 🎯 明確な目標

私たちは実際のユーザーからのSLI/SLOのみを観察したい — 例：API応答時間、主要エンドポイントでのエラー率など。

### ✅ サンプリングの調整：

```json
{
  "name": "OTEL_TRACES_SAMPLER",
  "value": "parentbased_traceidratio"
},
{
  "name": "OTEL_TRACES_SAMPLER_ARG",
  "value": "0.01"
}
```

`parentbased_traceidratio`はサンプラー：

- 親spanのトレース決定がある場合はそれを尊重
- 親がない場合 → 事前設定された比率（ここでは1%）でルートspanをサンプリング

### 💡 なぜparentbased_traceidratioを使うべきか？

マイクロサービスシステムでは、エンドユーザーからのエントリーポイントリクエストは通常`GET /api/products`、`POST /auth/login`などのAPIです。healthcheckやcronjobなどの内部タスクは通常実際のSLIに貢献しません。

`parentbased_traceidratio`サンプラーを使用することで：

- 内部トレース（InternalOperation、cronjobなど）からのノイズを削減
- トレース保存のリソースを節約
- ユーザーからの意味のあるリクエストの完全なトレースを保持

### ✅ 変更後の結果

| 変更前 | 変更後 |
|--------|--------|
| cwagentが6〜7時間後に定期的にクラッシュ | 安定、パニックなし |
| ログに多くのInternalOperation | ログがクリーン、外部APIのみ含む |
| ECSタスクが静かに再作成 | タスクが安定して実行 |
| 内部トレースによるSLI観察のノイズ | エンドユーザーからのトレースに正しくフォーカス |

## 📊 比較：xray vs parentbased_traceidratio

| 基準 | xray | parentbased_traceidratio |
|------|------|-------------------------|
| 目的 | ほぼすべてのspanをトレース | エンドユーザーからの比率に基づいてトレース |
| トレース範囲 | internal/backgroundも含む | 主にサンプリングされたルートspanを持つトレース |
| ルートspanサンプリング | かなり複雑なルールに基づく | 比率（0.01）によるランダムサンプリング |
| 子spanサンプリング | 親からのサンプリングを継承しない | 親spanのサンプリングに従う |
| リソース節約 | ❌ リソースを消費 | ✅ 非常に効率的 |
| いつ使用するか？ | 包括的なデバッグ、テスト | 本番環境、ユーザーリクエストの監視 |
| 内部操作のフィルタリング | フィルタリングが困難 | 自然に除去 |
| 設定の複雑さ | 高い | 低い、制御しやすい |

## 🤔 学んだ教訓
- 必要ない場合はすべてをトレースすべきではない。明確な目標を持つ：SLI/SLOか詳細なデバッグか？
- CloudWatch Agentは「無敵」ではない — トレースによるオーバーロードでサイドカーもボトルネックになる可能性がある

## 📚 参考資料

- [OpenTelemetry Sampling for Python](https://opentelemetry.io/docs/languages/python/sampling/)
- [AWS X-Ray Sampling Rules](https://docs.aws.amazon.com/xray/latest/devguide/xray-console-sampling.html)
- [CloudWatch Application Signals - Troubleshoot InternalOperation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Signals-Troubleshoot-InternalOperation.html)
- [ECS Sidecar for CloudWatch Application Signals](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Signals-ECS-Sidecar.html) 