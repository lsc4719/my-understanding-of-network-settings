- Timeout layering
  - timeout 종류
    - idle cleanup timeout: 유휴 상태의 connection을 정리하기 위한 timeout
    - in-flight inactivity timeout: 요청/응답 처리 중 read/write 등의 진전이 일정 시간 없을 때 발생하는 timeout
    - deadline timeout: 요청/응답/연결 단계 전체에 대한 최대 허용 시간HTTP
  - 서로 다른 level 간의 timeout 대소관계
    - HTTP keep-alive timeout <= socket idle timeout <= TCP keepalive detection time
    - HTTP request/response deadline <= socket read/write timeout <= OS/TCP backstop
    - HTTP-specific inactivity timeout <= generic socket inactivity timeout
  - client - proxy - server ordering
    - 같은 timeout이라도 hop마다 의미가 다를 수 있으므로, 모든 timeout에 대해 고정된 대소관계를 두지는 않는다.
    - idle / keep-alive 계열은 보통 바깥 hop이 안쪽 hop보다 더 오래 connection 재사용을 기대하지 않도록 설정한다.
      - client connection pool idle timeout <= proxy downstream keep-alive timeout
      - proxy upstream idle/keep-alive timeout <= server keep-alive timeout
    - overall deadline 계열은 보통 바깥의 전체 timeout이 가장 크고, 안쪽 hop의 세부 timeout은 그보다 짧게 둔다.
      - proxy per-try timeout <= proxy overall timeout <= client overall timeout
    - 반면 read/write timeout은 hop-local inactivity timeout 성격이 강하므로, 같은 이름으로 client < proxy < server 같은 고정 규칙을 두기보다는 각 hop의 역할에 따라 따로 조정한다.

- OS level

- Network Library Socket Level
  - connection timeout (client)
    - client가 connect() 완료를 기다리는 최대 시간
  - read/write timeout (client/server)
    - 연결 수립 후 socket read/write가 일정 시간 동안 진행되지 않으면 timeout 발생
  - idle timeout (client/server)
    - 연결이 열린 상태에서 일정 시간 동안 송수신이 없으면 연결을 종료하는 timeout
  - graceful shutdown (server)
    - 새로운 connection accept를 중단
    - 기존 connection 또는 진행 중인 요청이 종료될 시간을 준 뒤 application 종료
    - 필요 시 shutdown timeout 이후 남은 connection을 강제 종료할 수 있음
  - tcp keep-alive timeout (client/server)
    - 연결이 오랫동안 idle 상태일 때 peer가 살아있는지 확인하기 위해 probe를 전송
    - 일정 횟수 또는 일정 시간 동안 응답이 없으면 dead connection으로 판단하고 연결 종료
  - connection pool idle timeout (client)
    - pool 안의 idle connection을 일정 시간 후 제거 또는 종료
    - 일반적으로 outbound connection 재사용을 위한 client-side pool에서 사용
  - connection max lifetime / max age (client/server)
    - connection이 idle 여부와 무관하게 일정 시간이 지나면 더 이상 재사용되지 않음

