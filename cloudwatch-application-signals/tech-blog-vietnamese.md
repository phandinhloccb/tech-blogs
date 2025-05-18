# Thiết lập và Monitoring SLO/SLI với AWS CloudWatch Application Signals

## Tổng quan

Việc giám sát hệ thống AWS thường chỉ tập trung vào các chỉ số cơ bản như lỗi 5xx ở Load Balancer hoặc API Gateway. Tuy nhiên, khi hệ thống phát triển, việc thiết lập SLO (Service Level Objectives) và SLI (Service Level Indicators) trở nên cần thiết để đảm bảo chất lượng dịch vụ và phát hiện sớm các vấn đề.

Bài viết này hướng dẫn thiết lập SLO/SLI cho hệ thống serverless sử dụng AWS CloudWatch Application Signals - giải pháp monitoring tích hợp trong hệ sinh thái AWS.

## Ưu điểm của CloudWatch Application Signals

So với các giải pháp SaaS như Grafana, CloudWatch Application Signals mang lại những lợi ích:

- Bảo mật cao do không chia sẻ dữ liệu với bên thứ ba
- Quản lý tập trung trong môi trường AWS
- Tích hợp sẵn với các dịch vụ AWS khác

## Kiến trúc hệ thống mẫu
đây là kiến trúc tôi dùng để demo trong blog này. chi tiết hệ thống thì nằm ở link phía dưới.

![Kiến trúc hệ thống](architecture_diagram.png)

