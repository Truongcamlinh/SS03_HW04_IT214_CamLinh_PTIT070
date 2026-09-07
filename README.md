# Bài 4: Phân tích sự cố Service Discovery khi scale hệ thống

## 1. Mô tả sự cố

Hệ thống FoodX tăng `restaurant-service` từ 1 instance lên 4 instance để chịu tải giờ cao điểm. Tuy nhiên `order-service` vẫn chỉ gọi được một instance, nên việc scale gần như không có tác dụng.

Khi kiểm tra cấu hình, `restaurant-service` có dùng Eureka Client nhưng cấu hình chưa chuẩn. Vì vậy một số instance không đăng ký được hoặc đăng ký bằng tên không thống nhất.

## 2. Nguyên nhân

### 2.1 Tên service viết không thống nhất

Cấu hình cũ:

```yaml
spring:
  application:
    name: RestaurantService
```

Tên này khác kiểu với `order-service`, `payment-service`. Dù Eureka vẫn có thể nhận service name này, nhưng khi service khác gọi bằng tên `restaurant-service` thì sẽ không khớp với tên đã đăng ký.

Nên sửa thành:

```yaml
spring:
  application:
    name: restaurant-service
```

Việc thống nhất tên giúp khi gọi qua LoadBalancer, ví dụ `http://restaurant-service/...`, Eureka tìm đúng danh sách instance.

### 2.2 defaultZone thiếu dấu /

Cấu hình cũ:

```yaml
defaultZone: http://eureka-server:8761/eureka
```

Cấu hình đúng:

```yaml
defaultZone: http://eureka-server:8761/eureka/
```

Dấu `/` cuối URL là phần nên có trong endpoint Eureka. Nếu viết sai URL, client có thể không gửi request đăng ký đúng tới Eureka Server.

### 2.3 Chưa khai báo rõ register và fetch registry

Nên viết rõ:

```yaml
register-with-eureka: true
fetch-registry: true
```

Vì `restaurant-service` là service nghiệp vụ nên phải tự đăng ký lên Eureka và có thể lấy registry nếu cần gọi service khác.

## 3. Cấu hình sửa cho restaurant-service

File:

```text
config/restaurant-service.yml
```

```yaml
spring:
  application:
    name: restaurant-service

server:
  port: ${PORT:0}

eureka:
  instance:
    instance-id: ${spring.application.name}:${random.uuid}
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
```

Khi chạy 4 instance local, `server.port: 0` giúp mỗi instance dùng port khác nhau. `instance-id` có `random.uuid` để tránh các instance bị trùng tên.

## 4. Heartbeat của Eureka

Sau khi đăng ký, Eureka Client gửi heartbeat định kỳ để báo nó còn sống. Thông thường client gửi khoảng mỗi 30 giây. Eureka Server sẽ chờ một khoảng lease, thường khoảng 90 giây. Nếu quá thời gian này mà không nhận heartbeat, instance bị coi là chết và bị loại khỏi registry.

Nếu `restaurant-service-2` crash đột ngột, nó không kịp gửi tín hiệu shutdown. Eureka Server chưa xóa ngay, mà đợi hết thời gian lease rồi mới loại bỏ. Trong thời gian đó, `order-service` có thể vẫn nhận được địa chỉ instance đã chết, nên cần timeout/retry để chuyển sang instance khác.

## 5. Đề xuất cho order-service

`order-service` không nên gọi địa chỉ cứng như:

```text
http://localhost:8082
```

Nên dùng Eureka và gọi theo tên service:

```text
http://restaurant-service/api/restaurants
```

Nếu dùng Spring Boot, có thể cấu hình `RestTemplate`:

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

Khi đó Spring Cloud LoadBalancer sẽ lấy danh sách instance mới nhất từ Eureka và chọn một instance để gọi.

## 6. Kết luận

Sự cố không nằm ở việc scale 4 instance, mà nằm ở Service Discovery cấu hình sai. Cần sửa tên service, sửa `defaultZone`, khai báo rõ Eureka Client và để `order-service` gọi qua service name thay vì địa chỉ cứng.