- Network Library HTTP Level
  - request timeout (client)
    - HTTP 요청 1건 전체에 허용되는 최대 시간
  - response timeout (client)
    - HTTP 요청을 보낸 뒤 응답을 기다리는 최대 시간
    - 구현체에 따라 응답 시작까지의 시간일 수도 있고, 전체 응답 처리 완료까지 포함할 수도 있음
    - 구현체에 따라 부분 응답 사이의 최대 idle/read 간격으로 동작할 수도 있음
    - Reactor Netty에서는 응답 수신 중 각 network-level read operation 사이의 최대 허용 간격을 의미함
  - HTTP idle timeout (client/server)
    - HTTP connection에서 일정 시간 동안 HTTP-level activity가 없으면 connection 종료
  - HTTP keep-alive timeout (client/server)
    - 재사용 가능한 HTTP connection을 idle 상태로 유지할 수 있는 최대 시간
    - 일정 시간 동안 다음 요청이 없으면 underlying connection 종료
    - HTTP/1.1에서는 HTTP idle timeout과 거의 같은 의미로 쓰이는 경우가 많음
    - HTTP/2에서는 구현체에 따라 active stream, frame activity, ping 정책 등을 기준으로 판단할 수 있음
  - request header timeout (server)
    - request header를 수신하는 데 허용되는 최대 시간
  - request body timeout / upload timeout (client/server)
    - request body 송수신이 일정 시간 동안 완료되지 않거나 진전되지 않으면 timeout 발생
  - response header timeout (client)
    - response header를 수신하는 데 허용되는 최대 시간
  - stream idle timeout (client/server, HTTP/2)
    - 개별 stream이 일정 시간 activity 없이 유지되면 종료
    - ping / heartbeat timeout (client/server)
    - HTTP/2, gRPC 등에서 ping 또는 heartbeat 응답이 일정 시간 없으면 연결 이상으로 판단
  - max keep-alive requests (client/server)
    - 하나의 persistent connection에서 처리할 수 있는 최대 요청 수
  - graceful shutdown timeout (server)
    - 새로운 HTTP 요청 수락을 중단
    - 기존 요청 또는 stream이 종료될 시간을 준 뒤 application 종료
    - 필요 시 timeout 이후 강제 종료 가능

### Spring Cloud Gateway / Reactor Netty

