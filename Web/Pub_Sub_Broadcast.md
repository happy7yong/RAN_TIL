# Publish / Subscribe / Broadcast / Broker

## 핵심 정의

- `publish` = 메시지 발행
- `subscribe` = 메시지 수신 등록
- `broker` = 메시지 중개자
- `broadcast` = 여러 구독자에게 동시에 전달하는 방식

## 브로드캐스트 구조

```text
A (publisher)
   ↓ publish
/topic/chat/1  (destination)
   ↓
B (subscriber)
C (subscriber)
D (subscriber)
```

## STOMP에서의 구조

Spring STOMP에서 아래 설정을 사용하면:

```java
enableSimpleBroker("/topic", "/queue");
```

브로커는 아래 역할을 수행한다.

1. 발행된 메시지를 destination 기준으로 받는다.
2. 해당 destination을 구독한 클라이언트를 찾는다.
3. 구독자에게 메시지를 분배한다.

즉 흐름은 `Publisher -> Broker -> Subscribers` 이다.

## 채널 종류 차이

| 채널     | 의미                | 수신 형태                   |
| -------- | ------------------- | --------------------------- |
| `/topic` | 브로드캐스트 채널   | 여러 명 수신                |
| `/queue` | 단일/개인 지향 채널 | 한 명 또는 특정 사용자 수신 |

---

## 개념 정의

Publish/Subscribe는 실시간 메시징 모델이다.  
핵심 구조는 `Publisher -> Topic(Channel) -> Subscribers` 형태이며, 발행자와 수신자를 느슨하게 분리한다.  
여기서 `publish`는 채널로 메시지를 보내는 동작이고, `subscribe`는 채널 메시지를 받기 위해 등록하는 동작이다.  
`broadcast`는 하나의 메시지가 같은 채널을 구독한 다수에게 fan-out 되는 전달 결과이다.

## 동작 원리

1. 발행자가 특정 destination으로 메시지를 `publish` 한다.
2. 브로커가 destination별 구독자 목록을 조회한다.
3. 구독자 수만큼 메시지를 분배한다.
4. 구독자가 없으면 메시지는 버려지거나 브로커 정책에 따라 처리된다.
5. 구독 등록이 없는 클라이언트는 발행이 있어도 메시지를 받지 못한다.

여기서 중요한 포인트는 요청 기반과 등록 기반의 차이이다.

- HTTP는 요청을 보낼 때 응답을 받는 요청 기반 모델이다.
- Pub/Sub는 구독을 먼저 등록해두고 이벤트를 받는 등록 기반 모델이다.
- 이 차이를 이해하면 `메시지가 왜 안 오는가` 문제를 훨씬 빠르게 찾을 수 있다.

## 코드

```text
# 1) 채팅방 구독 (subscribe)
SUBSCRIBE
id:sub-chat-1
destination:/topic/chat/1

\0
```

```text
# 2) 채팅방 메시지 발행 (publish)
SEND
destination:/topic/chat/1
content-type:application/json

{"type":"CHAT_MESSAGE","roomId":1,"content":"hi"}\0
```

```text
# 3) 브로커가 구독자에게 전달하는 수신 프레임
MESSAGE
subscription:sub-chat-1
destination:/topic/chat/1
content-type:application/json

{"type":"CHAT_MESSAGE","roomId":1,"content":"hi"}\0
```

```text
# 4) 개인 채널 예시 (broadcast와 목적이 다름)
SUBSCRIBE
id:sub-user-handshake
destination:/user/queue/handshake

\0
```

위 1번은 수신 등록 단계이다.  
위 2번은 채널로 이벤트를 보내는 발행 단계이다.  
위 3번은 브로커가 실제 수신자에게 전달한 결과이며, `subscription` 값으로 어떤 구독에서 받은 메시지인지 식별한다.  
위 4번은 개인 채널 구독 예시이며, 다수 fan-out이 아니라 사용자 단위 응답에 가깝다.

## 언제 쓰는지

- 채팅, 실시간 알림, 협업 기능처럼 한 이벤트를 여러 사용자에게 동시에 전달해야 할 때 사용한다.
- 발행자와 수신자를 분리해 서비스 결합도를 낮추고 확장성을 확보해야 할 때 사용한다.
- STOMP 디버깅 시 `subscribe 누락`, `destination 오타`, `/topic`과 `/queue` 혼동을 분리 점검할 때 유용하다.
- 단순 조회성 기능에는 Pub/Sub보다 REST 요청-응답이 더 단순하고 운영 비용이 낮다.

## 핵심 한 줄

Publish는 메시지를 채널로 보내는 동작이고 Subscribe는 수신 등록이며, Broker가 destination 기준으로 분배해서 Broadcast가 만들어진다.
