---
name: error-handling
description: "에러 처리 패턴 가이드. 예외 계층 설계, 글로벌 예외 처리, 로깅 전략, 에러 응답 표준화를 제공합니다. 애플리케이션의 에러 처리 구조를 설계할 때 사용하세요."
---

# Error Handling Guide

## Overview

일관되고 유지보수가 쉬운 에러 처리 패턴을 제공합니다.
예외 계층 설계부터 로깅, 모니터링까지 종합적인 에러 처리 전략을 다룹니다.

## 예외 계층 설계

```
┌─────────────────────────────────────────────────────────────────┐
│                    Exception Hierarchy                          │
│                                                                 │
│   RuntimeException                                              │
│        │                                                        │
│        └── BusinessException (비즈니스 예외 기본 클래스)        │
│              │                                                  │
│              ├── EntityNotFoundException                        │
│              │     ├── UserNotFoundException                    │
│              │     └── OrderNotFoundException                   │
│              │                                                  │
│              ├── DuplicateException                             │
│              │     └── DuplicateEmailException                  │
│              │                                                  │
│              ├── InvalidStateException                          │
│              │     ├── OrderAlreadyCancelledException           │
│              │     └── UserAlreadyActiveException               │
│              │                                                  │
│              └── ValidationException                            │
│                    └── InvalidEmailFormatException              │
└─────────────────────────────────────────────────────────────────┘
```

## 예외 클래스 구현

### 기본 비즈니스 예외

```java
@Getter
public abstract class BusinessException extends RuntimeException {

    private final ErrorCode errorCode;
    private final Map<String, Object> details;

    protected BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
        this.details = new HashMap<>();
    }

    protected BusinessException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.details = new HashMap<>();
    }

    public BusinessException addDetail(String key, Object value) {
        this.details.put(key, value);
        return this;
    }
}
```

### 에러 코드 정의

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {

    // Common
    INVALID_INPUT(400, "C001", "입력값이 올바르지 않습니다"),
    INTERNAL_ERROR(500, "C002", "서버 오류가 발생했습니다"),

    // User
    USER_NOT_FOUND(404, "U001", "사용자를 찾을 수 없습니다"),
    DUPLICATE_EMAIL(409, "U002", "이미 사용 중인 이메일입니다"),
    USER_ALREADY_ACTIVE(409, "U003", "이미 활성화된 사용자입니다"),
    INVALID_PASSWORD(400, "U004", "비밀번호가 올바르지 않습니다"),

    // Order
    ORDER_NOT_FOUND(404, "O001", "주문을 찾을 수 없습니다"),
    ORDER_ALREADY_CANCELLED(409, "O002", "이미 취소된 주문입니다"),
    INSUFFICIENT_STOCK(400, "O003", "재고가 부족합니다"),

    // Auth
    UNAUTHORIZED(401, "A001", "인증이 필요합니다"),
    FORBIDDEN(403, "A002", "권한이 없습니다"),
    TOKEN_EXPIRED(401, "A003", "토큰이 만료되었습니다");

    private final int status;
    private final String code;
    private final String message;
}
```

### 구체적인 예외 클래스

```java
// 엔티티 조회 실패
public class EntityNotFoundException extends BusinessException {

    public EntityNotFoundException(ErrorCode errorCode, String entityName, Object id) {
        super(errorCode, String.format("%s를 찾을 수 없습니다. (ID: %s)", entityName, id));
        addDetail("entityName", entityName);
        addDetail("id", id);
    }
}

public class UserNotFoundException extends EntityNotFoundException {
    public UserNotFoundException(Long id) {
        super(ErrorCode.USER_NOT_FOUND, "사용자", id);
    }

    public UserNotFoundException(String email) {
        super(ErrorCode.USER_NOT_FOUND, "사용자", email);
    }
}

// 중복 예외
public class DuplicateException extends BusinessException {

    public DuplicateException(ErrorCode errorCode, String field, Object value) {
        super(errorCode, String.format("이미 존재하는 %s입니다: %s", field, value));
        addDetail("field", field);
        addDetail("value", value);
    }
}

public class DuplicateEmailException extends DuplicateException {
    public DuplicateEmailException(String email) {
        super(ErrorCode.DUPLICATE_EMAIL, "이메일", email);
    }
}

// 상태 예외
public class InvalidStateException extends BusinessException {

    public InvalidStateException(ErrorCode errorCode, String currentState, String expectedState) {
        super(errorCode);
        addDetail("currentState", currentState);
        addDetail("expectedState", expectedState);
    }
}