| 설정(property / hook / code)                                            | 레벨                                  | 매치되는 개념                                                    | 설명                                                                                                                                              |
| --------------------------------------------------------------------- | ----------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `spring.cloud.gateway.server.webflux.httpclient.connect-timeout`      | outbound HTTP client                | connection timeout (client)                                | SCG가 downstream 서버에 `connect()` 완료를 기다리는 최대 시간. 전역 설정 가능. ([Home][1])                                                                           |
| `route.metadata.connect-timeout`                                      | outbound HTTP client (route별)       | connection timeout (client)                                | 특정 route에만 적용하는 connect timeout. route metadata로 override 가능. ([Home][1])                                                                       |
| `spring.cloud.gateway.server.webflux.httpclient.response-timeout`     | outbound HTTP client                | response timeout / in-flight inactivity timeout            | downstream 응답을 기다리거나 응답 수신 중 일정 시간 진전이 없을 때 적용하는 timeout으로 쓰는 항목. 구현체 관점에서는 “전체 요청 deadline”보다는 response 대기/수신 timeout으로 보는 게 맞습니다. ([Home][1]) |
| `route.metadata.response-timeout`                                     | outbound HTTP client (route별)       | response timeout / in-flight inactivity timeout            | 특정 route에만 적용하는 response timeout. ([Home][1])                                                                                                   |
| `spring.cloud.gateway.server.webflux.httpclient.pool.acquire-timeout` | outbound connection pool            | pool acquire timeout                                       | connection pool에서 재사용 가능한 connection을 빌릴 때 기다리는 최대 시간. 질문에서 정리한 timeout taxonomy 중에는 직접 1:1 이름은 없지만, 운영상 pool 관련 대기 timeout으로 봅니다. ([Home][1])  |
| `spring.cloud.gateway.server.webflux.httpclient.pool.max-idle-time`   | outbound connection pool            | connection pool idle timeout / idle cleanup timeout        | pool 안의 idle connection을 얼마나 오래 유지할지 정하는 값. idle connection cleanup에 해당합니다. ([Home][1])                                                         |
| `spring.cloud.gateway.server.webflux.httpclient.pool.max-life-time`   | outbound connection pool            | connection max lifetime / max age                          | connection이 idle 여부와 무관하게 얼마나 오래 재사용될 수 있는지 정하는 상한. ([Home][1])                                                                                 |
| `spring.cloud.gateway.server.webflux.httpclient.pool.max-connections` | outbound connection pool            | pool capacity 관련 설정                                        | timeout 자체는 아니지만, pool 고갈 시 `acquire-timeout` 동작과 함께 봐야 하는 설정입니다. ([Home][1])                                                                   |
| `spring.cloud.gateway.server.webflux.httpclient.pool.type`            | outbound connection pool            | pool 동작 방식                                                 | `fixed` 등 pool 전략 설정. timeout 자체는 아니지만 idle/max-life 설정과 함께 동작합니다. ([Home][1])                                                                  |
| `server.netty.connection-timeout`                                     | inbound server                      | connection timeout (server-side accept/channel setup)      | Netty 서버 측 connection timeout. inbound 연결 수립 단계의 timeout 성격입니다. ([Home][2])                                                                     |
| `server.netty.idle-timeout`                                           | inbound server                      | idle timeout / HTTP idle timeout / keep-alive idle timeout | 서버 쪽 channel이 일정 시간 idle이면 정리하는 설정. HTTP/1.1 keep-alive idle timeout에 가깝게 운영하는 경우가 많습니다. ([Home][2])                                            |
| `server.netty.max-keep-alive-requests`                                | inbound server                      | max keep-alive requests                                    | 하나의 keep-alive connection에서 처리할 수 있는 최대 요청 수. timeout은 아니지만 keep-alive 재사용 정책에 직접 대응합니다. ([Home][2])                                            |
| `server.max-http-request-header-size`                                 | inbound HTTP server                 | request header size limit                                  | request header timeout은 아니고, header **크기 제한**에 대응하는 설정입니다. 시간 제한과는 별개입니다. ([Home][2])                                                           |
| `server.shutdown=graceful`                                            | application/server shutdown         | graceful shutdown                                          | 새 요청 수락을 중단하고 기존 요청이 끝날 시간을 주는 graceful shutdown 동작에 대응합니다. ([Home][3])                                                                         |
| `spring.lifecycle.timeout-per-shutdown-phase`                         | application/server shutdown         | graceful shutdown timeout                                  | graceful shutdown 시 각 phase에서 기다릴 최대 시간. 질문의 graceful shutdown timeout에 직접 매치됩니다. ([Home][3])                                                   |
| `HttpClientCustomizer`                                                | outbound HTTP client                | property로 안 되는 세부 timeout 커스터마이징 지점                        | SCG property만으로 부족한 read/write inactivity timeout, TCP keepalive 옵션 등을 넣을 때 사용하는 확장 포인트입니다. ([Home][1])                                         |
| `ChannelOption.CONNECT_TIMEOUT_MILLIS`                                | outbound HTTP client / Netty socket | connection timeout (client)                                | `httpclient.connect-timeout`과 같은 개념을 Netty 레벨에서 직접 제어할 때 쓰는 옵션입니다. ([Home][4])                                                                  |
| `ReadTimeoutHandler`                                                  | Netty channel handler               | read timeout / in-flight inactivity timeout                | 연결 수립 후 read가 일정 시간 진행되지 않으면 timeout 처리하는 방식. SCG 공용 property보다는 Netty handler 커스터마이징 영역입니다. ([Home][4])                                        |
| `WriteTimeoutHandler`                                                 | Netty channel handler               | write timeout / in-flight inactivity timeout               | 연결 수립 후 write가 일정 시간 진행되지 않으면 timeout 처리하는 방식. 역시 Netty handler로 다루는 항목입니다. ([Home][4])                                                         |
| `SO_KEEPALIVE` / `TCP_KEEPIDLE` / `TCP_KEEPINTVL` / `TCP_KEEPCNT`     | TCP socket option                   | TCP keep-alive timeout / detection time                    | 질문의 “tcp keep-alive timeout” 계열에 대응하는 건 보통 이쪽입니다. SCG 일반 property라기보다 Reactor Netty/Netty socket 옵션 커스터마이징으로 넣습니다. ([Home][5])                  |
| `onReadIdle(...)` / `onWriteIdle(...)` 류 connection hook              | Netty/Reactor connection hook       | generic socket inactivity timeout                          | read/write 각각의 idle 감지를 hook으로 다루는 방식입니다. property가 아니라 코드 레벨 설정 포인트입니다. ([Home][5])                                                            |

