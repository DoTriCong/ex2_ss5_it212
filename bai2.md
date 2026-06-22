# BÀI 2: Tối ưu Prompt (Kỹ thuật Multiple Options, Trade-offs và Phân tích Giả định)

## 1. Phân tích vấn đề của prompt thô

Prompt thô ban đầu là:

```text
Tôi muốn viết hàm gửi email bất đồng bộ trong Spring Boot, viết code cho tôi.
```

Prompt này có nhiều hạn chế lớn nếu áp dụng vào một bài toán kiến trúc hệ thống thực tế như **Notification Service** gửi thông báo cho **1 triệu người dùng**:

### 1.1. Ép AI nhảy vào code quá sớm

Prompt đang yêu cầu **“viết code cho tôi”** ngay lập tức. Điều này khiến AI dễ chọn một cách làm quen thuộc như:

* `@Async` trong Spring Boot
* `CompletableFuture`
* tạo thread pool đơn giản

Trong khi vấn đề thực sự ở đây **không phải chỉ là viết một hàm async**, mà là **thiết kế kiến trúc gửi thông báo chịu tải lớn**.

---

### 1.2. Không yêu cầu AI so sánh nhiều phương án

Hệ thống gửi thông báo cho **1 triệu người dùng** có thể có nhiều hướng tiếp cận khác nhau, ví dụ:

* **Phương án 1:** `@Async` + thread pool trong cùng ứng dụng
* **Phương án 2:** Queue trung gian như `RabbitMQ` / `Kafka`
* **Phương án 3:** Batch job / scheduler tách riêng worker xử lý
* **Phương án 4:** Event-driven architecture với nhiều consumer scale ngang

Nếu prompt chỉ yêu cầu “viết code async”, AI thường chọn **1 công nghệ duy nhất** mà không phân tích xem nó có phù hợp tải lớn hay không.

---

### 1.3. Không đánh giá trade-off kiến trúc

Trong thực tế, khi chọn giải pháp bất đồng bộ, cần so sánh rất rõ:

* độ phức tạp triển khai;
* khả năng retry khi gửi lỗi;
* khả năng scale khi số lượng user tăng;
* nguy cơ mất message;
* độ trễ chấp nhận được;
* chi phí vận hành.

Prompt thô hoàn toàn **không yêu cầu AI lập bảng ưu/nhược điểm**, nên đầu ra rất dễ bị thiên lệch sang một giải pháp “code được” nhưng **không chắc phù hợp hệ thống lớn**.

---

### 1.4. Không có phân tích giả định tải cao (What-if Scenario)

Bài toán có bối cảnh cực rõ: **gửi cho 1 triệu người dùng khi có sự kiện khuyến mãi**.
Đây là tình huống mà kiến trúc phải chịu được:

* lưu lượng tăng đột biến;
* mail/SMS provider trả lời chậm;
* API timeout;
* queue backlog;
* retry quá nhiều gây trùng lặp;
* worker chết giữa chừng.

Prompt thô không yêu cầu AI phân tích các tình huống “nếu… thì sao”, nên rất dễ sinh ra lời giải **chạy được ở demo** nhưng **vỡ ở production**.

---

## 2. Mục tiêu của prompt tối ưu

Prompt tối ưu cần buộc AI **không được viết code ngay**, mà phải:

1. **Đề xuất ít nhất 3 phương án kiến trúc** cho Notification Service bất đồng bộ.
2. **So sánh trade-offs** giữa các phương án bằng bảng.
3. **Phân tích giả định tải cao** và rủi ro thực tế khi gửi cho 1 triệu người dùng.
4. **Đề xuất phương án phù hợp nhất** cho production.
5. Chỉ **sau khi phân tích xong** mới sinh ra **mã nguồn Java / Spring Boot skeleton** cho phương án được chọn.

Nói cách khác, prompt mới phải biến AI từ “code generator” thành **solution architect + backend engineer**.

---

# 3. Prompt tối ưu sau khi thiết kế

