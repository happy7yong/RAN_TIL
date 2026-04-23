# WebSocket 연결 수명주기: TCP, URL, STOMP, 이벤트 통신까지

## 0. 질문에 대한 직답

질문: `TCP 핸드셰이크 자체가 첫번째로는 연결할 엔드포인트에 요청은 해야되기 때문에 URL이 필요하고, 그 이후는 이벤트로 대화하는가?`

핵심 답은 아래와 같다.

- 초기 연결에는 대상 식별 정보가 필요하다.
- 다만 TCP 핸드셰이크 자체는 `URL 경로`가 아니라 `IP + Port`에 대해 수행된다.
- `ws://.../ws`의 `/ws` 같은 경로는 HTTP Upgrade 단계에서 사용된다.
- WebSocket으로 업그레이드가 끝난 뒤에는 HTTP 재요청/응답이 아니라 프레임 기반 양방향 통신으로 동작한다.
- 연결이 끊기면 다시 처음부터(DNS/TCP/TLS/Upgrade/STOMP) 재수립해야 한다.

즉, `초기 연결은 엔드포인트 정보가 필요하고, 연결 유지 중에는 이벤트 프레임으로 대화한다`가 정확한 이해이다.

---

## 1. 계층을 먼저 분리해야 헷갈리지 않는다

아래 순서를 분리해서 보면 대부분의 혼동이 사라진다.

```text
[Application]      STOMP frame (CONNECT/SUBSCRIBE/SEND/MESSAGE)
[Message Channel]  WebSocket frame
[HTTP Layer]       Upgrade: websocket (초기 1회)
[Transport]        TCP (wss면 TCP 위에 TLS)
[Network]          IP 라우팅
```

자주 나오는 오해는 아래 두 가지이다.

- 오해 1: TCP가 URL로 연결된다고 이해하는 경우이다.
- 오해 2: WebSocket 이후에도 HTTP 요청/응답처럼 매번 재요청한다고 이해하는 경우이다.

정확한 해석은 아래와 같다.

- TCP는 URL 문자열을 직접 쓰지 않는다. DNS를 거쳐 얻은 IP와 Port로 연결한다.
- WebSocket은 HTTP Upgrade 이후 프레임 채널이 되므로, 통신 단위는 HTTP 요청이 아니라 WebSocket frame이다.

---

## 2. `ws://` URL을 열었을 때 실제로 일어나는 단계

예시 URL:

```text
wss://stg.molip.today/ws
```

### 2-1. URL 해석

클라이언트는 URL에서 아래 정보를 분해한다.

- Scheme: `wss` (TLS 필요)
- Host: `stg.molip.today`
- Port: 생략 시 기본값 사용 (`ws`는 80, `wss`는 443)
- Path: `/ws`

여기서 중요한 점은 아래이다.

- `Host`는 DNS 조회 대상이다.
- `Path`는 TCP 단계가 아니라 HTTP Upgrade 요청 라인에서 사용된다.

### 2-2. DNS 조회

```text
stg.molip.today -> 203.0.113.10 (예시)
```

도메인을 IP로 변환한다. 아직 연결은 시작되지 않았다.

### 2-3. TCP 3-way Handshake

```text
Client -> SYN -> Server
Client <- SYN-ACK <- Server
Client -> ACK -> Server
```

이 단계는 `IP + Port` 기준 연결이다.

- `URL path(/ws)`는 아직 쓰지 않는다.
- TCP 입장에서는 애플리케이션 경로를 알지 못한다.

### 2-4. TLS Handshake (`wss`인 경우)

`wss`라면 TCP 위에서 TLS를 먼저 성립한다.

- 서버 인증서 검증
- 암호 스위트 협상
- 세션 키 합의

이 단계가 끝나야 암호화된 HTTP 요청(Upgrade 요청)을 보낼 수 있다.

`ws`라면 이 단계는 없다.

### 2-5. HTTP Upgrade (WebSocket 핸드셰이크)