[1]: https://docs.spring.io/spring-cloud-gateway/reference/configprops.html?utm_source=chatgpt.com "Configuration Properties :: Spring Cloud Gateway"
[2]: https://docs.spring.io/spring-boot/appendix/application-properties/index.html?utm_source=chatgpt.com "Common Application Properties :: Spring Boot"
[3]: https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html?utm_source=chatgpt.com "Graceful Shutdown :: Spring Boot"
[4]: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux.html?utm_source=chatgpt.com "Spring Cloud Gateway Server WebFlux :: Spring Cloud Gateway"
[5]: https://docs.spring.io/projectreactor/reactor-netty/docs/current/reference/html/?utm_source=chatgpt.com "Reactor Netty Reference Guide"


### SCG에서 매핑이 애매하거나 별도 구현이 필요한 항목

| 개념                                       | SCG/Reactor Netty에서 보통 어떻게 잡는지                                                  | 설명                                                                                                    |
| ---------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| request timeout (HTTP 요청 1건 전체 deadline) | `response-timeout` + filter/circuit breaker/time limiter + app-level timeout 조합 | SCG 기본 property 하나가 “요청 1건 전체 deadline”을 완전히 대표하지는 않습니다. 보통 여러 계층 조합으로 구현합니다. ([Home][1])             |
| request header timeout                   | 별도 Netty/HTTP server 커스터마이징 필요                                                  | 공용 property로는 보통 header **크기 제한**은 있어도 “header를 몇 초 안에 받아야 한다”는 식의 대표 property는 보이지 않습니다. ([Home][2]) |
| request body timeout / upload timeout    | Netty handler 또는 application-level timeout                                      | body 업로드/다운로드 중 inactivity를 별도 제어하려면 코드 커스터마이징이 필요한 경우가 많습니다. ([Home][3])                             |
| response header timeout                  | 별도 구현 또는 `response-timeout`으로 운영                                                | “response header만의 timeout”으로 딱 떨어지는 대표 property보다는 `response-timeout`으로 운영하는 경우가 많습니다. ([Home][1])   |
| stream idle timeout (HTTP/2)             | HTTP/2 세부 커스터마이징                                                                | SCG 공용 property로 널리 노출된 대표 항목은 제한적입니다. ([Home][3])                                                    |
| ping / heartbeat timeout                 | HTTP/2 / TCP 레벨 커스터마이징                                                          | gRPC/HTTP2 heartbeat류는 일반 SCG property보다 하위 레벨 설정 성격이 강합니다. ([Home][4])                               |

[1]: https://docs.spring.io/spring-cloud-gateway/reference/configprops.html?utm_source=chatgpt.com "Configuration Properties :: Spring Cloud Gateway"
[2]: https://docs.spring.io/spring-boot/appendix/application-properties/index.html?utm_source=chatgpt.com "Common Application Properties :: Spring Boot"
[3]: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux.html?utm_source=chatgpt.com "Spring Cloud Gateway Server WebFlux :: Spring Cloud Gateway"
[4]: https://docs.spring.io/projectreactor/reactor-netty/docs/current/reference/html/?utm_source=chatgpt.com "Reactor Netty Reference Guide"

### ALB

