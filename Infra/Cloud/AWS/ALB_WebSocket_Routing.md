## 개념 정의
ALB(Application Load Balancer)는 AWS의 L7 로드밸런서이며, 도메인으로 들어온 요청을 규칙 기반으로 분배하는 진입 계층이다.  
Nginx는 ALB 뒤에서 동작하는 리버스 프록시이며, 요청 경로와 헤더를 실제 애플리케이션 서버(Next.js, Chat Backend)로 전달하는 중간 계층이다.  
WebSocket pending 문제는 대개 STOMP 코드 문제가 아니라 `ALB 경로 라우팅` 또는 `Nginx Upgrade 헤더 전달` 중 하나가 깨진 상태에서 발생한다.  
핵심은 WebSocket이 end-to-end로 Upgrade되어야 하며, 중간 프록시 어느 한 단계라도 일반 HTTP처럼 처리하면 세션이 열리지 않는다는 점이다.

## 동작 원리
1. 브라우저가 `wss://stg.molip.today/ws`로 연결을 시작하면 요청은 먼저 ALB(HTTPS Listener)로 진입한다.  
2. ALB는 Listener Rule 우선순위를 검사해 `/ws` 요청의 목적지 Target Group을 결정한다.  
3. ALB가 Nginx 타겟으로 전달하면, Nginx는 `/ws` 요청에서 `Upgrade`와 `Connection` 헤더를 유지한 채 chat backend로 프록시해야 한다.  
4. chat backend(Spring STOMP)가 `/ws` endpoint에서 handshake를 처리하고 `101 Switching Protocols`를 반환하면 WebSocket 세션이 성립한다.  
5. 이후에는 같은 연결 위에서 STOMP `CONNECT -> CONNECTED -> SUBSCRIBE -> MESSAGE` 흐름이 진행된다.

실패 패턴은 보통 두 가지이다.

- ALB 규칙에 `/ws`가 없어서 default route로 Next.js에 도달하는 패턴이다.
- Nginx에 `Upgrade` 설정이 누락되어 backend까지 handshake가 전달되지 않는 패턴이다.

여기서 깨달은 점은 WebSocket pending은 인증/쿠키 이슈처럼 보여도 실제 원인은 라우팅 체인의 끊김인 경우가 더 많다는 사실이다.

현재 구조를 그림으로 쓰면 아래와 같다.

```text
Internet
  ↓
ALB (HTTPS 종료)
  ↓
Nginx (리버스 프록시)
  ├─ Next.js
  └─ Chat Backend (Spring STOMP)
```

정상 경로와 오류 경로는 아래처럼 구분된다.

```text
정상: ALB /ws -> Nginx /ws -> Chat Backend /ws -> 101
오류: ALB(or Nginx) /ws -> Next.js -> WebSocket pending/실패
```

## 코드
```hcl
# ALB Listener Rule: /ws는 chat 경로로 우선 라우팅
resource "aws_lb_listener_rule" "ws_route" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nginx.arn
  }

  condition {
    path_pattern {
      values = ["/ws", "/ws/*"]
    }
  }
}
```

```nginx
# Nginx: /ws 요청을 chat backend로 전달할 때 Upgrade 헤더를 유지
location /ws {
    proxy_pass http://chat_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 600s;
}

# 일반 웹 요청은 Next.js
location / {
    proxy_pass http://nextjs_app;
    proxy_set_header Host $host;
}
```

위 Terraform은 ALB 레이어에서 `/ws`를 일반 웹 라우팅보다 우선 매칭시키는 IaC 지점이다.  
위 Nginx 설정은 `Upgrade` 헤더를 chat backend까지 유지하는 프록시 지점이며, 이 설정이 빠지면 handshake가 중간에서 깨진다.  
운영 검증은 브라우저 Network에서 `/ws` 응답이 `101 Switching Protocols`인지, 그리고 최종 업스트림이 chat backend인지 확인하면 된다.

## 언제 쓰는지
- ALB 뒤에 Nginx를 두고 하나의 도메인에서 Next.js와 실시간 채팅 서버를 함께 운영할 때 사용한다.  
- `/ws` 호출 시 HTML 페이지가 반환되거나 WebSocket이 pending으로 멈출 때 원인을 라우팅 계층으로 좁혀야 할 때 사용한다.  
- 인프라팀과 백엔드팀 간 점검 항목을 빠르게 합의해야 할 때 기준 문서로 사용한다.

## 점검 질문 템플릿
- `/ws` 경로가 ALB와 Nginx를 거쳐 채팅 서버 target으로 라우팅되고 있는지 확인 가능한가.  
- Nginx에 `proxy_http_version 1.1`, `Upgrade`, `Connection: upgrade` 설정이 실제 반영되어 있는가.  
- 채팅 백엔드가 Spring Boot라면 `registry.addEndpoint("/ws")` endpoint가 현재 환경에 동일 경로로 열려 있는가.  
- 현재 Network 탭에서 `/ws`가 `101 Switching Protocols`인지, 아니면 Next.js HTML 응답으로 떨어지는지 확인 가능한가.

## 핵심 한 줄
WebSocket pending의 본질은 ALB 또는 Nginx에서 `/ws` Upgrade 경로가 끊긴 상태이며, `/ws`를 chat backend까지 end-to-end로 전달해야 세션이 열린다.

---

참고 문서  
- AWS ALB 소개: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html  
- ALB WebSocket 지원: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#listener-websockets
