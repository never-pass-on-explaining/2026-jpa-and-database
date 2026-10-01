# gRPC 정리

> 『데이터 중심 애플리케이션 설계』 4장(부호화와 발전)을 공부하면서 이어서 깊게 파본 gRPC 정리
> 책에서 다룬 RPC(Remote Procedure Call, 원격 프로시저 호출)와 스키마 발전 개념이 실제 기술에서 어떻게 구현되고 운영되는지에 초점을 둔다.

---

## 1. gRPC란?

**원격 서버의 메서드를 로컬 메서드처럼 호출하게 해주는 RPC 프레임워크.** Google 내부 RPC 시스템(Stubby)을 바탕으로 2015년에 오픈소스로 공개되었고, 현재 CNCF(Cloud Native Computing Foundation, 클라우드 네이티브 컴퓨팅 재단) 프로젝트다.

| 구성 요소 | 역할 | 기본 선택 |
|---|---|---|
| IDL (Interface Definition Language, 인터페이스 정의 언어) | 서비스와 메시지 구조 정의 | Protocol Buffers (줄여서 Protobuf) |
| 부호화 | 메시지를 바이트로 변환 | Protobuf 바이너리 |
| 전송 | 바이트를 주고받는 통로 | HTTP/2 (HyperText Transfer Protocol 버전 2) |

> 부호화 방식은 교체할 수 있지만 현실에서는 거의 항상 Protobuf를 쓴다. "gRPC = Protobuf + HTTP/2"로 기억해도 무방하다.

### 1.1 네 개의 층으로 보기

```
① 계약     .proto 파일            무엇을 주고받는가
② 코드     생성된 Stub / ImplBase  개발자가 호출하는 코드
③ 런타임   Channel, 인터셉터       연결, 로드 밸런싱, deadline
④ 전송     HTTP/2 스트림           실제 바이트가 오가는 방식
```

---

## 2. ① 계약: `.proto` 파일

서버와 클라이언트가 공유하는 **단 하나의 명세**.

```protobuf
syntax = "proto3";
package order.v1;

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (GetOrderResponse);                 // Unary
  rpc WatchOrderStatus (WatchOrderStatusRequest) returns (stream OrderStatus); // Server streaming
}

message GetOrderRequest {
  string order_id = 1;
}
```

> **용어: Unary (단항 호출)**
> 요청 메시지 1개를 보내고 응답 메시지 1개를 받는, 가장 기본적인 호출 방식이다. 일반적인 함수 호출이나 REST의 요청/응답과 같은 형태다.
> 이와 달리 한 번의 호출 안에서 메시지를 여러 개 주고받는 방식을 **스트리밍**이라고 하며, 위 예시의 `stream` 키워드가 이를 나타낸다.

- `service`: 호출할 수 있는 메서드 목록 (REST(Representational State Transfer) 방식의 엔드포인트 목록)
- `message`: 요청과 응답의 구조 (REST의 DTO(Data Transfer Object, 계층 간 데이터 전달 객체))
- `stream`: 메시지를 여러 개 주고받는다는 표시

| | REST | gRPC |
|---|---|---|
| 계약 | OpenAPI 문서 (선택 사항, 코드와 어긋날 수 있음) | `.proto` 파일 (필수, 코드가 여기서 생성됨) |
| 식별 | URL + HTTP 메서드 (`GET /orders/1`) | 서비스 + 메서드 (`OrderService.GetOrder`) |
| 설계 관점 | 자원(명사) 중심 | 동작(동사) 중심 |

---

## 3. ② 코드: 생성된 Stub과 서버 베이스 클래스

`protoc` 컴파일러가 `.proto` 파일에서 언어별 코드를 생성한다.

```
                    order.proto
                         │ protoc
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   Java 서버 코드    Go 클라이언트    Python 클라이언트
```

**서버: 생성된 베이스 클래스를 상속해서 구현**

```java
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    @Override
    public void getOrder(GetOrderRequest request, StreamObserver<GetOrderResponse> responseObserver) {
        responseObserver.onNext(buildResponse(request));  // 응답 메시지 전송
        responseObserver.onCompleted();                    // 호출 종료
    }
}
```

**클라이언트: 생성된 Stub으로 호출**

```java
OrderServiceGrpc.OrderServiceBlockingStub stub = OrderServiceGrpc.newBlockingStub(channel);

GetOrderResponse response = stub
        .withDeadlineAfter(500, TimeUnit.MILLISECONDS)
        .getOrder(GetOrderRequest.newBuilder().setOrderId("1").build());
```

- **Stub**: 원격 서버를 대신하는 클라이언트 쪽 대리 객체. 로컬 메서드처럼 보이지만 호출하면 네트워크 요청이 나간다.
- Java의 Stub 세 종류: Blocking(동기), Async(콜백), Future(`ListenableFuture`)
- **`StreamObserver`**: `onNext`, `onError`, `onCompleted` 세 메서드로 모든 통신 방식을 표현한다. Unary도 "메시지 1개짜리 스트림"으로 다룬다.