| 설정(property)                   | 레벨                              | 매치되는 개념                                                    | 설명                                                                                                                                                           |
| ------------------------------ | ------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `idle_timeout.timeout_seconds` | ALB connection                  | idle timeout / HTTP idle timeout / keep-alive idle timeout | 기존 client connection 또는 target connection에 데이터 송수신이 없는 상태를 얼마나 허용할지 정하는 timeout입니다. ALB에서 timeout 개념으로 가장 대표적인 항목입니다. 기본 60초이며 범위는 1~4000초입니다. ([AWS 문서][1]) |
| `client_keep_alive.seconds`    | ALB client-side HTTP connection | HTTP keep-alive timeout / client keep-alive duration       | ALB가 클라이언트와의 persistent HTTP connection을 얼마나 오래 유지할지 정하는 설정입니다. ALB 쪽에서 별도로 노출되는 또 하나의 핵심 timeout/유사-timeout 항목입니다. ([AWS 문서][1])                            |

[1]: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html?utm_source=chatgpt.com "Edit attributes for your Application Load Balancer"

### Tomcat
| 설정(property / connector attribute / hook / code)                           | 레벨                                   | 매치되는 개념                                                     | 설명                                                                                                                                                                                                                                      |
| -------------------------------------------------------------------------- | ------------------------------------ | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `server.tomcat.connection-timeout`                                         | inbound HTTP server                  | connection timeout / request line timeout                   | Spring Boot 내장 Tomcat에서 connector가 connection 수락 후 request URI line이 들어오기를 기다리는 시간. Boot 문서 설명도 이 의미로 되어 있습니다. ([Home][1])                                                                                                              |
| `Connector.connectionTimeout`                                              | Tomcat HTTP Connector                | connection timeout / request line timeout                   | Tomcat connector 자체 설정. connection을 accept한 뒤 request URI line이 제시되기를 기다리는 시간. `disableUploadTimeout=false`가 아니면 request body read에도 이 값이 사용될 수 있습니다. ([Tomcat][2])                                                                     |
| `socket.soTimeout`                                                         | Java TCP socket / Tomcat socket attr | socket read timeout / connection timeout 대응                 | Tomcat 문서상 `socket.soTimeout`은 `connectionTimeout`과 equivalent라고 명시됩니다. 즉 Tomcat connector에서는 socket read timeout이 사실상 `connectionTimeout` 계열과 묶여 동작합니다. ([Tomcat][2])                                                                  |
| `server.tomcat.keep-alive-timeout`                                         | inbound HTTP server                  | HTTP keep-alive timeout / HTTP idle timeout                 | 다음 HTTP 요청을 얼마나 기다릴지 정하는 시간. 미설정 시 `connection-timeout` 값을 사용하고, `-1`이면 무제한입니다. ([Home][1])                                                                                                                                             |
| `Connector.keepAliveTimeout`                                               | Tomcat HTTP Connector                | HTTP keep-alive timeout / HTTP idle timeout                 | keep-alive connection에서 다음 요청이 오기를 기다리는 시간. 기본값은 `connectionTimeout` 값을 따릅니다. ([Tomcat][2])                                                                                                                                             |
| `server.tomcat.max-keep-alive-requests`                                    | inbound HTTP server                  | max keep-alive requests                                     | 하나의 keep-alive connection에서 처리 가능한 최대 요청 수. `0` 또는 `1`이면 keep-alive/pipelining 비활성화, `-1`이면 무제한입니다. ([Home][1])                                                                                                                         |
| `Connector.maxKeepAliveRequests`                                           | Tomcat HTTP Connector                | max keep-alive requests                                     | Tomcat connector 레벨의 최대 keep-alive/pipelined request 수 설정. 기본값은 100입니다. ([Tomcat][2])                                                                                                                                                   |
| `Connector.asyncTimeout`                                                   | Tomcat HTTP Connector                | deadline timeout (async request) / async processing timeout | Servlet async request의 기본 timeout. 질문의 taxonomy로 보면 일반 sync request timeout보다는 **async 요청 처리 전체의 기본 deadline** 쪽에 가깝습니다. ([Tomcat][2])                                                                                                  |
| `Connector.disableUploadTimeout`                                           | Tomcat HTTP Connector                | request body timeout / upload timeout 제어 플래그                | 업로드 중에는 별도의 더 긴 timeout을 쓸지 결정하는 플래그. 기본은 `true`여서 긴 upload timeout 사용이 비활성화됩니다. ([Tomcat][2])                                                                                                                                          |
| `Connector.connectionUploadTimeout`                                        | Tomcat HTTP Connector                | request body timeout / upload timeout                       | 업로드 진행 중 적용할 timeout. `disableUploadTimeout=false`일 때만 유효합니다. 대용량 request body 업로드 시간 제어에 직접 대응합니다. ([Tomcat][2])                                                                                                                       |
| `server.max-http-request-header-size`                                      | inbound HTTP server                  | request header size limit                                   | request **header timeout** 은 아니고 **header 크기 제한** 입니다. Tomcat에서는 request line + headers 전체 크기 기준으로 적용된다고 Boot 문서가 설명합니다. ([Home][1])                                                                                                    |
| `server.tomcat.max-http-response-header-size`                              | inbound HTTP server                  | response header size limit                                  | timeout은 아니고 response header 크기 제한입니다. 응답 헤더가 과도하게 커지는 것을 막는 용도입니다. ([Home][1])                                                                                                                                                         |
| `Connector.maxHttpResponseHeaderSize`                                      | Tomcat HTTP Connector                | response header size limit                                  | Tomcat connector 레벨에서 response line + headers 최대 크기를 제한합니다. timeout은 아닙니다. ([Tomcat][2])                                                                                                                                                |
| `server.tomcat.max-connections`                                            | inbound HTTP server                  | connection capacity 관련 설정                                   | timeout 자체는 아니지만 연결 수 상한입니다. 상한 도달 후 OS backlog/accept queue 동작과 함께 봐야 합니다. ([Home][1])                                                                                                                                                 |
| `Connector.maxConnections`                                                 | Tomcat HTTP Connector                | connection capacity 관련 설정                                   | 동시에 accept/process 할 수 있는 connection 수 상한. 질문의 timeout 분류와 1:1은 아니지만 idle/queue/accept 동작과 같이 운영해야 하는 핵심 설정입니다. ([Tomcat][2])                                                                                                           |
| `Connector.acceptCount`                                                    | Tomcat HTTP Connector                | backlog / accept queue 관련 설정                                | timeout은 아니지만 `maxConnections` 도달 후 OS가 queue에 쌓아둘 수 있는 incoming connection 대기 길이입니다. queue가 차면 연결 거절 또는 client timeout으로 이어질 수 있습니다. ([Tomcat][2])                                                                                     |
| `server.shutdown=graceful`                                                 | application/server shutdown          | graceful shutdown                                           | Spring Boot에서 내장 Tomcat graceful shutdown 활성화. 새 요청 수락을 멈추고 기존 요청이 끝날 시간을 줍니다. ([Home][1])                                                                                                                                              |
| `spring.lifecycle.timeout-per-shutdown-phase`                              | application/server shutdown          | graceful shutdown timeout                                   | graceful shutdown 각 phase의 최대 대기 시간. 질문의 graceful shutdown timeout에 직접 대응합니다. ([Home][1])                                                                                                                                               |
| `Connector.executorTerminationTimeoutMillis`                               | Tomcat Connector shutdown            | connector shutdown timeout                                  | connector 내부 executor의 worker thread 종료를 기다리는 시간. Tomcat connector stop 과정의 timeout입니다. ([Tomcat][2])                                                                                                                                   |
| `server.tomcat.max-swallow-size`                                           | inbound HTTP server                  | aborted upload 처리 한도                                        | timeout은 아니지만 업로드 중단/무시 상황에서 Tomcat이 삼켜(swallow) 줄 request body 최대 크기입니다. upload 관련 운영에서 함께 봐야 합니다. ([Home][1])                                                                                                                         |
| `Connector.connectionLinger` / `socket.soLingerOn` / `socket.soLingerTime` | TCP socket close                     | graceful close 보조 설정 / SO_LINGER                            | timeout taxonomy의 핵심 HTTP timeout은 아니지만 socket close linger 동작에 대응합니다. 연결 종료 시점의 소켓 레벨 동작을 바꿉니다. ([Tomcat][2])                                                                                                                          |
| `TomcatConnectorCustomizer`                                                | Boot customization hook              | property로 안 되는 세부 timeout/connector 설정 주입 포인트               | Boot property로 없는 connector attribute를 programmatic 하게 넣고 싶을 때 쓰는 확장 포인트입니다. 예: `connectionUploadTimeout`, `disableUploadTimeout`, `asyncTimeout` 등 세부 connector 값 주입. Boot property 문서와 Tomcat connector 속성 문서를 같이 봐야 합니다. ([Home][1]) |