public class OrderAlreadyCancelledException extends InvalidStateException {
    public OrderAlreadyCancelledException(Long orderId) {
        super(ErrorCode.ORDER_ALREADY_CANCELLED, "CANCELLED", "PENDING or CONFIRMED");
        addDetail("orderId", orderId);
    }
}
```

## 글로벌 예외 처리

### GlobalExceptionHandler

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 비즈니스 예외 처리
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(
            BusinessException e, HttpServletRequest request) {

        ErrorCode errorCode = e.getErrorCode();

        log.warn("Business exception: {} - {} [{}]",
            errorCode.getCode(), e.getMessage(), request.getRequestURI());

        return ResponseEntity
            .status(errorCode.getStatus())
            .body(ErrorResponse.of(errorCode, e.getMessage(), e.getDetails()));
    }

    // 유효성 검사 예외 (Bean Validation)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException e, HttpServletRequest request) {

        List<ErrorResponse.FieldError> fieldErrors = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> ErrorResponse.FieldError.of(
                error.getField(),
                error.getDefaultMessage(),
                error.getRejectedValue()
            ))
            .toList();

        log.warn("Validation failed: {} fields [{}]",
            fieldErrors.size(), request.getRequestURI());

        return ResponseEntity
            .badRequest()
            .body(ErrorResponse.ofValidation(fieldErrors));
    }

    // 잘못된 요청 형식
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ErrorResponse> handleMessageNotReadable(
            HttpMessageNotReadableException e, HttpServletRequest request) {

        log.warn("Message not readable: {} [{}]",
            e.getMessage(), request.getRequestURI());

        return ResponseEntity
            .badRequest()
            .body(ErrorResponse.of(ErrorCode.INVALID_INPUT, "요청 형식이 올바르지 않습니다"));
    }

    // 지원하지 않는 HTTP 메서드
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseEntity<ErrorResponse> handleMethodNotSupported(
            HttpRequestMethodNotSupportedException e) {

        return ResponseEntity
            .status(HttpStatus.METHOD_NOT_ALLOWED)
            .body(ErrorResponse.of(
                405, "METHOD_NOT_ALLOWED",
                String.format("지원하지 않는 메서드입니다: %s", e.getMethod())
            ));
    }

    // 예상치 못한 예외 (최후 방어)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(
            Exception e, HttpServletRequest request) {

        // 예상치 못한 예외는 ERROR 레벨로 로깅 (스택 트레이스 포함)
        log.error("Unexpected error [{}]: {}", request.getRequestURI(), e.getMessage(), e);

        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.of(ErrorCode.INTERNAL_ERROR));
    }
}
```

### ErrorResponse DTO

```java
@Getter
@Builder
public class ErrorResponse {

    private final int status;
    private final String code;
    private final String message;
    private final Map<String, Object> details;
    private final List<FieldError> fieldErrors;
    private final LocalDateTime timestamp;

    @Getter
    @Builder
    public static class FieldError {
        private final String field;
        private final String message;
        private final Object rejectedValue;

        public static FieldError of(String field, String message, Object rejectedValue) {
            return FieldError.builder()
                .field(field)
                .message(message)
                .rejectedValue(rejectedValue)
                .build();
        }
    }

    public static ErrorResponse of(ErrorCode errorCode) {
        return ErrorResponse.builder()
            .status(errorCode.getStatus())
            .code(errorCode.getCode())
            .message(errorCode.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
    }

    public static ErrorResponse of(ErrorCode errorCode, String message) {
        return ErrorResponse.builder()
            .status(errorCode.getStatus())
            .code(errorCode.getCode())
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }

    public static ErrorResponse of(ErrorCode errorCode, String message,
                                   Map<String, Object> details) {
        return ErrorResponse.builder()
            .status(errorCode.getStatus())
            .code(errorCode.getCode())
            .message(message)
            .details(details)
            .timestamp(LocalDateTime.now())
            .build();
    }

    public static ErrorResponse of(int status, String code, String message) {
        return ErrorResponse.builder()
            .status(status)
            .code(code)
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }

    public static ErrorResponse ofValidation(List<FieldError> fieldErrors) {
        return ErrorResponse.builder()
            .status(400)
            .code("VALIDATION_ERROR")
            .message("입력값이 올바르지 않습니다")
            .fieldErrors(fieldErrors)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

## 로깅 전략

### 로그 레벨 가이드

| 레벨 | 용도 | 예시 |
|------|------|------|
| ERROR | 즉시 대응 필요한 오류 | 예상치 못한 예외, 외부 시스템 장애 |
| WARN | 주의 필요한 상황 | 비즈니스 예외, 재시도 실패 |
| INFO | 중요 비즈니스 이벤트 | 사용자 가입, 결제 완료 |
| DEBUG | 개발/디버깅 정보 | 메서드 진입/종료, 변수 값 |
| TRACE | 상세 추적 정보 | 루프 내부, 상세 흐름 |

### 로깅 베스트 프랙티스

```java
@Slf4j
@Service
public class OrderService {

    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request, Long userId) {
        // INFO: 중요한 비즈니스 이벤트 시작
        log.info("Creating order for user: {}", userId);

        try {
            User user = userRepository.findById(userId)
                .orElseThrow(() -> {
                    // WARN: 비즈니스 예외 (스택 트레이스 불필요)
                    log.warn("User not found: {}", userId);
                    return new UserNotFoundException(userId);
                });

            // DEBUG: 디버깅용 상세 정보
            log.debug("Found user: {}, processing {} items",
                user.getEmail(), request.items().size());

            Order order = Order.create(user, request.items());
            Order savedOrder = orderRepository.save(order);

            // INFO: 중요한 비즈니스 이벤트 완료
            log.info("Order created successfully: orderId={}, userId={}, total={}",
                savedOrder.getId(), userId, savedOrder.getTotal());

            return OrderResponse.from(savedOrder);

        } catch (BusinessException e) {
            // 비즈니스 예외는 GlobalExceptionHandler에서 로깅
            throw e;
        } catch (Exception e) {
            // ERROR: 예상치 못한 예외 (스택 트레이스 포함)
            log.error("Failed to create order for user: {}", userId, e);
            throw new OrderCreationFailedException(userId, e);
        }
    }
}
```

### 구조화된 로깅 (JSON)

```yaml
# application.yml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  level:
    root: INFO
    com.example: DEBUG
    org.springframework.web: WARN