```text
Bạn là một **Solution Architect kiêm Senior Java/Spring Boot Developer** có kinh nghiệm thiết kế hệ thống gửi thông báo quy mô lớn.

## Bối cảnh
Tôi đang xây dựng một **Notification Service** cho hệ thống khuyến mãi. Khi có một sự kiện marketing, hệ thống cần gửi **email và SMS cho khoảng 1,000,000 người dùng**.

Nếu xử lý gửi đồng bộ ngay trong API request (synchronous), hệ thống sẽ bị nghẽn, timeout và trải nghiệm người dùng rất kém. Tôi muốn thiết kế giải pháp **asynchronous processing** trong hệ sinh thái Java / Spring Boot, nhưng chưa muốn chốt công nghệ ngay vì cần đánh giá kiến trúc tổng thể trước.

## Yêu cầu
Hãy **không viết code ngay**. Trước tiên, hãy phân tích bài toán theo 3 kỹ thuật sau:

---

## PHẦN 1 - Multiple Options
Hãy đề xuất **ít nhất 3 phương án kiến trúc khác nhau** để xử lý gửi email/SMS bất đồng bộ cho 1 triệu người dùng.  
Ví dụ, bạn có thể cân nhắc các hướng như:
- Spring Boot `@Async` + ThreadPoolTaskExecutor
- Message Queue / Event-driven (`RabbitMQ`, `Kafka`, hoặc tương đương)
- Batch processing / Scheduler + worker service
- Outbox pattern + message broker
- Tách Notification Producer và Notification Worker

Với mỗi phương án, hãy mô tả:
1. Kiến trúc tổng quan
2. Luồng xử lý request → enqueue → worker → email/SMS provider
3. Thành phần chính cần có
4. Mức độ phù hợp nếu số lượng người nhận lên đến 1 triệu

---

## PHẦN 2 - Trade-offs
Lập **bảng so sánh trade-off** giữa các phương án theo các tiêu chí sau:
- Độ dễ triển khai
- Khả năng mở rộng (scalability)
- Khả năng retry khi gửi lỗi
- Khả năng chống timeout cho API
- Độ bền dữ liệu / tránh mất thông báo
- Độ phức tạp vận hành
- Phù hợp với tải 1 triệu người dùng ở production
- Chi phí hạ tầng tương đối
- Khả năng theo dõi trạng thái gửi (sent / failed / retrying)

Sau bảng so sánh, hãy **chọn 1 phương án phù hợp nhất** và giải thích vì sao.

---

## PHẦN 3 - What-if Scenarios (Phân tích giả định)
Hãy phân tích rủi ro và cách ứng phó cho phương án đề xuất nếu xảy ra các tình huống sau:

1. **Nếu có 1 triệu notification được tạo trong vòng 2 phút** thì điều gì sẽ xảy ra với API, queue, worker và database?
2. **Nếu nhà cung cấp email hoặc SMS phản hồi chậm / timeout / rate limit** thì hệ thống nên xử lý ra sao?
3. **Nếu worker chết giữa chừng khi đang gửi dở** thì làm thế nào để không mất message?
4. **Nếu một user nhận bị gửi trùng thông báo do retry** thì cần cơ chế gì để đảm bảo idempotency?
5. **Nếu backlog trong queue tăng quá lớn** thì nên scale theo chiều ngang như thế nào?
6. **Nếu cần theo dõi trạng thái từng notification (PENDING, SENT, FAILED, RETRYING)** thì nên thiết kế bảng dữ liệu hoặc event model ra sao?

---

## PHẦN 4 - Kết luận và triển khai mẫu
Sau khi phân tích xong 3 phần trên:
1. Chọn **1 kiến trúc khuyến nghị** cho production.
2. Sinh ra **mã nguồn Java/Spring Boot mẫu** cho kiến trúc đó, ở mức skeleton nhưng có đủ các thành phần chính, ví dụ:
   - REST API tạo yêu cầu gửi notification
   - DTO / Entity NotificationJob
   - Publisher hoặc Producer đẩy job vào queue
   - Consumer / Worker xử lý gửi email
   - Retry cơ bản
   - Trạng thái xử lý notification
3. Sử dụng **Java 17 + Spring Boot**.
4. Ưu tiên code dễ đọc, dễ mở rộng, tách lớp rõ ràng theo trách nhiệm.

## Định dạng đầu ra mong muốn
### Phần A: Tóm tắt bài toán
### Phần B: Ít nhất 3 phương án kiến trúc
### Phần C: Bảng trade-off so sánh
### Phần D: Phân tích What-if Scenarios
### Phần E: Kiến trúc được chọn + lý do
### Phần F: Mã nguồn Java/Spring Boot mẫu trong code markdown
```