[1]: https://docs.spring.io/spring-boot/appendix/application-properties/index.html "Common Application Properties :: Spring Boot"
[2]: https://tomcat.apache.org/tomcat-9.0-doc/config/http.html "Apache Tomcat 9 Configuration Reference (9.0.116) - The HTTP Connector"

### Tomcat에서 매핑이 애매하거나 별도 구현/운영 해석이 필요한 항목
| 개념                                       | Tomcat에서 보통 어떻게 잡는지                                                        | 설명                                                                                                                                                               |
| ---------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| request timeout (HTTP 요청 1건 전체 deadline) | 별도 application timeout / async timeout / filter / servlet async timeout 조합 | Tomcat connector에는 “동기 HTTP 요청 1건 전체에 대한 명시적 overall deadline” 하나가 딱 있는 구조는 아닙니다. 보통 request line, keep-alive, upload, async timeout 등으로 나뉘어 있습니다. ([Tomcat][1]) |
| read timeout                             | 주로 `connectionTimeout` 으로 해석                                               | Tomcat 문서상 `socket.soTimeout`이 `connectionTimeout`과 equivalent이므로, 일반적인 socket read timeout을 별도 이름으로 두기보다 `connectionTimeout` 계열로 운영하는 편입니다. ([Tomcat][1])       |
| write timeout                            | 별도 대표 connector property 없음                                                | Tomcat HTTP connector 표준 속성에서 Netty의 `WriteTimeoutHandler` 같은 식의 대표 write-timeout 속성은 보이지 않습니다. 보통 app/server thread 처리와 OS/TCP 동작으로 흡수됩니다. ([Tomcat][1])        |
| HTTP idle timeout                        | 주로 `keepAliveTimeout`                                                      | keep-alive 상태에서 다음 요청을 기다리는 시간이므로 HTTP idle timeout 개념에 가장 직접 대응합니다. ([Tomcat][1])                                                                               |
| request header timeout                   | 대표 property 없음                                                             | header **크기 제한** 은 `server.max-http-request-header-size` 등으로 가능하지만, “header를 몇 초 내로 받아야 함” 같은 대표 설정은 별도로 보이지 않습니다. ([Home][2])                                   |
| response header timeout                  | 대표 property 없음                                                             | response header **크기 제한** 은 있어도, response header 작성/전송 시간 timeout으로 바로 매핑되는 대표 설정은 보이지 않습니다. ([Home][2])                                                         |
| TCP keepalive detection time             | OS/JVM socket 옵션 영역                                                        | Tomcat HTTP connector 표준 문서에서 질문하신 TCP keepalive detection time을 직접 조절하는 대표 connector property보다는 OS/JVM socket 설정 쪽으로 보는 편이 맞습니다. ([Tomcat][1])                 |

[1]: https://tomcat.apache.org/tomcat-9.0-doc/config/http.html "Apache Tomcat 9 Configuration Reference (9.0.116) - The HTTP Connector"
[2]: https://docs.spring.io/spring-boot/appendix/application-properties/index.html "Common Application Properties :: Spring Boot"

