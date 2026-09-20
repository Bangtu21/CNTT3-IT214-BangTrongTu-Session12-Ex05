Phần 1 – Đề xuất đa giải pháp
Giải pháp 1: Sử dụng Circuit Breaker thuần túy
Giải pháp đầu tiên là sử dụng Circuit Breaker để bảo vệ Order-Service khi gọi sang GHTK. Circuit Breaker sẽ theo dõi các request và tỷ lệ lỗi khi giao tiếp với GHTK. Nếu số lượng lỗi vượt quá ngưỡng được cấu hình, Circuit Breaker sẽ chuyển sang trạng thái OPEN, tạm thời ngăn không cho Order-Service tiếp tục gọi sang GHTK nhằm tránh làm ảnh hưởng đến hệ thống.
Giải pháp này phù hợp hơn trong trường hợp GHTK gặp sự cố nghiêm trọng hoặc bị sập hoàn toàn. Tuy nhiên, với tình huống của bài toán, lỗi chủ yếu là Transient Failure, tức lỗi mạng xảy ra chớp nhoáng và request có thể thành công nếu thử lại ngay sau đó. Vì vậy, nếu sử dụng Circuit Breaker thuần túy, các lỗi Timeout xảy ra trong thời gian ngắn có thể khiến Circuit Breaker mở mạch trong khi GHTK vẫn có khả năng xử lý request bình thường.

Giải pháp 2: Sử dụng Retry Pattern kết hợp Exponential Backoff
Giải pháp thứ hai là sử dụng Retry Pattern kết hợp với Exponential Backoff. Khi Order-Service gọi GHTK và gặp lỗi tạm thời như TimeoutException, hệ thống sẽ tự động thực hiện lại request thay vì lập tức ngắt mạch.
Exponential Backoff giúp hệ thống tạo khoảng thời gian chờ giữa các lần retry và khoảng thời gian này có thể tăng dần. Cách tiếp cận này phù hợp với lỗi mạng chập chờn vì GHTK có thể hoạt động trở lại sau một khoảng thời gian ngắn. Việc retry có giới hạn cũng giúp tránh việc Order-Service gửi quá nhiều request liên tiếp đến GHTK.

Phần 2: So sánh

| Tiêu chí                   | Circuit Breaker                                       | Retry + Exponential Backoff                       |
| -------------------------- | ----------------------------------------------------- | ------------------------------------------------- |
| Mục đích chính             | Ngăn không cho hệ thống tiếp tục gọi service đang lỗi | Thử lại request khi lỗi tạm thời                  |
| Transient Failure          | Có thể không tối ưu nếu lỗi chỉ xảy ra chớp nhoáng    | Phù hợp                                           |
| System Crash               | Phù hợp                                               | Có thể làm tăng thêm request vào service đang sập |
| Cách xử lý lỗi             | Có thể chuyển sang `OPEN`                             | Thực hiện lại request                             |
| Khả năng phục hồi request  | Không trực tiếp thử lại request ngay                  | Có                                                |
| Tác động đến hệ thống lỗi  | Giảm request khi Circuit mở                           | Có thể tạo thêm request                           |
| Nguy cơ gây quá tải        | Thấp khi Circuit đã OPEN                              | Có nếu retry quá nhiều                            |
| Cấu hình thời gian chờ     | `waitDurationInOpenState`                             | Backoff giữa các lần retry                        |
| Trường hợp sử dụng phù hợp | Service downstream bị lỗi hoặc sập                    | Lỗi mạng tạm thời, timeout ngắn hạn               |
| -------------------------- | ----------------------------------------------------- | ------------------------------------------------- |

Bẫy dữ liệu – Idempotency

Retry Pattern có thể gây ra rủi ro nếu API của GHTK thực hiện các thao tác có tác dụng phụ, chẳng hạn như trừ tiền tài khoản trên mỗi lần gọi API.
Ví dụ, Order-Service gửi một request đến GHTK để tạo vận đơn và GHTK đã thực hiện việc trừ tiền, nhưng response trả về bị `TimeoutException`. Do Order-Service không biết request trước đó đã được xử lý thành công hay chưa, hệ thống sẽ thực hiện Retry. Nếu GHTK tiếp tục trừ tiền khi nhận request Retry, tài khoản của khách hàng có thể bị trừ tiền nhiều lần cho cùng một giao dịch.
Để giải quyết vấn đề này cần sử dụng Idempotency. Idempotency là tính chất đảm bảo rằng khi cùng một request được thực hiện nhiều lần thì kết quả nghiệp vụ vẫn tương đương với việc request đó chỉ được thực hiện một lần.
Một cách phổ biến là sử dụng Idempotency Key cho mỗi giao dịch. Order-Service tạo một key duy nhất cho một đơn hàng, ví dụ `ORDER-12345`, và gửi key này trong mỗi lần Retry. GHTK có thể sử dụng key để kiểm tra request đã được xử lý trước đó hay chưa. Nếu request đã được xử lý, GHTK trả lại kết quả của request trước thay vì thực hiện lại thao tác trừ tiền.
Vì vậy, khi sử dụng Retry cho các API có tác dụng phụ như trừ tiền, tạo đơn hàng hoặc tạo vận đơn**, cần đảm bảo API có cơ chế Idempotency để tránh việc một nghiệp vụ bị thực hiện nhiều lần.