> 책은 "원격 호출을 로컬처럼 숨기는 것이 문제"라고 지적한다. gRPC는 Stub이라는 이름과 deadline 설정으로 원격 호출이라는 점을 드러낸다.

---

## 4. ③ 런타임: Channel

**Channel은 특정 서버(또는 서버 그룹)로 가는 논리적인 연결**이다.

| 역할 | 설명 |
|---|---|
| 이름 해석 | `order-service`를 실제 IP 목록으로 변환 (DNS(Domain Name System) 등) |
| 연결 관리 | 서버별 HTTP/2 연결을 맺고 유지, 끊기면 재연결 |
| 로드 밸런싱 | 호출마다 어느 연결로 보낼지 선택 |
| 호출 설정 | 인터셉터, 압축, keepalive 적용 |

- Channel은 생성 비용이 크므로 **애플리케이션 전체에서 한 번 만들어 재사용**한다. Spring에서는 빈(Bean)으로 등록해 컨테이너가 수명을 관리하게 한다.
- Stub은 가볍고 스레드 안전해서 빈으로 공유해도 된다.
- **Metadata**: HTTP 헤더에 해당. 인증 토큰, 트레이싱 ID를 담는다.
- **Deadline**: 호출이 끝나야 하는 절대 시각. 호출 체인을 따라 전파된다.
- **Interceptor**: 모든 호출에 공통 로직을 끼워 넣는다. Spring의 필터와 같은 역할.

### 4.1 Spring에서 Channel을 빈으로 관리하기

아래는 grpc-java API를 직접 써서 구성한 예시다. 클래스 세 개로 나눈다.

| 클래스 | 역할 |
|---|---|
| `OrderGrpcClientProperties` | 설정값 묶음. `application.yml`에서 주입 |
| `OrderGrpcChannelConfig` | Channel과 Stub을 빈으로 등록, 종료 시 정리 |
| `OrderServiceClient` | 실제 호출 담당. 호출마다 deadline 적용, 에러 변환 |

**① 설정값: `application.yml` + `OrderGrpcClientProperties`**

```yaml
app:
  grpc:
    order-service:
      target: dns:///order-service:9090
      load-balancing-policy: round_robin
      deadline: 500ms
      keep-alive-time: 5m
      keep-alive-timeout: 20s
      keep-alive-without-calls: false
      idle-timeout: 30m
      max-inbound-message-size: 4194304   # 4MB
      plaintext: true                     # 로컬 개발에서만 true
```

```java
@ConfigurationProperties(prefix = "app.grpc.order-service")
public record OrderGrpcClientProperties(
        String target,
        String loadBalancingPolicy,
        Duration deadline,
        Duration keepAliveTime,
        Duration keepAliveTimeout,
        boolean keepAliveWithoutCalls,
        Duration idleTimeout,
        int maxInboundMessageSize,
        boolean plaintext
) {
}
```

**② Channel과 Stub 빈 등록: `OrderGrpcChannelConfig`**

