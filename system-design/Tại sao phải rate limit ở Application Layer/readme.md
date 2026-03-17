Vì sao phải có cả 2?
Nginx rate limit hoạt động ở tầng network — nó chỉ thấy IP address. Nó rất giỏi chặn DDoS volumetric (1 IP bắn 10,000 request/s), nhưng hoàn toàn mù về ngữ nghĩa của request. Nó không biết đây là user A hay user B, không biết đây là endpoint /qr/confirm hay /health.
App-layer token bucket (Guard) hoạt động sau khi JWT đã được decode — nó biết chính xác là ai đang gọi, đang gọi endpoint nào, và đang dùng quota của tài khoản nào. Nó giải quyết các vấn đề mà Nginx không thể:
Blind spot của Nginx: Một attacker có thể dùng 100 JWT khác nhau (từ nhiều tài khoản stolen/free), tất cả từ 1 IP. Nginx thấy IP đó vẫn nằm trong ngưỡng cho phép, nên pass hết. App guard thấy từng token đang bị abuse và chặn chính xác từng account.
Fairness giữa users: Nginx rate limit là shared — nếu user A (legitimate) ở cùng IP với user B (attacker, ví dụ cùng office, cùng VPN), user A bị vạ lây. App guard limit theo JWT nên hoàn toàn isolated.
Business logic: Nginx không thể biết /qr/confirm cần limit 5 req/phút còn /health thì không cần limit. Guard làm được điều này vì nó chạy trong context của NestJS router.
Multi-pod environment: Khi bạn scale NestJS lên nhiều pod (đúng với setup Docker Compose + Nginx upstream của bạn), state của Nginx chỉ local trên container đó. Guard dùng Redis nên counter được share across all pods — rate limit mới thực sự accurate.

Tóm lại: Nginx bảo vệ infrastructure, App guard bảo vệ business logic và từng user. Bỏ một trong hai đều tạo ra blind spot. Đây là pattern "defense in depth" — mỗi layer assume layer kia có thể bị bypass.