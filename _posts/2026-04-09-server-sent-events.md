---
title: "Server-Sent Events 애플리케이션"
date: 2026-04-09
---

## 프로젝트 소개
  - 서버가 클라이언트에게 HTTP 연결을 통해 실시간으로 데이터를 push하는 SSE를 간략하게 구현한다.

## 깃헙 레포지토리
  - [Server-Sent Events Example](https://github.com/taehwanyang/sse-example)

## 개발 스택
  - Java : Spring Boot

## Server-Sent Events
  - 서버 -> 클라이언트 단방향 실시간 통신
  - text/event-stream 형식으로 이벤트를 계속 스트리밍한다.
    - 알림
    - 실시간 로그
    - AI 답변 스트리밍

```sh
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

  - SSE 메시지 형식

```sh
event: notification
data: client connected
```

  - WebSocket과의 차이점
    - WebSocket은 클라이언트, 서버 양방향 (채팅, 게임, 협업툴)
    - WebSocket은 구현/운영 복잡도가 높고 SSE는 구현이 단순하고 상태 변경 통지 등 서버 푸시에 적합
    - SSE는 프록시/로드밸런서 환경에서 상대적으로 다루기 쉽다. 

## 동작 순서

###  클라이언트 연결
  - 브라우저가 /subscribe 호출
  - 해당 Spring Boot 인스턴스가 SseEmitter 생성
  - 현재 인스턴스 메모리에 저장

### 이벤트 발행
  - 어느 인스턴스에서 POST /notifications/broadcast 호출
  - RedisPubSubService.publish()
  - Redis 채널에 JSON 메시지 발행

### Redis 메시지 수신
  - 모든 인스턴스의 RedisSubscriber가 그 메시지 받음

### 각 인스턴스에서 로컬 SSE 전송
  - 각 인스턴스는 자기 SseEmitterRepository에 있는 클라이언트들에게만 broadcastLocal()

## Redis 실행

```shell
docker run --name local-redis -p 6379:6379 -d redis:7
```

## 테스트 : 클라이언트로 브라우저 이용

### 인스턴스 2개 실행
  - 인스턴스 1은 8080에서 실행

```shell
./gradlew bootRun
```

  - 인스턴스 2는 8081에서 실행

```shell
./gradlew bootRun --args='--server.port=8081'
```

  - 브라우저 2개를 띄워서
    - http://localhost:8080
    - http://localhost:8081
  - 8080 쪽으로 이벤트 발행
```shell
curl -X POST http://localhost:8080/notifications/broadcast \
  -H "Content-Type: application/json" \
  -d '{"message":"hello from 8080"}'
```

## 테스트 : 클라이언트로 curl 이용

### 인스턴스 1개 실행

```shell
./gradlew bootRun
```

### 구독(SSE 연결)

  - 터미널을 2개 연다.
  - 두개의 터미널에서 curl로 SSE 연결
```shell
curl -N http://localhost:8080/subscribe
```

### 다른 터미널에서 이벤트 발생
  - 터미널을 연다.
  - 터미널에서 이벤트 발생

```shell
curl -X POST http://localhost:8080/notifications/broadcast \
  -H "Content-Type: application/json" \
  -d '{"message":"hello"}'
```

## 쿠버네티스 운영시 주의할 점
  - 쿠버네티스에 애플리케이션을 파드로 올린다면 앞단에 nginx ingress controller가 있거나 Azure의 경우 애플리케이션 게이트웨이가 있을 것이다.

### Nginx Ingress Controller의 경우
  - 이때는 ingress를 일반 API와 SSE로 나누어야 한다.
  - 일반 API용 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: sseexample.ythwork.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: sse-example
                port:
                  number: 8080
```

  - SSE 전용 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sse-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
spec:
  ingressClassName: nginx
  rules:
    - host: sseexample.ythwork.com
      http:
        paths:
          - path: /subscribe
            pathType: Prefix
            backend:
              service:
                name: sse-example
                port:
                  number: 8080
```

  - nginx 설정은 아래와 같이 될 것이다.

```sh
location /api {
    proxy_pass ...
}

location /subscribe {
    proxy_buffering off;
    proxy_read_timeout 3600;
}
```

  - Spring에는 아래와 같이 헤더를 추가한다.

```java
@GetMapping(value = "/subscribe", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter subscribe(HttpServletResponse response) {
    response.setHeader("Cache-Control", "no-cache");
    response.setHeader("X-Accel-Buffering", "no");
    return sseService.subscribe();
}
``` 

## 테스트 : nginx ingress controller가 있을 때
  - 쿠버네티스에 파드로 애플리케이션을 배포한다면 클라이언트와 서버 사이에 nginx ingress controller가 있다.
  - 이 경우 nginx가 데이터를 모았다가 보낼 수도 있음.
  - 이 경우 sse 엔드포인트와 일반 API ingress를 분리한다.
  - 레디스와 sse-example 애플리케이션을 배포

```shell
cd k8s
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml

kubectl apply -f sse-example-deploy.yaml
kubectl apply -f sse-example-service.yaml
kubectl apply -f sse-example-ingress-sse.yaml
kubectl apply -f  sse-example-ingress-api.yaml
```

  - 쿠버네티스에 배포된 ingress controller가 nginx라고 가정한다.
  - /etc/hosts에 도메인을 추가한다.

```shell
127.0.0.1       sse.ythwork.com
```

  - 요청 URL만 sse.ythwork.com으로 변경하여 테스트한다.

```shell
curl -N http://sse.ythwork.com/subscribe
```

```shell
curl -X POST http://sse.ythwork.com/notifications/broadcast \
  -H "Content-Type: application/json" \
  -d '{"message":"hello"}'
```

## 구현 상세

  - 클라이언트로 보낼 DTO

```java
public class BroadcastMessage {

    private String eventName;
    private String message;
    private Long timestamp;
    private String senderInstanceId;
    // ...
}
```

  - 클라이언트가 /subscribe 호출

```java
@RestController
public class SseController {
    // ...
    @GetMapping(value = "/subscribe", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe() {
        return sseService.subscribe();
    }
}
```

  - 해당 Spring Boot 애플리케이션에서 SseEmitter 생성

```java
@Service
public class SseService {
    // ...
    public SseEmitter subscribe() {
        // ...
        SseEmitter emitter = new SseEmitter(timeout);
        // ..
        try {
            emitter.send(SseEmitter.event()
                    .name("connect")
                    .data("connected: " + clientId));
        } catch (IOException | IllegalStateException ex) {
            // ...
        }
        return emitter;
    }
    // ...
}
```

  - 클라이언트가 Spring Boot 애플리케이션 중 하나에 POST /notification/broadcast 호출

```java
@RestController
@RequestMapping("/notifications")
public class NotificationController {
    // ...
    @PostMapping("/broadcast")
    public String broadcast(@RequestBody NotificationRequest request) {
        BroadcastMessage message = new BroadcastMessage(
                "notification",
                request.getMessage(),
                System.currentTimeMillis(),
                "server-" + port
        );

        redisPubSubService.publish(message);

        return "published. local connected clients = " + sseService.count();
    }
    // ...
}
```

  - Redis 채널에 메시지 publish

```java
@Service
public class RedisPubSubService {
    // ...
    public void publish(BroadcastMessage message) {
        try {
            String payload = objectMapper.writeValueAsString(message);
            stringRedisTemplate.convertAndSend(channel, payload);
        } catch (JsonProcessingException e) {
            // ...
        }
    }
}
```

  - 모든 Spring Boot 애플리케이션에서 메시지 수신

```java
@Component
public class RedisSubscriber implements MessageListener {
    // ...
    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            String body = new String(message.getBody());
            BroadcastMessage broadcastMessage =
                    objectMapper.readValue(body, BroadcastMessage.class);
            // ...
            sseService.broadcastLocal(
                    broadcastMessage.getEventName(),
                    broadcastMessage
            );
        } catch (Exception e) {
            // ...
        }
    }
}
```

  - 모든 Spring Boot 애플리케이션에서 로컬 SSE 전송

```java
@Service
public class SseService {
    // ...
    public void broadcastLocal(String eventName, Object data) {
        // ...
        repository.findAll().forEach((clientId, emitter) -> {
            try {
                emitter.send(SseEmitter.event()
                        .name(eventName)
                        .data(data));
            } catch (IOException | IllegalStateException ex) {
                // ...
            }
        });
        // ...
    }
}
```