```java
@Configuration
@EnableConfigurationProperties(OrderGrpcClientProperties.class)
public class OrderGrpcChannelConfig {

    /**
     * 주문 서비스로 가는 Channel. 애플리케이션 전체에서 하나만 만들어 재사용한다.
     * destroyMethod = "" : Spring의 종료 메서드 자동 추론(shutdown)을 끄고,
     *                      아래 orderServiceChannelCloser 빈에서 정상 종료 절차를 직접 처리한다.
     */
    @Bean(destroyMethod = "")
    public ManagedChannel orderServiceChannel(OrderGrpcClientProperties props) {
        validateLoadBalancingPolicy(props.loadBalancingPolicy());

        ManagedChannelBuilder<?> builder = ManagedChannelBuilder
                // ── 대상 주소: "스킴:///주소" 형식 ────────────────────────────────
                //   "dns:///host:port"  : DNS로 IP 목록을 조회한다. 기본 스킴(스킴 생략 시 dns로 처리)
                //   "unix:///path"      : 유닉스 도메인 소켓 (같은 머신 안의 프로세스 간 통신)
                //   "xds:///service"    : xDS(x Discovery Service, 서비스 메시의 컨트롤 플레인이
                //                         주소·라우팅·정책을 내려주는 API 묶음)에서 정보를 받는다.
                //                         grpc-xds 의존성 필요
                //   Kubernetes에서 round_robin을 쓰려면 headless Service 주소를 지정해야
                //   Service IP 하나가 아니라 Pod IP 목록을 받는다.
                .forTarget(props.target())

                // ── 로드 밸런싱 정책 ──────────────────────────────────────────────
                //   enum이 아니라 "문자열 이름"으로 지정한다. 정책은 LoadBalancerProvider 구현체가
                //   LoadBalancerRegistry에 등록되는 플러그인 구조라서, 클래스패스에 어떤 모듈이
                //   있느냐에 따라 쓸 수 있는 값이 달라지기 때문이다.
                //   기본 제공:
                //     "pick_first"  : 기본값. 처음 연결되는 서버 하나만 사용 (사실상 분산 없음)
                //     "round_robin" : 모든 서버에 연결하고 호출마다 순서대로 분산
                //   grpc-xds 의존성을 추가하면 등록되는 정책 (이름은 버전에 따라 바뀔 수 있음):
                //     "weighted_round_robin"       : 서버가 보고하는 부하 지표로 가중치를 두어 분산
                //     "least_request_experimental" : 처리 중인 요청이 적은 서버를 우선 선택
                //     "ring_hash_experimental"     : 일관된 해싱. 같은 키의 요청을 같은 서버로
                .defaultLoadBalancingPolicy(props.loadBalancingPolicy())

                // ── keepalive: 유휴 연결이 중간 장비(LB, NAT)에서 조용히 끊기는 것을 방지 ──
                //   keepAliveTime         : 이 시간 동안 아무 통신이 없으면 HTTP/2 PING을 보낸다.
                //                           기본값은 사용 안 함. 서버가 허용하는 최소 간격
                //                           (grpc-java 서버 기본 5분)보다 짧으면 서버가
                //                           too_many_pings로 연결을 끊는다.
                //   keepAliveTimeout      : PING 응답을 기다리는 시간. 넘으면 연결이 끊긴 것으로 판단
                //   keepAliveWithoutCalls : 진행 중인 호출이 없어도 PING을 보낼지 (기본 false)
                .keepAliveTime(props.keepAliveTime().toMillis(), TimeUnit.MILLISECONDS)
                .keepAliveTimeout(props.keepAliveTimeout().toMillis(), TimeUnit.MILLISECONDS)
                .keepAliveWithoutCalls(props.keepAliveWithoutCalls())

                // ── 유휴 모드: 이 시간 동안 호출이 없으면 연결을 닫고, 다음 호출 때 다시 연결 (기본 30분)
                .idleTimeout(props.idleTimeout().toMillis(), TimeUnit.MILLISECONDS)

                // ── 받을 수 있는 최대 메시지 크기 (기본 4MB). 넘으면 RESOURCE_EXHAUSTED
                .maxInboundMessageSize(props.maxInboundMessageSize())

                // ── 서비스 설정: 메서드별 재시도 정책 (아래 retryServiceConfig 참고)
                .defaultServiceConfig(retryServiceConfig())
                .enableRetry();

        // ── 전송 보안 ─────────────────────────────────────────────────────────
        //   usePlaintext()         : 암호화 없음. 로컬 개발 전용
        //   useTransportSecurity() : TLS(Transport Layer Security) 사용. 기본값
        if (props.plaintext()) {
            builder.usePlaintext();
        } else {
            builder.useTransportSecurity();
        }

        return builder.build();
    }

    /**
     * 애플리케이션 종료 시 Channel을 정상 종료한다.
     * 이 빈은 Channel에 의존하므로 Channel보다 먼저 정리된다.
     */
    @Bean
    public DisposableBean orderServiceChannelCloser(ManagedChannel orderServiceChannel) {
        return () -> {
            orderServiceChannel.shutdown();                  // 새 호출은 거절, 진행 중인 호출은 마저 처리
            if (!orderServiceChannel.awaitTermination(5, TimeUnit.SECONDS)) {
                orderServiceChannel.shutdownNow();           // 시간 안에 끝나지 않으면 강제 취소
            }
        };
    }

    /**
     * Stub은 스레드 안전하고 가벼워서 빈으로 공유해도 된다.
     * 단, 여기서 withDeadlineAfter를 걸면 안 된다 (4.2 참고).
     */
    @Bean
    public OrderServiceGrpc.OrderServiceBlockingStub orderServiceBlockingStub(ManagedChannel orderServiceChannel) {
        return OrderServiceGrpc.newBlockingStub(orderServiceChannel);
    }

    /**
     * 서비스 설정(service config): 메서드별 재시도 정책을 JSON 구조(Map)로 지정한다.
     * 주의: 숫자는 반드시 Double로 넣는다 (JSON 숫자로 해석되기 때문).
     */
    private static Map<String, Object> retryServiceConfig() {
        Map<String, Object> retryPolicy = Map.of(
                "maxAttempts", 3.0,              // 첫 시도 포함 최대 3번
                "initialBackoff", "0.1s",        // 첫 재시도 전 대기 시간
                "maxBackoff", "1s",              // 대기 시간 상한
                "backoffMultiplier", 2.0,        // 재시도마다 대기 시간을 2배로
                // 재시도할 상태 코드. 값은 io.grpc.Status.Code enum의 이름과 같다:
                //   OK, CANCELLED, UNKNOWN, INVALID_ARGUMENT, DEADLINE_EXCEEDED, NOT_FOUND,
                //   ALREADY_EXISTS, PERMISSION_DENIED, RESOURCE_EXHAUSTED, FAILED_PRECONDITION,
                //   ABORTED, OUT_OF_RANGE, UNIMPLEMENTED, INTERNAL, UNAVAILABLE, DATA_LOSS,
                //   UNAUTHENTICATED
                // 보통 일시적 장애를 뜻하는 UNAVAILABLE만 넣는다.
                "retryableStatusCodes", List.of("UNAVAILABLE"));

        Map<String, Object> methodConfig = Map.of(
                // 조회(GetOrder)에만 적용한다. 멱등하지 않은 CreateOrder에 재시도를 걸면 주문이 중복될 수 있다.
                // "method"를 빼면 서비스의 모든 메서드에 적용된다.
                "name", List.of(Map.of(
                        "service", "order.v1.OrderService",
                        "method", "GetOrder")),
                "retryPolicy", retryPolicy);

        return Map.of("methodConfig", List.of(methodConfig));
    }

    /**
     * 로드 밸런싱 정책 이름은 문자열이라 오타가 컴파일 시점에 잡히지 않는다.
     * 애플리케이션 시작 시점에 등록 여부를 확인해서 빨리 실패하게 한다.
     */
    private static void validateLoadBalancingPolicy(String policy) {
        if (LoadBalancerRegistry.getDefaultRegistry().getProvider(policy) == null) {
            throw new IllegalStateException(
                    "등록되지 않은 로드 밸런싱 정책: " + policy + " (오타이거나 필요한 의존성이 없음)");
        }
    }
}
```

