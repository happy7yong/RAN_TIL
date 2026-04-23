# WebSocket 연결과 STOMP CONNECT의 차이

지금 헷갈리기 쉬운 지점은 아래 한 문장이다.

`WebSocket은 연결이라는데, 왜 그 위에서 또 STOMP CONNECT를 하느냐`

이 문서는 이 질문을 계층 관점에서 분리해서 설명한다.

---

## 1. 전체 구조

```text
TCP
 └─ WebSocket
      └─ STOMP
```

세 계층은 같은 일이 아니라 서로 다른 책임을 가진다.

- TCP: 신뢰성 있는 전송 통로를 만든다.
- WebSocket: HTTP Upgrade 이후 양방향 채널을 유지한다.
- STOMP: 그 채널 위에서 메시지 문법과 세션 규칙을 정의한다.

---

## 2. WebSocket은 무엇인가

WebSocket은 본질적으로 `양방향 통신 통로`이다.

- 클라이언트와 서버가 계속 열린 소켓을 공유한다.
- 한쪽이 원할 때 데이터를 push할 수 있다.
- 메시지의 비즈니스 의미를 강제하지는 않는다.

비유로 보면 아래가 가장 직관적이다.

- WebSocket: 전화선 연결
- STOMP: 통화 규칙(누가 먼저 말하고, 어떻게 구독하고, 어떻게 종료하는지)

즉, WebSocket 연결 성공만으로는 채팅 구독 세션이 자동으로 성립되지 않는다.

---

## 3. STOMP는 무엇인가

STOMP는 WebSocket 위에서 동작하는 메시징 프로토콜이다.

- 메시지 형태를 프레임으로 통일한다.
- 명령어(`CONNECT`, `SUBSCRIBE`, `SEND`, `DISCONNECT`)를 표준화한다.
- 헤더 기반으로 인증 정보, destination, heartbeat, 버전 협상을 전달한다.

STOMP 1.2 스펙: https://stomp.github.io/stomp-specification-1.2.html

프레임 기본 구조는 아래와 같다.

```text
COMMAND
header:value
header:value

body
\0
```

여기서 `\0`는 프레임 종료를 의미한다.

---

## 4. STOMP CONNECT는 정확히 무엇인가

핵심은 아래 문장이다.

`WebSocket 연결 성공`과 `STOMP 세션 성립`은 같은 이벤트가 아니다.

### 4-1. WebSocket 연결 단계

먼저 HTTP Upgrade가 성공한다.

```text
HTTP Upgrade -> 101 Switching Protocols
```

이 시점은 단지 소켓 통로가 열린 상태이다.

### 4-2. STOMP CONNECT 전송

그 다음 클라이언트가 STOMP `CONNECT` 프레임을 보낸다.

```text
CONNECT
accept-version:1.2
host:stg.molip.today
accessToken:Bearer ...
deviceId:uuid

\0
```

의미는 아래와 같다.

- STOMP 세션 시작 요청
- 프로토콜 버전 협상
- 인증/컨텍스트 전달

### 4-3. 서버 CONNECTED 응답

서버가 세션을 받아들이면 아래 프레임을 보낸다.

```text
CONNECTED
version:1.2

\0
```

이제부터 `SUBSCRIBE`, `SEND`, `MESSAGE`가 정상적으로 오간다.

---

## 5. 왜 STOMP CONNECT가 필요할까

WebSocket만으로는 아래 기능이 기본 제공되지 않는다.

- 애플리케이션 메시징 세션 개념
- destination 기반 구독 모델
- 메시지 레벨 인증/헤더 전달
- heartbeat 합의
- STOMP 버전 협상

STOMP CONNECT는 이 공백을 메운다.

정리하면 아래 두 줄이 정확하다.

- WebSocket: 물리적/전송 채널
- STOMP CONNECT: 논리적 메시징 세션 시작

---

## 6. 네트워크 탭에서 WS는 열렸는데 메시지가 비는 이유

`WS 연결 있음 + Messages 없음`은 보통 아래 중 하나이다.

1. 클라이언트가 STOMP `CONNECT`를 보내지 않았다.
2. 서버가 `CONNECTED`를 보내지 않았다.
3. 인증 실패로 서버가 즉시 `ERROR` 또는 `CLOSE` 처리했다.
4. CONNECTED 이후 `SUBSCRIBE`를 안 해서 수신할 채널이 없다.

즉, WebSocket은 열렸지만 STOMP 세션 단계로 진입하지 못했을 가능성이 크다.

---

## 7. 전체 흐름을 한 번에 보면

```text
1. TCP 연결
2. WebSocket handshake (101)
3. STOMP CONNECT frame 전송
4. STOMP CONNECTED frame 수신
5. SUBSCRIBE
6. MESSAGE 수신
7. 필요 시 SEND
```

디버깅할 때는 3번과 4번 사이가 가장 자주 막힌다.

---

## 8. WebSocket 성공, STOMP CONNECT 실패 시 연결은 유지될까

정답은 `서버/브로커 정책에 따라 다르다`이다.

- 어떤 서버는 인증 실패나 프로토콜 오류 시 즉시 연결을 닫는다.
- 어떤 서버는 ERROR 프레임 후 close 한다.
- 제한적으로는 잠시 열린 상태를 유지하다 타임아웃으로 닫기도 한다.

실무 관점에서는 아래처럼 가정하는 것이 안전하다.

- STOMP CONNECT 실패는 세션 실패로 간주한다.
- 클라이언트는 재인증 또는 재연결 루틴으로 복구한다.

---

## 9. 운영 체크포인트

STOMP 기반 채팅에서 안정성을 높이려면 아래를 확인한다.

1. `onConnect` 콜백이 실제 호출되는지 확인한다.
2. CONNECT 프레임에 필요한 인증 헤더가 포함되는지 확인한다.
3. CONNECTED 수신 전 `SUBSCRIBE`를 보내지 않도록 순서를 강제한다.
4. 재연결 시 이전 구독을 다시 등록하는 로직을 둔다.
5. 서버 로그에서 CONNECT/CONNECTED/ERROR/CLOSE 흐름을 같이 본다.

---

## 레퍼런스

- RFC 6455: The WebSocket Protocol  
  https://datatracker.ietf.org/doc/html/rfc6455
- STOMP 1.2 Specification  
  https://stomp.github.io/stomp-specification-1.2.html
