# STOMP-WEBSOCKET-TESTER

STOMP 프로토콜을 테스트 하는데 있어 Postman 으로 message 에서 null octet 을 처리하는게 불편하여 브라우저 상에서 간단하게 사용할 수 있는 테스터를 만듬.

---

### 클라이언트-서버 KOPI 실시간 시세 통신 흐름 (간소화)

```mermaid
sequenceDiagram
    participant Client as 클라이언트
    participant Server as 서버

    Note over Client,Server: 1. WebSocket 연결
    Client->>Server: CONNECT ws://localhost:8080/stock
    Server-->>Client: CONNECTED (세션 생성)

    Note over Client,Server: 2. 응답 채널 구독
    Client->>Server: SUBSCRIBE /user/queue/reply
    Server-->>Client: 구독 완료

    Note over Client,Server: 3. 종목 구독 요청
    Client->>Server: SEND /app/stock/subscribe<br/>{"stockCode": "005930"}
    Server-->>Client: MESSAGE /user/queue/reply<br/>{"status":"subscribed", "stockCode":"005930",<br/>"dataEndpoint":"/topic/stock/005930"}

    Note over Client,Server: 4. 실시간 데이터 채널 구독
    Client->>Server: SUBSCRIBE /topic/stock/005930
    Server-->>Client: 구독 완료

    Note over Client,Server: 5. 실시간 데이터 수신
    loop 실시간 데이터
        Server->>Client: MESSAGE /topic/stock/005930<br/>{실시간 시세 데이터}
    end

    Note over Client,Server: 6. 구독 해제
    Client->>Server: SEND /app/stock/unsubscribe<br/>{"stockCode": "005930"}
    Server-->>Client: MESSAGE /user/queue/reply<br/>{"status":"unsubscribed", "stockCode":"005930"}
    Client->>Server: UNSUBSCRIBE /topic/stock/005930
    Server-->>Client: 구독 해제 완료

    Note over Client,Server: 7. 연결 종료
    Client->>Server: DISCONNECT
    Server-->>Client: 연결 종료
```

---

## 클라이언트-서버 KOPI 실시간 시세 통신 전체 시스템 흐름 (상세)

```mermaid
sequenceDiagram
    participant Client as 클라이언트
    participant STOMP as STOMP Broker
    participant Controller as StockWebSocketController
    participant Facade as WebSocketFacade
    participant SubMgr as ClientStockSubscriptionManager
    participant KIS as KisWebSocketService
    participant KISMgr as KisWebSocketConnectionManager
    participant KISServer as KIS 서버

    Note over Client,KISServer: 1. WebSocket 연결 단계
    Client->>STOMP: CONNECT ws://localhost:8080/ws/stock
    STOMP-->>Client: CONNECTED (세션 생성)

    Note over Client,KISServer: 2. 응답 채널 구독
    Client->>STOMP: SUBSCRIBE /user/queue/reply
    STOMP-->>Client: 구독 완료 (sub-0)

    Note over Client,KISServer: 3. 종목 구독 요청
    Client->>STOMP: SEND /app/stock/subscribe<br/>{"stockCode": "005930"}
    STOMP->>Controller: @MessageMapping("/stock/subscribe")
    Controller->>Facade: subscribe(sessionId, "005930")
    Facade->>SubMgr: addSubscriber("005930", sessionId)
    SubMgr-->>Facade: 구독자 등록 완료
    Facade->>KIS: subscribe("005930")

    alt 첫 구독인 경우
        KIS->>KISMgr: ensureConnected()
        KISMgr->>KISServer: WebSocket 연결 (KIS)
        KISServer-->>KISMgr: 연결 완료
        KIS->>KISServer: 종목 구독 요청<br/>{"tr_id":"H0STCNT0","tr_key":"005930"}
        KISServer-->>KIS: 구독 승인
    else 이미 구독 중
        Note over KIS: KIS 서버에 이미 구독됨<br/>추가 요청 없음
    end

    KIS-->>Facade: 구독 완료
    Facade-->>Controller: 완료
    Controller-->>STOMP: WebSocketResponse<br/>{"status":"subscribed",<br/>"stockCode":"005930",<br/>"dataEndpoint":"/topic/stock/005930"}
    STOMP->>Client: MESSAGE /user/queue/reply<br/>(구독 응답)

    Note over Client,KISServer: 4. 실시간 데이터 구독
    Client->>STOMP: SUBSCRIBE /topic/stock/005930
    STOMP-->>Client: 구독 완료 (sub-1)

    Note over Client,KISServer: 5. 실시간 데이터 수신 (반복)
    loop 실시간 데이터 스트림
        KISServer->>KISMgr: 시세 데이터 전송<br/>(WebSocket Message)
        KISMgr->>KISMgr: handleKisMessage()
        KISMgr->>SubMgr: hasSubscribers("005930")?
        SubMgr-->>KISMgr: true
        KISMgr->>KISMgr: buildStockData()
        KISMgr->>STOMP: convertAndSend(<br/>"/topic/stock/005930", data)
        STOMP->>Client: MESSAGE /topic/stock/005930<br/>{실시간 시세 데이터}
        Client->>Client: UI 업데이트
    end

    Note over Client,KISServer: 6. 구독 해제
    Client->>STOMP: SEND /app/stock/unsubscribe<br/>{"stockCode": "005930"}
    STOMP->>Controller: @MessageMapping("/stock/unsubscribe")
    Controller->>Facade: unsubscribe(sessionId, "005930")
    Facade->>SubMgr: removeSubscriber("005930", sessionId)
    SubMgr-->>Facade: 구독자 제거 완료
    Facade->>SubMgr: hasSubscribers("005930")?

    alt 구독자가 없는 경우
        SubMgr-->>Facade: false
        Facade->>KIS: unsubscribe("005930")
        KIS->>KISServer: 종목 구독 해제 요청<br/>{"tr_id":"H0STCNT0","tr_key":"005930"}
        KISServer-->>KIS: 구독 해제 승인
    else 다른 구독자가 있는 경우
        SubMgr-->>Facade: true
        Note over Facade,KIS: KIS 구독 유지<br/>(다른 클라이언트가 사용 중)
    end

    KIS-->>Facade: 완료
    Facade-->>Controller: 완료
    Controller-->>STOMP: WebSocketResponse<br/>{"status":"unsubscribed",<br/>"stockCode":"005930"}
    STOMP->>Client: MESSAGE /user/queue/reply<br/>(구독 해제 응답)

    Client->>STOMP: UNSUBSCRIBE sub-1<br/>(/topic/stock/005930)
    STOMP-->>Client: 구독 해제 완료

    Note over Client,KISServer: 7. 연결 종료
    Client->>STOMP: DISCONNECT
    STOMP->>SubMgr: 세션의 모든 구독 정리
    STOMP-->>Client: 연결 종료
```