**③ 실제 호출: `OrderServiceClient`**

```java
@Component
public class OrderServiceClient {

    private final OrderServiceGrpc.OrderServiceBlockingStub stub;
    private final Duration deadline;

    public OrderServiceClient(OrderServiceGrpc.OrderServiceBlockingStub stub,
                              OrderGrpcClientProperties props) {
        this.stub = stub;
        this.deadline = props.deadline();
    }

    public Order getOrder(String orderId) {
        GetOrderRequest request = GetOrderRequest.newBuilder()
                .setOrderId(orderId)
                .build();
        try {
            return stub
                    // deadline은 "지금부터 N밀리초 뒤"라는 절대 시각이다. 반드시 호출할 때마다 새로 건다.
                    .withDeadlineAfter(deadline.toMillis(), TimeUnit.MILLISECONDS)
                    .getOrder(request)
                    .getOrder();   // 실제로는 여기서 도메인 객체로 변환해서 반환하는 편이 안전하다
        } catch (StatusRuntimeException e) {
            throw translate(e, orderId);
        }
    }

    /** gRPC 상태 코드를 애플리케이션 예외로 변환한다. */
    private RuntimeException translate(StatusRuntimeException e, String orderId) {
        return switch (e.getStatus().getCode()) {
            case NOT_FOUND -> new OrderNotFoundException(orderId);
            case DEADLINE_EXCEEDED, UNAVAILABLE -> new OrderServiceUnavailableException(e);
            default -> e;
        };
    }
}
```

### 4.2 주의: deadline을 Stub 빈에 걸면 안 되는 이유

```java
// 잘못된 예
@Bean
public OrderServiceGrpc.OrderServiceBlockingStub orderServiceBlockingStub(ManagedChannel channel) {
    return OrderServiceGrpc.newBlockingStub(channel)
            .withDeadlineAfter(500, TimeUnit.MILLISECONDS);   // 빈 생성 시점 + 500ms로 고정됨
}
```

- `withDeadlineAfter`는 "지금부터 500ms"를 계산해서 **절대 시각**으로 저장한다.
- 빈은 애플리케이션 시작 때 한 번 만들어지므로, deadline이 "시작 시각 + 500ms"로 고정된다.
- 시작하고 500ms가 지나면 **이후의 모든 호출이 즉시 `DEADLINE_EXCEEDED`로 실패**한다.
- 그래서 deadline은 4.1의 `OrderServiceClient`처럼 **호출할 때마다** 건다.

> 메서드별 기본 제한 시간은 서비스 설정의 `timeout` 항목으로도 지정할 수 있다. 이때 실제 적용되는 값은 서비스 설정의 값과 애플리케이션 코드에서 지정한 값 중 **더 짧은 쪽**이다.

### 4.3 Spring gRPC를 쓰면: 프로퍼티로 대체

Spring gRPC는 위 설정 대부분을 프로퍼티로 제공한다. Channel 생성과 종료, Stub 빈 등록도 프레임워크가 처리한다.

```yaml
# Spring gRPC 1.0 문서 기준. Spring Boot 4.1 통합 이후 접두어가 바뀌었을 수 있으니 사용하는 버전의 문서를 확인할 것
spring:
  grpc:
    client:
      channels:
        order-service:
          address: dns:///order-service:9090
          default-load-balancing-policy: round_robin
          default-deadline: 500ms          # 호출마다 적용되는 기본 deadline → 4.2의 함정을 프레임워크가 해결
          enable-keep-alive: true
          keep-alive-time: 5m
          keep-alive-timeout: 20s
          keep-alive-without-calls: false
          idle-timeout: 30m
          max-inbound-message-size: 4194304
          negotiation-type: plaintext      # plaintext / plaintext_upgrade / tls
```