여기서 처음으로 `/ws` 경로가 등장한다.

클라이언트 요청 예시:

```http
GET /ws HTTP/1.1
Host: stg.molip.today
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Version: 13
```

서버 응답 예시:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
```

`101 Switching Protocols`가 오면 HTTP 세션이 WebSocket 채널로 전환된다.

### 2-6. WebSocket frame 통신

이 시점부터는 아래가 성립한다.

- HTTP 요청/응답 모델이 아니다.
- 지속 연결 위에서 데이터 프레임을 주고받는다.
- 서버도 클라이언트에게 비동기로 push할 수 있다.

### 2-7. STOMP를 사용하면 프레임 payload에 STOMP 문법을 실어 보낸다

예시 STOMP 프레임:

```text
CONNECT
accept-version:1.2
host:stg.molip.today

\0
```

```text
SUBSCRIBE
id:sub-1
destination:/topic/chat/1

\0
```

```text
SEND
destination:/app/chat/send
content-type:application/json

{"roomId":1,"text":"hello"}\0
```

정리하면, WebSocket은 운반 채널이고 STOMP는 메시지 문법이다.

---

## 3. "재요청"이라는 단어를 정확히 바꿔야 한다

실무에서 자주 나오는 문장:

- "메시지 보낼 때마다 요청한다"

이 문장은 HTTP 문맥에서는 맞지만 WebSocket 문맥에서는 어휘가 부정확하다.

정확한 표현은 아래이다.

- 메시지 보낼 때마다 HTTP 요청을 새로 만드는 것이 아니다.
- 이미 열린 소켓 위로 WebSocket 데이터 프레임을 전송하는 것이다.

비교하면 아래와 같다.

```text
HTTP: 요청 1회 -> 응답 1회 -> 연결/재사용 정책
WebSocket: 연결 1회 수립 -> 프레임 N회 송수신
```

그래서 채팅, 알림, 실시간 보드에 유리하다.

---

## 4. 연결과 구독은 다른 문제이다

### 연결(Connection)

- 물리/전송 통로가 살아 있는 상태이다.
- `connected = true`여도 특정 토픽을 듣고 있다는 뜻은 아니다.

### 구독(Subscribe)

- 열린 연결 위에서 특정 목적지를 수신 등록하는 논리 상태이다.
- `/topic/chat/1`을 구독하면 그 채널 메시지를 받는다.

아래 구조를 기억하면 된다.

```text
WebSocket Connection 1개
  ├─ SUBSCRIBE /topic/chat/1
  ├─ SUBSCRIBE /topic/chat/2
  └─ SUBSCRIBE /queue/user/123