# logback-spring.xml (JSON 포맷)
```

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="prod">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>userId</includeMdcKeyName>
                <includeMdcKeyName>requestId</includeMdcKeyName>
            </encoder>
        </appender>

        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>
</configuration>
```

### MDC를 활용한 요청 추적

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String requestId = UUID.randomUUID().toString().substring(0, 8);

        MDC.put("requestId", requestId);
        MDC.put("method", request.getMethod());
        MDC.put("uri", request.getRequestURI());

        try {
            response.setHeader("X-Request-Id", requestId);
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

## 외부 서비스 에러 처리

### 재시도 패턴

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentClient paymentClient;

    @Retryable(
        value = {PaymentTemporaryException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Processing payment: orderId={}", request.orderId());

        try {
            return paymentClient.pay(request);
        } catch (PaymentClientException e) {
            if (e.isRetryable()) {
                log.warn("Payment failed (retryable): orderId={}, error={}",
                    request.orderId(), e.getMessage());
                throw new PaymentTemporaryException(e);
            }
            log.error("Payment failed (permanent): orderId={}", request.orderId(), e);
            throw new PaymentFailedException(request.orderId(), e);
        }
    }

    @Recover
    public PaymentResult recoverPayment(PaymentTemporaryException e,
                                        PaymentRequest request) {
        log.error("Payment failed after retries: orderId={}", request.orderId());
        throw new PaymentFailedException(request.orderId(), e);
    }
}
```

### Circuit Breaker 패턴

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ExternalApiService {

    private final ExternalApiClient apiClient;

    @CircuitBreaker(name = "externalApi", fallbackMethod = "fallbackCall")
    public ApiResponse callExternalApi(ApiRequest request) {
        return apiClient.call(request);
    }

    public ApiResponse fallbackCall(ApiRequest request, Exception e) {
        log.warn("Circuit breaker fallback: {}", e.getMessage());

        // 캐시된 데이터 반환 또는 기본값 반환
        return ApiResponse.defaultResponse();
    }
}
```

## 에러 모니터링

### 커스텀 메트릭

```java
@Aspect
@Component
@RequiredArgsConstructor
public class ExceptionMetricsAspect {

    private final MeterRegistry meterRegistry;

    @AfterThrowing(pointcut = "execution(* com.example..service.*.*(..))",
                   throwing = "ex")
    public void countException(JoinPoint joinPoint, Exception ex) {
        String exceptionClass = ex.getClass().getSimpleName();
        String method = joinPoint.getSignature().toShortString();

        Counter.builder("app.exceptions")
            .tag("exception", exceptionClass)
            .tag("method", method)
            .register(meterRegistry)
            .increment();

        if (ex instanceof BusinessException be) {
            Counter.builder("app.business_exceptions")
                .tag("code", be.getErrorCode().getCode())
                .register(meterRegistry)
                .increment();
        }
    }
}
```

## 체크리스트

### 예외 설계
- [ ] 비즈니스 예외와 시스템 예외가 분리되어 있는가?
- [ ] 예외에 충분한 컨텍스트 정보가 포함되어 있는가?
- [ ] 에러 코드가 체계적으로 정의되어 있는가?

### 예외 처리
- [ ] GlobalExceptionHandler가 모든 예외를 처리하는가?
- [ ] 클라이언트에 적절한 상태 코드를 반환하는가?
- [ ] 민감한 정보가 에러 응답에 노출되지 않는가?

### 로깅
- [ ] 로그 레벨이 적절하게 사용되고 있는가?
- [ ] 예상치 못한 예외에 스택 트레이스가 포함되는가?
- [ ] 요청 추적이 가능한가 (requestId)?

### 모니터링
- [ ] 에러 메트릭이 수집되고 있는가?
- [ ] 알림이 설정되어 있는가?