| 방식 | 장점 | 단점 |
|---|---|---|
| grpc-java 직접 구성 (4.1) | 모든 설정이 코드로 드러남. 프레임워크와 무관하게 동작 | 종료 처리, 검증 등을 직접 작성 |
| Spring gRPC 프로퍼티 (4.3) | 코드가 거의 없음. 기본 deadline 같은 편의 기능 제공 | 프로퍼티로 표현되지 않는 설정은 커스터마이저 빈으로 추가해야 함 |

---

## 5. 언제 gRPC를 쓰나

### 5.1 전형적인 구조

```
[브라우저 / 외부 파트너] ──REST + JSON──▶ [API Gateway / BFF]
                                              │
                                       gRPC + Protobuf     (내부 동기 호출)
                                              ▼
                                        [내부 서비스] ──Kafka + Avro/Protobuf──▶ [다른 서비스]
```

- **JSON**(JavaScript Object Notation): 사람이 읽을 수 있는 텍스트 기반 데이터 형식
- **API Gateway**: 외부 요청을 받아 내부 서비스로 전달하는 단일 진입점
- **BFF**(Backend For Frontend): 특정 프론트엔드(웹, 앱)를 위해 만든 전용 백엔드. 여러 내부 서비스를 호출해 화면에 맞는 응답을 조립한다
- 사람이 보는 경계는 JSON, 기계끼리 대량으로 주고받는 내부는 바이너리 + 스키마.
- 모바일 앱은 브라우저 제약이 없어서 gRPC로 서버를 직접 호출하기도 한다.

### 5.2 여러 언어가 섞인 환경에서의 흐름

```
           [proto 저장소]  ← 계약의 단일 원본
                 │
     ┌───────────┼────────────┐
     ▼           ▼            ▼
 주문 서비스   결제 서비스    추천 서비스
  (Java)        (Go)         (Python)
 grpc-java     grpc-go      grpcio      ← 언어별 gRPC 런타임
```

**계약 하나로 모든 언어의 클라이언트가 자동으로 생긴다**는 점이 가장 큰 가치다. 특히 **서비스 경계가 곧 팀 경계일 때** 효과가 크다. 다른 팀의 코드를 읽지 않아도 proto만 보면 무엇을 주고받는지 알 수 있고, 계약이 깨지면 컴파일이나 CI(Continuous Integration, 지속적 통합) 단계에서 막힌다.

### 5.3 gRPC가 확실히 유리한 경우

| 상황 | 이유 |
|---|---|
| 내부 서비스 간 호출량이 매우 많음 | 직렬화 비용과 페이로드가 호출 수만큼 누적 |
| 지연 시간이 중요함 | 파싱 비용 감소, 연결 재사용 |
| 스트리밍이 필요함 | HTTP/2 위에서 양방향 스트리밍 기본 지원 |
| 여러 언어가 섞여 있음 | 계약 하나로 모든 언어 클라이언트 생성 |
| 계약을 엄격하게 관리하고 싶음 | 타입 강제, 호환성 검사 자동화 |
| 모바일, IoT(Internet of Things, 사물인터넷) | 대역폭, 배터리 |

### 5.4 크기 이득은 데이터 성격에 따라 다르다

- Protobuf가 JSON보다 작은 이유는 필드 이름 대신 태그 번호를 쓰고, 정수를 가변 길이로 부호화하기 때문이다.
- 숫자와 짧은 필드가 많은 메시지 → 이득이 크다.
- 긴 문자열이나 대용량 텍스트가 대부분인 메시지 → 값 자체가 크므로 이득이 작다.
- 그래서 크기보다 **계약 관리, 코드 생성, 스트리밍** 때문에 gRPC를 고르는 경우가 많다.

### 5.5 사람이 읽을 수 없다는 단점은 도구로 메운다

- **grpcurl**: curl의 gRPC 버전. 요청과 응답을 JSON으로 보여줌
- **Server reflection**: 서버가 자신의 proto 정의를 알려줘서 도구가 proto 파일 없이도 호출 가능
- **Postman, Insomnia**: gRPC 호출 지원
- **JSON 매핑**: Protobuf 공식 JSON 변환 규칙으로 로그에는 JSON으로 기록

---

## 6. gRPC와 이벤트 기반 아키텍처

### 6.1 핵심 차이: "지금 답이 필요한가"