Chi tiết về kiến trúc: [Qitta Blog](https://qiita.com/phandinhloccb/items/3368c44c68999e64f736)

## Thiết lập mục tiêu SLO/SLI

Với hệ thống serverless gồm 3 Lambda services, chúng ta đặt mục tiêu:

| Thành phần | Giá trị |
|------------|---------|
| Thời gian đánh giá SLO | 1 giờ (rolling) |
| Phương pháp tính toán | Requests |
| Mục tiêu SLO | 95% request thành công |
| Ngưỡng cảnh báo | 30% của error budget |
| Lượng request/giờ | ~200 |

### Phân tích chi tiết

**SLO 95% thành công:**
- Cho phép tối đa 5% request lỗi
- Với 200 request/giờ: tối đa 10 request lỗi được phép

**Error Budget:**
- Tổng error budget = 5% = 10 lỗi/giờ
- Ngưỡng cảnh báo = 30% của 10 lỗi = 3 lỗi
- Khi số lỗi ≥ 3: SLO ở trạng thái "Warning"
- Khi số lỗi > 10: SLO không đạt (Unhealthy)

## Triển khai thực tế

### 1. Kích hoạt Application Signals

Trong cấu hình Lambda, tại phần **Monitoring and operations tools**, bật tính năng **Application Signals**:

![Kích hoạt Application Signals](enable-application-signal.png)

Lambda sẽ tự động tạo policies cần thiết và gắn cho role của Lambda để push metric. Sau khi kích hoạt, chờ hệ thống thu thập dữ liệu và hiển thị dashboard:

![Dashboard Application Signals](cloudwatch-application-signals.png)

### 2. Thiết lập SLI

![Thiết lập SLI](set-sli.png)

#### So sánh phương pháp tính SLI

| Tiêu chí | Request-based | Time-based (Periods) |
|----------|---------------|----------------------|
| Phản ánh trải nghiệm người dùng | ✅ Chính xác cao | 🔸 Kém chính xác với traffic cao |
| Khả năng chịu lỗi | ❌ Nhạy cảm với lỗi nhỏ | ✅ Chịu được lỗi tạm thời |
| Phù hợp với traffic thấp | ❌ Không phù hợp | ✅ Rất phù hợp |
| Phát hiện lỗi kéo dài | ❌ Kém hiệu quả | ✅ Hiệu quả cao |
| Dễ hiểu | ✅ Dễ hiểu (995/1000) | 🔸 Khó hiểu hơn (9/12 periods) |

#### Ví dụ minh họa sự khác biệt

Giả sử hệ thống trong 3 phút có:

| Thời gian | Tổng request | Request lỗi | Trạng thái phút |
|-----------|--------------|-------------|----------------|
| 10:00–10:01 | 10,000 | 300 | ✅ Good (3% lỗi) |
| 10:01–10:02 | 10,000 | 200 | ✅ Good (2% lỗi) |
| 10:02–10:03 | 10,000 | 600 | ✅ Good (6% lỗi - giả sử ngưỡng 10%) |
| **Tổng** | **30,000** | **1,100** | **SLI theo Period: 100% Good** <br> **SLI theo Request: 96.3% Good** |

**Kết quả:**
- Theo Periods: Hệ thống hoàn hảo (3/3 phút đạt)
- Theo Requests: Có 1,100 lỗi (3.7%) - phản ánh trải nghiệm thực tế của người dùng

#### Khi nào nên dùng Time-based (Periods)

Periods phù hợp khi **traffic thấp** (dưới vài trăm request/giờ):

**Với 200 request/giờ:**
- 1 lỗi = 0.5% error rate
- 2 lỗi = 1% error rate → dễ vi phạm SLO nếu dùng Request-based

**Với Period-based:**
- 1 giờ chia thành 60 periods (1 phút/period)
- Nếu chỉ 1 phút có lỗi, còn 59 phút tốt → SLO = 59/60 = 98.3% (vẫn đạt)

Periods giúp hệ thống ít bị ảnh hưởng bởi các lỗi ngắn hạn, phù hợp với môi trường traffic thấp.

### 3. Thiết lập SLO

![Thiết lập SLO](set-slo.png)

### 4. Thiết lập cảnh báo (Alarms)

![Thiết lập cảnh báo](set-alarm.png)

CloudWatch Application Signals hỗ trợ 3 loại cảnh báo:

1. **SLI Health alarm**: Cảnh báo khi SLI không đạt ngưỡng theo thời gian thực
2. **SLO attainment goal alarm**: Cảnh báo khi không đạt mục tiêu SLO
3. **SLO warning alarm**: Cảnh báo khi tiêu thụ quá nhiều error budget

**SLI Health alarm** dùng dữ liệu theo sliding window:
- AWS sử dụng cửa sổ thời gian ngắn (thường 1-5 phút)
- Tính tỷ lệ good requests/total requests trong cửa sổ gần nhất
- So sánh với mục tiêu SLO đã đặt

khi setting alarm thì AWS sẽ tự dộng tạo các cloudawtch alarm.


## Kết Quả 
Đọc hiểu báo cáo SLO
![Kết quả theo dõi SLO](result.png)

| Trường | Giá trị | Ý nghĩa |
|--------|---------|---------|
| SLO name | response-slack-slo | Tên SLO được đặt |
| Goal | 95% | Mục tiêu: 95% request thành công |
| SLI status | Unhealthy | SLI không đạt mục tiêu |
| Latest attainment | 92.5% | Tỷ lệ thành công hiện tại (< 95%) |
| Error budget | 1 requests over budget | Vượt quá ngân sách lỗi cho phép |
| Error budget delta | -25% | Tiêu thụ thêm 25% ngân sách lỗi so với trước |
| Time window | 1 hour rolling | Đánh giá liên tục trong 1 giờ gần nhất |

✅ SLO Goal: 95%
Đây là mục tiêu bạn đặt ra: ít nhất 95% request phải đạt tiêu chí thành công (ví dụ: HTTP 2xx, thời gian phản hồi < 1s...).
Nếu chỉ đạt dưới 95%, hệ thống được xem là vi phạm SLO.

🚦 SLI status: Unhealthy
Nghĩa là: Dựa vào dữ liệu trong 1 giờ vừa qua, hệ thống không đạt được SLO.
Đây là trạng thái sức khoẻ SLI, chứ không phải "lỗi phần mềm".

📊 Latest attainment: 93.9%
Trong vòng 1 giờ gần nhất (rolling window), bạn chỉ đạt được 93.9% yêu cầu thành công.
Điều này < 95%, tức là không đạt yêu cầu SLO.

❗ Error budget: 1 requests over budget
Bạn đã vượt quá giới hạn lỗi được phép theo SLO.
Nếu SLO cho phép 5% lỗi trong 1000 request (tức là 50 lỗi), thì:
Bạn đã gặp 51 lỗi → 1 request vượt ngân sách lỗi.
Điều này chính thức là vi phạm SLO.

🔻 Error budget delta: -25%
Đây là sự thay đổi so với thời điểm trước:
Nghĩa là bạn đã tiêu tốn thêm 25% error budget (so với chu kỳ trước hoặc snapshot trước).
Có thể là do một đợt lỗi tăng đột biến gần đây.
Mục đích của delta là phát hiện xu hướng xấu đi nhanh chóng.

🕐 Time window: 1 hour rolling
SLO được tính theo cửa sổ thời gian trượt liên tục 1 giờ gần nhất (rolling).
Cứ mỗi phút, hệ thống sẽ nhìn lại 60 phút trước đó để tính toán lại tất cả chỉ số.

## Monitoring chi tiết
### Theo dõi request cụ thể

Ngoài monitoring tổng thể, có thể thiết lập SLO cho từng loại request:
Ví dụ: Trong service read-rag, có thể thiết lập SLO riêng cho các request tới Bedrock và OpenSearch dựa trên latency hoặc tỷ lệ lỗi.
![Monitoring request cụ thể](slo-for-specific-request.png)

### Service Map

![Service Map](service-map-2.png)
Service Map giúp quan sát toàn cảnh hệ thống và xác định các service không đạt SLO:

| Chỉ số | Giá trị |
|--------|---------|
| Requests | 9 |
| Avg latency | 655.9 ms |
| Error rate | 0% |
| Fault rate | 0% |

#### So sánh Service Map và Trace Map (X-ray)

| Tiêu chí | Service Map | Trace Map |
|----------|-------------|-----------|
| Mục tiêu | Tổng quan hệ thống và mối quan hệ | Chi tiết một request từ đầu đến cuối |
| Phạm vi | Toàn hệ thống | Request đơn lẻ |
| Công dụng | - Tìm bottleneck hệ thống<br>- Xem tương tác giữa services<br>- Giám sát sức khỏe tổng thể | - Debug một lỗi cụ thể<br>- Phân tích latency chi tiết<br>- Theo dõi luồng xử lý |
| Hiển thị | Đồ thị services và kết nối | Timeline hoặc cây span |
| Dữ liệu | Nhiều trace + metric tổng hợp | Một trace duy nhất |
| Ví dụ | Service A → B → C với B có latency cao | Trace ID xyz: API → Lambda → DynamoDB → S3 |

## Kết luận

AWS CloudWatch Application Signals là giải pháp hiệu quả cho hệ thống AWS, giúp monitoring và cảnh báo khi không đạt SLO/SLI mà không cần sử dụng công cụ bên thứ ba.