```

즉, 연결은 보통 하나를 유지하고 구독을 동적으로 바꾼다.

---

## 5. Pub/Sub와 Broker 흐름을 STOMP로 보면

Spring STOMP 관례를 예시로 보면 흐름은 아래와 같다.

```text
Client SEND /app/chat/send
-> Controller(MessageMapping)
-> Broker destination /topic/chat/1
-> topic 구독자들에게 MESSAGE fan-out
```

핵심 포인트는 아래이다.

- 발행자(Publisher)는 수신자 목록을 직접 알 필요가 없다.
- 구독자(Subscriber)는 발행자를 직접 알 필요가 없다.
- 중간의 Broker가 구독 테이블을 기준으로 메시지를 분배한다.

---

## 6. 연결이 끊기면 왜 다시 URL이 필요한가

WebSocket 연결은 영구 연결이 아니라 "유지되는 동안만 유효한 세션"이다.

아래 상황에서 끊길 수 있다.

- 네트워크 단절
- 서버 재시작/배포
- 토큰 만료 후 서버가 close
- 프록시/로드밸런서 idle timeout
- 브라우저 탭 종료
- 앱 재시작

연결이 끊기면 상태는 아래처럼 초기화된다.

```text
TCP 종료
WebSocket close
서버 세션 종료
구독 정보 소멸(구현/설정에 따라)
```

그래서 다시 아래 단계로 돌아간다.

```text
DNS -> TCP -> (TLS) -> HTTP Upgrade -> STOMP CONNECT -> SUBSCRIBE
```

즉, URL은 "최초 1회만 평생 사용"이 아니라 "연결 세션을 다시 열 때마다" 다시 사용된다.

---

## 7. 재연결에서 가장 많이 놓치는 부분

`reconnect` 옵션을 켰다고 해서 메시지 수신이 자동 복구되는 것은 아니다.

복구 대상은 2가지이다.

- 물리 연결 복구: 소켓 재연결
- 논리 상태 복구: 구독 재등록, 인증 헤더 재주입, room context 복구

실무 패턴은 아래이다.

```ts
onConnect: () => {
  subscribeCurrentRoom();
  subscribeUserQueue();
}
```

재연결 콜백에서 "현재 필요한 구독"을 다시 등록해야 안전하다.

---

## 8. Next.js + FSD 구조에 적용하는 기준

권장 수명주기:

```text
로그인 시 1회 connect()
화면/방 전환 시 subscribe/unsubscribe 교체
로그아웃 시 disconnect()
재연결 시 onConnect에서 구독 복구
```

레이어 분리 예시는 아래가 안정적이다.

- `shared/socket`: STOMP client 싱글턴, connect/disconnect, 공통 에러 핸들러
- `features/chat`: room 구독/해제, 메시지 핸들링, 현재 room 상태
- `entities/user` 또는 인증 레이어: 토큰 갱신/재주입

피해야 할 구조는 아래이다.

```text
방 입장할 때마다 connect()
방 나갈 때마다 disconnect()
```

문제는 아래처럼 나타난다.

- Handshake 비용 반복
- 서버 세션 churn 증가
- 모바일 환경 불안정
- reconnect 루프 디버깅 난이도 상승

---

## 9. "연결은 살아있는데 메시지가 안 온다" 체크리스트

아래 순서로 보면 원인을 빨리 좁힐 수 있다.

1. `connected=true`인가
2. 현재 토픽을 실제로 `SUBSCRIBE` 했는가
3. destination 문자열이 서버 발행 경로와 정확히 일치하는가
4. 재연결 직후 재구독 로직이 실행되었는가
5. 인증 토큰이 만료되어 server-side subscribe가 거부되지 않았는가
6. 브로커 prefix(`/topic`, `/queue`)와 app prefix(`/app`)를 혼동하지 않았는가
7. 서버가 해당 topic으로 실제 publish 하고 있는가
8. 프록시 idle timeout으로 소켓이 half-open 상태가 아닌가

이 체크리스트의 핵심은 "연결 상태"와 "구독 상태"를 분리해서 점검하는 것이다.

---

## 10. 질문 문장을 기술적으로 정확하게 고쳐 쓰면

원문 의도를 살려 정확히 쓰면 아래 문장이 된다.

`초기 연결 시에는 도메인/IP/Port와 WebSocket endpoint 경로가 필요하며, 연결 성립 이후에는 HTTP 재요청이 아니라 WebSocket 프레임(STOMP 프레임 포함) 기반의 양방향 이벤트 통신을 수행한다. 연결이 끊기거나 세션이 새로 시작되면 동일 endpoint URL로 다시 연결 수립 절차를 수행해야 한다.`

---

## 레퍼런스

- RFC 6455: The WebSocket Protocol  
  https://datatracker.ietf.org/doc/html/rfc6455
- RFC 6455 Section 4 (Opening Handshake)  
  https://datatracker.ietf.org/doc/html/rfc6455#section-4
- STOMP 1.2 Specification  
  https://stomp.github.io/stomp-specification-1.2.html
- Spring WebSocket/STOMP Reference  
  https://docs.spring.io/spring-framework/reference/web/websocket/stomp.html
