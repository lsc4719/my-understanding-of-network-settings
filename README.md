```
  1. [TCP Layer] 연결 수립 및 유지 (L4 - Transport)
  핵심: 데이터가 지나갈 물리적인 통로(Pipe)의 상태를 관리합니다.

    1    [Client]                                               [Server]
    2        |                                                     |
    3        |  (1) [Connect Timeout] 감시 시작                      |
    4        |  ------- SYN (연결 요청) ---------------------------> |
    5        |  <------ SYN-ACK (연결 수락) ------------------------ | (2) Listen (Port)
    6        |  ------- ACK (최종 확인) ---------------------------> |
    7        |       [ TCP Connection Established ]                |
    8        |                                                     |
    9        |  <======== (3) 데이터 패킷 (Segment) 교환 ========>    |
   10        |                                                     |
   11        |  (4) [Read / Write / Idle Timeout] 감시               | (4) [Idle Timeout] 감시
   12        |      (데이터 흐름의 정체나 중단을 감시)                  |
   13        |                                                     |
   14        |  ------- FIN (종료 요청) ---------------------------> | (5) [Graceful Shutdown]
   15        |  <------ ACK / FIN / ACK (종료 절차) --------------- |
   16        |         [ TCP Connection Closed ]                   |
   * (1) Connect Timeout (? ms): 전화를 걸고 상대방이 응답(SYN-ACK)할 때까지의 대기 시간.
   * (3) Read / Write Timeout (? ms): 데이터 패킷 조각(Segment)을 하나 보낼 때 혹은 받을 때 허용되는 최대 시간.
   * (4) Idle Timeout (? ms): 양방향 모두 데이터 교환(Read/Write)이 전혀 없을 때 연결을 정리하는 시간.

  ---

  2. [HTTP Layer] 요청 및 응답 (L7 - Application)
  핵심: 비즈니스적인 "주문-서빙" 트랜잭션이 완료되는지를 관리합니다.

    1      [Client]                                             [Server]
    2        |                                                     |
    3        |  ======= HTTP Request (주문서 전송) =======>          |
    4        |  (Request LTTB: 전송 완료 시점)                       |
    5        |                                                     |
    6        |  (1) [Response Timeout] 감시 시작                     | (2) [Server Processing]
    7        |      (백엔드가 요리/처리를 시작함...)                 | (DB 조회, 로직 수행 등)
    8        |                                                     |
    9        |  <------ HTTP Response Header (TTFB) -------------- | (3) [Response 시작]
   10        |  (1) [Response Timeout] 타이머 중단                    |
   11        |                                                     |
   12        |  <------ HTTP Response Body (LTTB) ---------------- | (4) [Response 완료]
   * (1) Response Timeout (? ms): 주문을 마친 시점(LTTB)부터 첫 번째 응답 조각(TTFB)이 올 때까지의 총 인내 시간.
   * (2) Server Processing: 실제 서버 내부에서 코드가 실행되며 시간이 소요되는 구간 (장애의 주요 원인).

  ---

  3. [TCP Connection Pool] 자원 관리 (Client-side Internal)
  핵심: 성능을 위해 미리 뚫어놓은 파이프라인(TCP)을 재사용하는 효율성을 관리합니다.

    1     [Client]                                              [Server]
    2        |                                                     |
    3        |   [ TCP Connection Pool (Fixed Size) ]              |
    4        |  +---------------------------------------+          |
    5 Req A  |  | [Socket 1: Busy (사용 중)]             | -------> | (1) [Max Connections (?)]
    6 Req B  |  | [Socket 2: Busy (사용 중)]             | -------> | (동시 통화 가능 개수 제한)
    7        |  | [Socket 3: Idle (Pool 보관 중)]        |          |
    8        |  | [Socket 4: Idle (Pool 보관 중)]        |          | (2) [Max Idle Time (? ms)]
    9        |  +---------------------------------------+          | (미사용 소켓 자동 폐기)
   10        |                                                     |
   11 Req C  |  (3) [Acquire Timeout] 감시 시작                      |
   12        |      (빈 소켓이 생길 때까지 대기...)                  |
   * (1) Max Connections (?): 백엔드 서버당 동시에 맺을 수 있는 최대 TCP 연결 수.
   * (2) Max Idle Time (? ms): 사용이 끝난 소켓이 풀에서 재사용을 기다리며 생존할 수 있는 시간.
   * (3) Acquire Timeout (? ms): 풀이 가득 찼을 때, 다른 요청이 끝나서 소켓을 넘겨받기 위해 기다리는 시간.

``
