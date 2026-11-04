# Bài tập: Thêm Custom Fields vào Log JSON

**Ngày thực hiện:** 04/11/2026
**Dự án:** `order-app` (Spring Boot 3.3, Java 17, PostgreSQL)
**Môi trường chạy:** AWS EC2 (Ubuntu), PostgreSQL chạy bằng Docker

---

## 1. Cấu hình gốc và vấn đề

### Tình huống

Giám đốc yêu cầu tìm kiếm log giao dịch trên Kibana theo `service_name` (tên dịch vụ) và `environment` (môi trường), để biết lỗi phát sinh ở service nào và ở môi trường dev hay prod. Tuy nhiên log JSON hiện chỉ có các trường mặc định (`message`, `level`, `thread_name`...).

### Cấu hình gốc

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <!-- Đang thiếu custom fields -->
</encoder>
```

### Vấn đề

- Mỗi dòng log không cho biết nó đến từ service nào và môi trường nào.
- Khi nhiều service cùng đẩy log lên một Elasticsearch/Kibana, không thể lọc "chỉ log của `order-service`" hay "chỉ lỗi ở môi trường production".
- Chỉ có thể đoán dựa vào `logger_name` hoặc nội dung `message`, cách này không đáng tin cậy.

---

## 2. Yêu cầu và cách thực hiện

### Yêu cầu

1. Sửa `LogstashEncoder` trong `logback-spring.xml`, thêm block `<customFields>`.
2. Khai báo 2 field: `{"app_name":"order-service","env":"production"}`.
3. Chạy thử ứng dụng để kiểm tra log đầu ra.

### Bước 1. Thêm `<customFields>` vào encoder

Thêm vào **cả hai** encoder (console và file) để log ở đâu cũng có hai field:

```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app_name":"order-service","env":"production"}</customFields>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>10MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app_name":"order-service","env":"production"}</customFields>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

Nội dung của `<customFields>` phải là một chuỗi JSON hợp lệ. Nếu sai cú pháp, Logback chỉ cảnh báo lúc khởi động và field sẽ không xuất hiện.

### Bước 2. Build và chạy

```bash
rm -rf logs/*
mvn -DskipTests package
DB_PORT=5433 java -jar target/order-app-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod
```

### Bước 3. Sinh các loại log khác nhau

```bash
curl -s localhost:8080/api/products > /dev/null
curl -s -X POST localhost:8080/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"customerName":"Nguyen Van A","customerPhone":"0901234567","items":[{"productId":1,"quantity":1}]}' > /dev/null
curl -s localhost:8080/api/debug/error > /dev/null
```

### Bước 4. Kiểm chứng mọi dòng đều có hai field

```bash
python3 - <<'EOF'
import json
total = missing = 0
for line in open("logs/app.log"):
    total += 1
    d = json.loads(line)
    if d.get("app_name") != "order-service" or d.get("env") != "production":
        missing += 1
        print("Thiếu field:", line[:120])
print(f"Tổng số dòng: {total}, số dòng thiếu field: {missing}")
EOF
```

---

## 3. Kết quả

### Kết quả kiểm tra tự động

```text
Tổng số dòng: 30, số dòng thiếu field: 0
```

Cả 30 dòng log trong `logs/app.log` đều có `"app_name":"order-service"` và `"env":"production"`.

### Dòng log INFO (tạo đơn hàng)

```json
{
    "@timestamp": "2026-11-04T04:07:26.690553895Z",
    "@version": "1",
    "message": "Order created id=33 customer=Nguyen Van A total=1200000.00",
    "logger_name": "com.example.orderapp.service.OrderService",
    "thread_name": "http-nio-8080-exec-3",
    "level": "INFO",
    "level_value": 20000,
    "app_name": "order-service",
    "env": "production"
}
```

### Dòng log ERROR (có stack trace)

Dưới đây là bản đã rút gọn các frame của Spring và Tomcat:

```json
{
    "@timestamp": "2026-11-04T04:07:32.92035628Z",
    "@version": "1",
    "message": "Payment processing failed for orderId=123",
    "logger_name": "com.example.orderapp.controller.DebugController",
    "thread_name": "http-nio-8080-exec-7",
    "level": "ERROR",
    "level_value": 40000,
    "stack_trace": "java.lang.IllegalStateException: Invalid payment amount\n\tat com.example.orderapp.controller.DebugController.processPayment(DebugController.java:33)\n\tat com.example.orderapp.controller.DebugController.triggerError(DebugController.java:21)\n\t... (các frame của Spring MVC, Tomcat) ...\nCaused by: java.lang.NumberFormatException: For input string: \"abc\"\n\t... 51 common frames omitted\n",
    "app_name": "order-service",
    "env": "production"
}
```

### Nhận xét

- Đạt yêu cầu của đề: mọi dòng log JSON đều có thêm hai thuộc tính `"app_name":"order-service"` và `"env":"production"`.
- Hai field xuất hiện ở mọi mức log (INFO, ERROR...) và cả ở dòng có `stack_trace`, vì `customFields` được gắn ở tầng encoder.
- Trên Kibana có thể lọc trực tiếp, ví dụ `app_name: "order-service" and env: "production" and level: "ERROR"`, mà không cần grok hay đoán từ nội dung message.
- Khi hệ thống có nhiều service, mỗi service khai báo `app_name` riêng thì log của chúng vẫn phân biệt được trong cùng một index.

### Lưu ý và hướng mở rộng

- Giá trị `"env":"production"` đang được viết cứng trong file XML, đúng theo đề bài. Nếu chạy với profile `dev` thì log vẫn ghi `production`, gây hiểu nhầm khi lọc trên Kibana.
- Cách làm đúng cho thực tế là lấy giá trị từ profile đang chạy, ví dụ dùng `<springProperty>` để đọc `spring.profiles.active` hoặc `spring.application.name` rồi dùng `${...}` trong `<customFields>`, hoặc tách `<springProfile name="dev">` / `<springProfile name="prod">` với giá trị riêng.
- Với dữ liệu thay đổi theo từng request (ví dụ `orderId`, `traceId`), dùng MDC hoặc `StructuredArguments` thay cho `customFields`, vì `customFields` chỉ dành cho giá trị cố định của toàn ứng dụng.