---

# 4. Vì sao prompt trên tốt hơn prompt thô?

## 4.1. Tích hợp kỹ thuật Multiple Options

Prompt mới **không cho AI chốt một cách làm ngay**, mà bắt buộc AI phải đưa ra **ít nhất 3 phương án**.
Điều này rất quan trọng vì với hệ thống lớn, “giải pháp đúng” không chỉ là “chạy được”, mà phải cân nhắc:

* quy mô;
* khả năng mở rộng;
* vận hành;
* chi phí;
* rủi ro production.

---

## 4.2. Tích hợp kỹ thuật Trade-offs

Prompt mới yêu cầu AI **lập bảng so sánh** theo nhiều tiêu chí kỹ thuật và vận hành.
Điều này giúp tránh kiểu trả lời hời hợt như:

> “Dùng `@Async` là được.”

Thay vào đó, AI phải trả lời câu hỏi quan trọng hơn:

* `@Async` **được đến mức nào**?
* khi nào nên dùng queue?
* khi nào nên tách worker?
* giải pháp nào phù hợp cho **1 triệu user**?

---

## 4.3. Tích hợp kỹ thuật What-if Scenario

Đây là phần giúp prompt mạnh nhất về mặt kiến trúc.
Nó buộc AI suy nghĩ như người thiết kế hệ thống thật sự:

* nếu queue backlog thì sao?
* nếu provider timeout thì sao?
* nếu worker chết thì sao?
* nếu retry gây duplicate thì sao?

Nhờ đó, đầu ra không còn là “một đoạn code async”, mà là **một phương án production-ready hơn**.

---

# 5. Minh họa đầu ra mà AI có thể sinh ra từ prompt tối ưu

Dưới đây là **một ví dụ bảng so sánh kiến trúc** mà AI có thể sinh ra sau khi nhận prompt tối ưu.

```markdown
| Phương án | Mô tả ngắn | Ưu điểm | Nhược điểm | Khả năng scale | Phù hợp 1 triệu user |
|----------|------------|---------|------------|----------------|----------------------|
| 1. Spring @Async + ThreadPool | Gửi bất đồng bộ trong cùng app bằng thread pool | Dễ làm, ít hạ tầng, nhanh để demo | Khó scale lớn, phụ thuộc tài nguyên app chính, khó retry bền vững | Thấp - Trung bình | Không phù hợp production tải rất lớn |
| 2. RabbitMQ/Kafka + Worker Service | API chỉ tạo job, đẩy vào queue; worker tiêu thụ và gửi | Chống timeout tốt, scale worker ngang, retry tốt, tách biệt producer/consumer | Hạ tầng phức tạp hơn, cần monitoring queue | Cao | Phù hợp |
| 3. Batch Job + Scheduler | Lưu danh sách người nhận, job nền xử lý theo lô | Kiểm soát tốc độ gửi tốt, phù hợp chiến dịch lớn | Độ trễ cao hơn, không realtime bằng queue | Trung bình - Cao | Phù hợp nếu chấp nhận delay |
| 4. Outbox Pattern + MQ | Ghi DB transactionally rồi publish ra queue | Tăng độ tin cậy, tránh mất message khi lỗi giữa chừng | Phức tạp hơn khi triển khai | Rất cao | Rất phù hợp cho production lớn |
```

---

# 6. Kiến trúc khuyến nghị cho bài toán này

## 6.1. Phương án đề xuất

Nếu mục tiêu là **production cho 1 triệu người dùng**, phương án phù hợp nhất là:

> **Spring Boot API + Message Broker (RabbitMQ hoặc Kafka) + Notification Worker riêng + bảng lưu trạng thái NotificationJob**