| | gRPC (동기 요청/응답) | 이벤트 기반 (비동기 메시지) |
|---|---|---|
| 메시지의 의미 | **명령 / 질문**: "재고 확인해줘" | **사실 통보**: "주문이 생성됐다" |
| 보내는 쪽이 아는 것 | 누구에게 보내는지 앎 | 누가 받는지 모름 |
| 응답 | 기다림 | 기다리지 않음 |
| 받는 쪽이 죽으면 | 호출 실패 | 메시지가 쌓였다가 나중에 처리 |
| 결합도 | 시간적 결합 (둘 다 살아 있어야 함) | 시간적으로 분리 |
| 일관성 | 즉시 결과 확인 | 결국 일관성 |

책의 데이터플로 분류로 보면 gRPC는 **서비스를 통한 흐름**, 이벤트 기반은 **비동기 메시지 전달**이다.

### 6.2 둘은 함께 쓴다

```
[클라이언트] ──REST──▶ [주문 서비스]
                          ├─ gRPC ─▶ [재고 서비스]   "재고 있어?"  ← 답이 있어야 주문 가능
                          ├─ gRPC ─▶ [결제 서비스]   "결제해줘"    ← 성공해야 주문 확정
                          └─ Kafka: OrderCreated 발행
                                 ├──▶ [알림 서비스]
                                 ├──▶ [포인트 서비스]
                                 └──▶ [정산 서비스]
```

**판단 기준:** 이 결과 없이 다음 단계로 갈 수 있는가?

### 6.3 통신 방식과 부호화 형식은 별개의 축

|  | JSON | Protobuf |
|---|---|---|
| **동기 호출** | REST + JSON | gRPC |
| **비동기 이벤트** | Kafka + JSON | Kafka + Protobuf (+ Schema Registry) |

---

## 7. proto 파일 관리

현업의 proto 관리는 **"계약 하나를 여러 팀과 여러 언어가 어떻게 안전하게 공유하느냐"**의 문제다.

### 7.1 어디에 두나

| | 서비스 저장소 내부 | 중앙 proto 저장소 | 스키마 레지스트리 |
|---|---|---|---|
| 소유권 | 명확 (서버 팀) | CODEOWNERS(GitHub에서 디렉터리별 리뷰 담당자를 지정하는 파일)로 지정 | 모듈별 소유자 |
| 변경 속도 | 빠름 | 느림 (계약 PR(Pull Request)과 구현 PR 분리) | 중간 |
| 전체 계약 파악 | 어려움 | 쉬움 | 쉬움 |
| 일관된 규칙 적용 | 저장소마다 설정 | 한 번 설정으로 전체 적용 | 레지스트리가 강제 |
| 적합한 규모 | 서비스 수가 적은 조직 | 팀이 여러 개 | 언어와 팀이 많음 |

### 7.2 어떻게 나눠주나

| | 소비자가 직접 생성 | 생성 코드를 라이브러리로 배포 | 레지스트리 SDK(Software Development Kit) 자동 생성 |
|---|---|---|---|
| 방식 | submodule 등으로 proto를 가져와 빌드 시 생성 | CI에서 생성 후 사내 Maven 등에 배포 | 레지스트리가 Maven, npm 패키지 제공 |
| 소비자 부담 | 큼 | 작음 | 작음 |
| 운영 부담 | 작음 | 큼 (언어별 파이프라인) | 작음 (외부 서비스 비용) |

> 가장 위험한 안티패턴은 proto 파일을 **복사해서** 쓰는 것이다. 복사본은 원본과 어긋나고 계약이 두 개가 된다.

### 7.3 어떻게 바꾸나

```
① proto 수정 PR
② CI: buf lint, buf breaking (main 대비 호환성 검사), 코드 생성 테스트
③ CODEOWNERS 승인
④ 병합 → 태그 → 생성 코드 배포
⑤ 서버 배포 → 소비자가 필요할 때 버전 올림
```

**`buf breaking` 검사 수준은 책의 호환성 개념과 직결된다.**

| 수준 | 무엇을 지키나 | 필드 이름 변경 |
|---|---|---|
| `WIRE` | 바이너리 부호화 호환성만 | 허용 (바이트 열에는 태그 번호만 있음) |
| `WIRE_JSON` | 바이너리 + JSON 변환 | 금지 (JSON에는 이름이 들어감) |
| `PACKAGE` / `FILE` | 생성 코드의 소스 호환성까지 | 금지 |

> 책의 "태그 번호로 식별하니 이름을 바꿔도 된다"는 `WIRE` 수준의 이야기다. 생성 코드를 쓰는 현업에서는 보통 `FILE`이나 `PACKAGE`를 쓴다.

**호환되지 않는 변경이 꼭 필요하면** 패키지 버전을 올려 공존시킨다 (expand-contract).

1. `order/v2` 추가, 서버가 v1과 v2 모두 제공. v1에는 `option deprecated = true;`
2. 소비자들이 v2로 전환 (메트릭으로 v1 호출량 추적)
3. v1 호출이 0이 되면 제거

