## 개념 정의
STOMP에서 `destination`은 메시지가 흐르는 논리적 주소이다.  
이 값은 HTTP의 URL과 비슷한 역할을 하지만, 요청-응답 엔드포인트가 아니라 메시징 채널 주소라는 점이 다르다.  
반면 이벤트 이름은 STOMP 프로토콜 필드가 아니라 payload 내부에서 메시지 종류를 구분하는 애플리케이션 레벨 값이다.  
즉 `destination`은 어디로 보내는지에 대한 정보이고, `event type`은 무엇을 보냈는지에 대한 정보이다.

## 동작 원리
1. 클라이언트가 `SEND` 프레임으로 `destination`을 지정하면 서버는 해당 주소 기준으로 메시지 매핑 로직을 실행한다.  
2. 클라이언트가 `SUBSCRIBE` 프레임으로 `destination`을 등록하면 브로커는 그 주소로 발행된 메시지를 수신 대상으로 연결한다.  
3. 같은 destination 안에서도 payload의 `type` 필드로 `CHAT_MESSAGE`, `USER_JOIN`, `TYPING` 같은 이벤트 종류를 분기 처리한다.  
4. Spring STOMP에서는 보통 `/pub`는 서버 진입 경로, `/topic`은 브로드캐스트, `/queue`와 `/user`는 개인 수신 경로로 분리한다.  
5. 따라서 설계 시 destination 체계와 이벤트 타입 체계를 동시에 정의해야 메시지 라우팅과 비즈니스 처리 로직이 모두 안정적으로 동작한다.

여기서 깨달은 점은 destination만 정확해도 메시지가 처리될 것 같지만, 실제 기능 분기는 payload type이 없으면 결국 if-else 지옥으로 무너진다는 사실이다.

## 코드
```java
// Spring STOMP prefix 구성 예시
@Override
public void configureMessageBroker(MessageBrokerRegistry config) {
    config.setApplicationDestinationPrefixes("/pub");
    config.enableSimpleBroker("/topic", "/queue");
    config.setUserDestinationPrefix("/user");
}
```

```text
# 서버로 "소켓 연결 이벤트" 발행
SEND
destination:/pub/socket.connect
content-type:application/json

{"type":"SOCKET_CONNECTED","deviceId":"cfc8e4af-f7d7-4f68-8b74-dd582cc45409"}\0
```

```text
# 채팅방 1번 구독
SUBSCRIBE
id:sub-chat-1
destination:/topic/chat/1

\0
```

```json
{
  "type": "TYPING",
  "roomId": 1,
  "userId": 42,
  "isTyping": true
}
```

위 Java 코드는 destination prefix를 어디에 쓸지 서버에서 정의하는 지점이다.  
위 STOMP 프레임은 `destination`이 실제 전송 주소로 어떻게 들어가는지 보여주는 실행 예시이다.  
마지막 JSON은 같은 `/topic/chat/1` 채널에서도 이벤트 타입이 다르면 처리 로직이 달라진다는 점을 보여주는 payload 예시이다.

## 언제 쓰는지
- STOMP 채팅 설계에서 `destination`과 이벤트 필드 역할이 섞여 코드가 복잡해질 때 기준 모델로 사용한다.  
- `/topic/chat/1` 하나에서 채팅, 입장, 퇴장, 타이핑 이벤트를 함께 처리해야 할 때 메시지 스키마를 정리하는 기준으로 사용한다.  
- WebSocket 로그를 해석할 때 `주소 문제`인지 `이벤트 분기 문제`인지 빠르게 분리 진단할 때 유용하다.  
- 다만 이벤트 타입만 늘리고 destination 설계를 방치하면 구독 단위가 커져 불필요한 메시지 수신 비용이 증가하므로 둘의 균형 설계가 필요하다.

## 핵심 한 줄
STOMP에서 destination은 메시지의 주소이고 event type은 메시지의 의미이므로, 둘을 분리 설계해야 라우팅과 비즈니스 처리가 동시에 단순해진다.