### Luồng tổng quát

1. API nhận yêu cầu tạo chiến dịch gửi thông báo.
2. API **không gửi email/SMS trực tiếp**.
3. API tạo các `NotificationJob` hoặc `NotificationBatch`.
4. API đẩy message vào **queue**.
5. Các **worker** đọc queue và gọi email/SMS provider.
6. Kết quả được cập nhật về trạng thái:

   * `PENDING`
   * `SENT`
   * `FAILED`
   * `RETRYING`

---

## 6.2. Vì sao phương án này hợp lý?

* API không bị block lâu → giảm timeout.
* Có thể scale nhiều worker song song.
* Dễ retry khi provider lỗi.
* Có thể giới hạn tốc độ gửi để tránh rate limit.
* Có nơi lưu trạng thái để theo dõi chiến dịch gửi.
* Tách biệt trách nhiệm giữa:

  * **Producer** (tạo job)
  * **Consumer/Worker** (thực hiện gửi)

---

# 7. Mã nguồn Java/Spring Boot mẫu cho kiến trúc khuyến nghị

> Lưu ý: Đây là **skeleton code minh họa kiến trúc**, không phải full production implementation.
> Mục tiêu của đoạn code là thể hiện đúng tinh thần prompt tối ưu: **API → Queue → Worker → Update trạng thái**.

```java
// NotificationStatus.java
public enum NotificationStatus {
    PENDING,
    SENT,
    FAILED,
    RETRYING
}
```

```java
// NotificationRequest.java
public class NotificationRequest {
    private String userId;
    private String email;
    private String phone;
    private String subject;
    private String content;

    public NotificationRequest() {
    }

    public NotificationRequest(String userId, String email, String phone, String subject, String content) {
        this.userId = userId;
        this.email = email;
        this.phone = phone;
        this.subject = subject;
        this.content = content;
    }

    public String getUserId() {
        return userId;
    }

    public String getEmail() {
        return email;
    }

    public String getPhone() {
        return phone;
    }

    public String getSubject() {
        return subject;
    }

    public String getContent() {
        return content;
    }
}
```

```java
// NotificationJob.java
import java.time.LocalDateTime;
import java.util.UUID;

public class NotificationJob {
    private String id;
    private String userId;
    private String email;
    private String phone;
    private String subject;
    private String content;
    private NotificationStatus status;
    private int retryCount;
    private LocalDateTime createdAt;

    public NotificationJob() {
    }

    public NotificationJob(String userId, String email, String phone, String subject, String content) {
        this.id = UUID.randomUUID().toString();
        this.userId = userId;
        this.email = email;
        this.phone = phone;
        this.subject = subject;
        this.content = content;
        this.status = NotificationStatus.PENDING;
        this.retryCount = 0;
        this.createdAt = LocalDateTime.now();
    }

    public String getId() {
        return id;
    }

    public String getUserId() {
        return userId;
    }

    public String getEmail() {
        return email;
    }

    public String getPhone() {
        return phone;
    }

    public String getSubject() {
        return subject;
    }

    public String getContent() {
        return content;
    }

    public NotificationStatus getStatus() {
        return status;
    }

    public int getRetryCount() {
        return retryCount;
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public void setStatus(NotificationStatus status) {
        this.status = status;
    }

    public void increaseRetryCount() {
        this.retryCount++;
    }
}
```

```java
// NotificationQueue.java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class NotificationQueue {
    private final BlockingQueue<NotificationJob> queue = new LinkedBlockingQueue<>();

    public void publish(NotificationJob job) throws InterruptedException {
        queue.put(job);
    }

    public NotificationJob consume() throws InterruptedException {
        return queue.take();
    }
}
```

```java
// EmailSender.java
public class EmailSender {

    public void sendEmail(String email, String subject, String content) {
        // Giả lập gửi email
        System.out.println("Sending email to " + email + " | subject=" + subject);
    }
}
```