| | 패키지 버전 | 라이브러리 버전 |
|---|---|---|
| 형태 | `order.v1`, `order.v2` | `order-api:1.4.0` |
| 바뀌는 시점 | 호환되지 않는 변경 (드묾) | proto가 바뀔 때마다 |
| 의미 | 계약의 세대 | 배포물의 스냅샷 |

### 7.4 작성 규칙

| 규칙 | 이유 |
|---|---|
| 디렉터리 경로와 패키지 일치, 패키지 끝에 버전 (`order.v1`) | 호환되지 않는 변경을 새 패키지로 분리 |
| RPC마다 전용 요청/응답 메시지 | 나중에 필드를 추가할 공간 확보 |
| enum 0번은 `*_UNSPECIFIED` | proto3는 0을 전송하지 않아 "값 없음"과 구분 불가 |
| enum 값에 enum 이름 접두어 (`ORDER_STATUS_PAID`) | enum 값은 패키지 범위를 공유해서 이름 충돌 가능 |
| 시간은 `google.protobuf.Timestamp` | 단위 혼란 방지 |
| ID는 `string` | `int64`는 JavaScript에서 2^53 초과 시 정밀도 손실 (책의 트위터 ID 사례) |
| 금액은 정수 | `double`의 부동소수점 오차 |


---

---

## 8. 운영 단계에서 알아야 할 것

### 8.1 호출 설계

| 항목 | 내용 |
|---|---|
| deadline 필수 | 설정하지 않으면 무한 대기. 호출 체인을 따라 전파되는지 확인 |
| 재시도 정책 | service config로 메서드별 재시도 횟수, 대상 상태 코드, 백오프 설정 |
| 멱등성 | 재시도 대상은 멱등해야 함. 변경 작업은 멱등성 키 |
| 메시지 크기 제한 | 기본 최대 수신 4MB. 넘으면 `RESOURCE_EXHAUSTED`. 올리기 전에 스트리밍 검토 |

### 8.2 에러 모델

| 항목 | 내용 |
|---|---|
| 상태 코드 매핑 규칙 | 주문 없음 → `NOT_FOUND`, 재고 부족 → `FAILED_PRECONDITION` 등 팀 규칙 |
| `UNKNOWN`, `INTERNAL` 남용 금지 | 처리하지 않은 예외는 `UNKNOWN`으로 나가서 클라이언트가 재시도 여부를 판단할 수 없음 |
| 재시도 판단 | 재시도할 만한 코드(`UNAVAILABLE`)와 안 되는 코드(`INVALID_ARGUMENT`) 구분 |
| 상세 에러 | `google.rpc.Status`의 details로 전달 |
| 모니터링 | gRPC는 호출이 실패해도 HTTP 상태가 대부분 200이고, 실제 결과는 응답 끝의 `grpc-status`에 담긴다. 에러율은 반드시 `grpc-status` 기준으로 집계 |

### 8.3 연결과 로드 밸런싱

| 항목 | 내용 |
|---|---|
| Channel 재사용 | 호출마다 생성하면 연결 폭증 |
| 로드 밸런싱 | L4(OSI 4계층, 전송 계층) 로드 밸런서는 **연결 단위**로 분배 → HTTP/2는 연결 하나를 오래 쓰므로 한 서버로 쏠림. L7(OSI 7계층, 애플리케이션 계층) 프록시(Envoy, 서비스 메시)나 클라이언트 측 밸런싱으로 해결. Kubernetes에서는 headless Service |
| 연결 수명 제한 | 서버에 최대 연결 수명을 두면 클라이언트가 주기적으로 재연결하며 새 서버로 부하 분산 |
| keepalive | 유휴 연결이 중간 장비에서 끊기는 것 방지. 단, ping이 너무 잦으면 서버가 `too_many_pings`로 끊음 |
| 헬스 체크 | 표준 헬스 체크 서비스(`grpc.health.v1`). Kubernetes는 gRPC 프로브 지원 |

### 8.4 관측성, 보안, 테스트

| 영역 | 항목 |
|---|---|
| 관측성 | 메서드 × 상태 코드별 메트릭, 인터셉터로 트레이스 ID 전파(OpenTelemetry), Protobuf를 JSON으로 로그 기록, 운영 환경에서는 reflection 제한 |
| 보안 | TLS(Transport Layer Security) / 서비스 간 mTLS(mutual TLS, 서버와 클라이언트가 서로 인증서를 확인하는 TLS), metadata 토큰을 인터셉터에서 검증, 요청 크기와 동시 스트림 수 제한 |
| 테스트 | In-process 서버로 네트워크 없이 테스트, `buf breaking`으로 계약 테스트 |
| 외부 경계 | grpc-gateway로 REST 동시 제공, 브라우저는 gRPC-Web 또는 Connect, Kubernetes Gateway API의 `GRPCRoute` |

---

## 9. 최근 동향 (2026년 기준)

gRPC는 새로 뜨는 기술이 아니라 **이미 자리 잡은 인프라 표준**이다. 최근 변화는 이 표준의 적용 범위가 넓어지는 흐름이다.