```java
// NotificationService.java
public class NotificationService {

    private final NotificationQueue notificationQueue;

    public NotificationService(NotificationQueue notificationQueue) {
        this.notificationQueue = notificationQueue;
    }

    public NotificationJob createNotificationJob(NotificationRequest request) throws InterruptedException {
        NotificationJob job = new NotificationJob(
                request.getUserId(),
                request.getEmail(),
                request.getPhone(),
                request.getSubject(),
                request.getContent()
        );

        notificationQueue.publish(job);
        return job;
    }
}
```

```java
// NotificationWorker.java
public class NotificationWorker implements Runnable {

    private static final int MAX_RETRY = 3;

    private final NotificationQueue queue;
    private final EmailSender emailSender;

    public NotificationWorker(NotificationQueue queue, EmailSender emailSender) {
        this.queue = queue;
        this.emailSender = emailSender;
    }

    @Override
    public void run() {
        while (true) {
            try {
                NotificationJob job = queue.consume();
                process(job);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    private void process(NotificationJob job) {
        try {
            emailSender.sendEmail(job.getEmail(), job.getSubject(), job.getContent());
            job.setStatus(NotificationStatus.SENT);
            System.out.println("Job " + job.getId() + " sent successfully.");
        } catch (Exception ex) {
            handleFailure(job, ex);
        }
    }

    private void handleFailure(NotificationJob job, Exception ex) {
        job.increaseRetryCount();

        if (job.getRetryCount() <= MAX_RETRY) {
            job.setStatus(NotificationStatus.RETRYING);
            System.out.println("Retrying job " + job.getId() + ", retryCount=" + job.getRetryCount());
            try {
                queue.publish(job);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        } else {
            job.setStatus(NotificationStatus.FAILED);
            System.out.println("Job " + job.getId() + " failed permanently: " + ex.getMessage());
        }
    }
}
```

```java
// NotificationController.java
public class NotificationController {

    private final NotificationService notificationService;

    public NotificationController(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public NotificationJob send(NotificationRequest request) throws InterruptedException {
        return notificationService.createNotificationJob(request);
    }
}
```

```java
// DemoApplication.java
public class DemoApplication {

    public static void main(String[] args) throws Exception {
        NotificationQueue queue = new NotificationQueue();
        EmailSender emailSender = new EmailSender();

        NotificationService notificationService = new NotificationService(queue);
        NotificationController controller = new NotificationController(notificationService);

        Thread worker1 = new Thread(new NotificationWorker(queue, emailSender));
        Thread worker2 = new Thread(new NotificationWorker(queue, emailSender));
        worker1.start();
        worker2.start();

        NotificationRequest request = new NotificationRequest(
                "user-001",
                "user@example.com",
                "0123456789",
                "Khuyến mãi lớn",
                "Bạn có ưu đãi mới hôm nay!"
        );

        NotificationJob job = controller.send(request);
        System.out.println("Created notification job: " + job.getId());
    }
}
```

---

# 8. Kết luận

Prompt thô ban đầu chỉ yêu cầu “viết hàm gửi email bất đồng bộ”, nên rất dễ khiến AI:

* nhảy vào code ngay;
* chọn một kỹ thuật cục bộ như `@Async`;
* bỏ qua đánh giá kiến trúc;
* không phân tích rủi ro khi tải tăng cao.

Prompt tối ưu mới đã cải thiện rõ rệt bằng cách tích hợp **3 kỹ thuật nâng cao**:

## 8.1. Multiple Options

Buộc AI đề xuất **nhiều phương án kiến trúc**, tránh chốt giải pháp quá sớm.

## 8.2. Trade-offs

Buộc AI lập **bảng so sánh ưu/nhược điểm**, giúp lựa chọn giải pháp có cơ sở kỹ thuật.

## 8.3. What-if Scenario

Buộc AI suy nghĩ theo góc nhìn production:

* tải tăng đột biến,
* provider timeout,
* retry trùng lặp,
* worker chết,
* backlog queue,
* theo dõi trạng thái gửi.

Nhờ vậy, đầu ra không còn là “một đoạn code async đơn lẻ”, mà trở thành **một phân tích kiến trúc hoàn chỉnh và sát thực tế hơn cho hệ thống Notification Service quy mô lớn**.