| 흐름 | 내용 |
|---|---|
| **Spring 공식 지원** | 커뮤니티 라이브러리(`net.devh:grpc-spring-boot-starter`) 중심에서 공식 Spring gRPC로 전환. Spring Boot 4.1부터 gRPC 서버·클라이언트 지원이 Spring Boot 본체에 포함되고 스타터 좌표가 `spring-boot-starter-grpc-server` / `-client`로 이동 |
| **AI 에이전트 연결** | MCP(Model Context Protocol)에서 gRPC를 **선택형 커스텀 전송**으로 쓰려는 움직임. 공식 표준 전송이 된 것은 아님 (9.1 참고) |
| **브라우저 대응** | Connect RPC(CNCF Sandbox)가 gRPC, gRPC-Web, 자체 프로토콜을 모두 지원해서 같은 proto로 브라우저까지 처리 |
| **언어 확장** | Rust 구현체 Tonic이 gRPC 공식 프로젝트로 편입, gRPC-Rust 프리뷰 공개 (아직 프로덕션 비권장) |


### 9.1 AI 에이전트 연결: MCP와 gRPC (2026년 10월 기준 검증)

**MCP**(Model Context Protocol)는 AI 에이전트가 외부 도구와 데이터에 접근하는 표준 프로토콜이다. 메시지는 JSON-RPC 형식이다.

**배경**
- gRPC를 표준으로 쓰는 기업이 MCP를 도입하려면, JSON-RPC 요청을 gRPC로 바꿔주는 변환 게이트웨이(transcoding gateway)를 따로 둬야 했다.

**사실 관계 (1차 출처 기준)**

| 시점 | 주체 | 내용 |
|---|---|---|
| 2025-12 | MCP 전송 워킹그룹 | 공식 전송은 **STDIO**(Standard Input/Output, 로컬용)와 **Streamable HTTP**(원격용) 두 가지만 유지. 특수한 요구는 **커스텀 전송**으로 해결하고, SDK에서 커스텀 전송을 쉽게 붙일 수 있게 개선하겠다고 발표 |
| 2026-01 | Google Cloud | MCP 메인테이너들이 SDK에 교체 가능한 전송(pluggable transport)을 지원하기로 합의했다고 밝히고, gRPC 전송 패키지를 기여·배포하겠다고 발표. 블로그 제목도 "custom transport" |
| 2026-01 | Spotify (Google 블로그 인용) | 백엔드 표준이 gRPC라서 사내에서 MCP over gRPC를 실험적으로 지원 중. 정적 타입 API 덕분에 MCP 서버 작성이 쉬워졌다고 언급 |
| 2026-04 | Google | gRPC 전송용 proto 정의 패키지(`mcp-grpc-transport-proto` 0.1.0)를 PyPI(Python Package Index)에 공개 |
| 2026-07 | MCP 스펙 2026-07-28 | 프로토콜이 상태 없는(stateless) 구조로 바뀜. Streamable HTTP에 `Mcp-Method`, `Mcp-Name` 헤더가 생겨서 로드 밸런서가 본문을 열지 않고도 라우팅 가능 |

**정리**
- gRPC는 MCP의 **공식 전송이 아니라, 필요한 조직이 골라 쓰는 커스텀 전송**이다.
- 일부 블로그는 "gRPC가 MCP 표준 전송으로 제안·채택되었다"는 식으로 소개하지만, MCP 공식 블로그와 Google 블로그 원문은 커스텀 전송으로 설명한다.
- 한편 2026-07-28 스펙은 Streamable HTTP 쪽에서도 라우팅과 확장성 문제를 직접 개선했다. 즉 "JSON-RPC는 기업 인프라에 안 맞는다"는 문제의 일부는 gRPC가 아닌 방식으로도 풀리고 있다.

**면접에서 쓸 수 있는 관점**
- 이미 gRPC로 서비스 계약을 엄격하게 관리하는 조직이라면, 같은 proto 계약과 운영 도구(deadline, mTLS, 트레이싱)를 AI 에이전트 연결에도 그대로 쓰고 싶어 한다.
- 이 흐름은 "외부는 표준(JSON), 내부는 gRPC"라는 기존 경계 패턴이 AI 에이전트 영역에서도 반복되는 것으로 볼 수 있다.

---

## 10. 한 줄 요약

- gRPC = `.proto` 계약 + 생성된 Stub + HTTP/2 스트림 위의 Protobuf 메시지
- Channel은 한 번 만들어 재사용하고, 모든 호출에는 deadline을 건다
- 가장 큰 가치는 크기보다 **여러 언어와 팀 사이의 엄격한 계약**
- 동기 호출(gRPC)과 이벤트(Kafka)는 경쟁이 아니라 "지금 답이 필요한가"로 나눠 쓰는 것
- proto 관리의 핵심은 **호환성 검사를 CI로 강제**하는 것 → 책의 스키마 발전 규칙을 기계가 지킨다
