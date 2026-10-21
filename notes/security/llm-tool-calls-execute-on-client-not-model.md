---
status: 활성
verified: 2026-10-21
---
# LLM 도구 호출은 모델이 아니라 클라이언트가 실행한다 — 확정적으로 막을 수 있는 곳은 클라이언트다

> 모델(원격 API)은 「이 도구를 이 인자로 불러 달라」는 `tool_use` 블록을 만들어 돌려주고 멈출 뿐이다.
> 실제 실행은 사용자 기기의 에이전트 클라이언트(예: Claude Code)가 하고, 로컬 MCP 서버도 그 클라이언트의 자식 프로세스로 돈다.
> 그래서 도구 호출을 **확정적으로** 막는 지점은 클라이언트(도구 이름 단위 deny 규칙, 실행 전 훅)다.
> 모델 쪽에서 막는 것은 「제안하지 않게」 하는 확률적 차단이다. 예외는 API 가 직접 실행하는 server tool 과 API 의 MCP connector 다.

## 왜 그렇게 동작하나

한 번의 도구 호출은 다음 왕복이다.

1. 클라이언트가 사용자 메시지와 `tools` 목록을 API 로 보낸다.
2. 모델이 메시지를 해석해 `tool_use` 블록(도구 이름 + JSON 인자)을 만들고, `stop_reason: "tool_use"` 로 응답을 끝낸다.
3. 클라이언트가 그 블록을 받아 MCP `tools/call` JSON-RPC 요청으로 감싸 MCP 서버에 보낸다.
4. MCP 서버가 요청을 실제 동작(외부 REST API 호출, 로컬 앱 조작 등)으로 바꾸고, 결과를 `content` 배열로 돌려준다.
5. 클라이언트가 그 결과를 `tool_result` 로 다음 API 요청에 실어 보낸다. 모델은 그것을 읽고 다음 행동을 정한다.

- `[문서]` client tool 은 「사용자 코드가 실행하는」 도구다. API 는 `tool_use` 를 돌려주고, 사용자 코드가 실행한 뒤 `tool_result` 를 보내야 대화가 이어진다 (Claude API tool use 문서).
- `[문서]` stdio transport 에서는 클라이언트가 MCP 서버를 subprocess 로 띄우고, JSON-RPC 메시지를 stdin/stdout 으로 주고받는다 (MCP spec 2025-11-25, Transports).
- `[문서]` 도구 실행 요청은 `tools/call`(인자 `name`, `arguments`)이고, 응답은 `content` 배열과 `isError` 다 (MCP spec 2025-11-25, Tools).
- `[실측]` 로컬 MCP 서버들은 Claude Code 프로세스의 자식으로 떠 있다. 원격 커넥터로 붙은 문서 도구는 자식 프로세스 목록 전체에 없다 (아래 확인 방법).
- `[문서]` server tool(예: `web_fetch`)과 API 의 MCP connector(`mcp_tool_use`)는 API 쪽이 실행한다. 이 경로에서는 3~4번이 사용자 기기가 아니라 API 쪽에서 일어난다.
- `[추정]` Claude Code 에 붙은 원격 커넥터는 위 API MCP connector 와 다른 경로다. 로컬 프로세스만 없을 뿐, 클라이언트가 HTTP 로 원격 MCP 서버에 `tools/call` 을 보낼 가능성이 크다. 그렇다면 실행 주체는 여전히 클라이언트다 (호출 중 `lsof` 로 연결 주체를 확인하지 않음).

그래서 막는 지점이 둘로 갈린다.

- **클라이언트(3번 직전)**: 권한 deny 규칙과 실행 전 훅이 여기서 동작한다. 도구 이름 단위 deny 와 훅은 모델이 무엇을 제안하든 실행 자체를 거부한다. 결정적이다.
  - ⚠️ 인자 패턴 deny(`Bash(<접두사>…)` 류)는 예외다. 같은 동작을 다른 문자열로 우회할 수 있다. → [[prefix-allowlist-is-not-a-capability-boundary]]
- **모델 쪽(2번)**: 시스템 프롬프트나 모델 판단으로 「제안하지 않게」 하는 것이다. 입력 속 글(prompt injection)에 흔들릴 수 있으므로 경계로 믿지 않는다.
- `[추정]` `tools` 목록에서 도구를 빼면 모델이 그 도구를 제안하지 않는다. 다만 목록을 정하는 것도 클라이언트이므로, 이 역시 클라이언트 쪽 통제다 (목록에 없는 이름으로 `tool_use` 가 나오는지는 시험하지 않음).

메시지 흐름으로 보면, 도구 **요청**의 종단은 MCP 서버(더 정확히는 그 뒤의 외부 시스템)다. 도구 **결과**의 종단은 다시 모델이다. MCP 서버는 대화의 끝이 아니라 왕복 중간의 한 홉이다.

## 확인하는 방법

```
$ for c in $(pgrep -P <claude PID>); do ps -o pid=,ppid=,command= -p $c; done
85648 85613 node .../things/server/...
85649 85613 node .../osascript/server/...
85650 85613 node .../chrome-control/server/...
85651 85613 npm exec @upstash/context7-mcp
85652 85613 uv tool uvx --from <사내 MCP 서버 git URL> ...
```

`claude` 프로세스(85613)의 직접 자식 전체다. 로컬 MCP 서버 5개가 모두 여기 있고, 원격 커넥터로 붙은 문서 도구는 없다.

## 이 노트가 틀렸다면

- 로컬 stdio MCP 서버가 `claude` 프로세스의 자식이 아닌 것으로 관측되면 → 「클라이언트가 띄운다」가 이 클라이언트에서는 틀렸다.
- 권한 deny 규칙에 걸린 도구가 그래도 실행된 흔적(MCP 서버 로그의 `tools/call` 수신)이 보이면 → 클라이언트 차단이 결정적이라는 주장이 틀렸다.
- 원격 커넥터 도구를 호출하는 순간 `lsof -p <claude PID>` 에 그 커넥터 호스트로의 연결이 새로 보이면 → 위 `[추정]`(클라이언트가 HTTP 로 직접 보낸다)이 `[실측]` 으로 승격된다. 보이지 않으면 API 쪽 경로일 수 있다.

## 적용 범위

Claude API 의 client tool 과 MCP stdio transport 를 쓰는 에이전트 클라이언트. Claude Code 에서 확인한 것은 프로세스 구조(PPID)뿐이다. deny 에 걸린 호출이 MCP 서버 로그에 `tools/call` 로 도착하지 않는지는 시험하지 않았다. 다른 클라이언트도 같은 구조일 것으로 보지만 확인하지 않았다.

## 함정

- ⚠️ 「모델이 MCP 서버를 부른다」고 생각하기 쉽다. 실제로 모델은 로컬 MCP 서버와 직접 통신하지 않는다. 로컬 서버는 사용자 기기의 권한과 토큰, 네트워크로 동작한다.
- ⚠️ MCP 서버의 결과는 5번에서 그대로 모델 컨텍스트에 들어간다. 남이 쓴 글이 결과에 섞이면 지시로 읽힐 수 있다. → [[untrusted-text-to-tool-agent-becomes-instructions]]
- 클라이언트 세션마다 MCP 서버 묶음이 따로 뜬다. 세션 여러 개를 열면 같은 서버 프로세스가 여러 벌 보인다.

## 검증 이력

| 날짜 | 무엇을 했나 | 결과 |
| --- | --- | --- |
| 2026-10-21 | 최초 작성. `ps` 로 프로세스 트리 확인, MCP spec 과 Claude API 문서 확인 | 로컬 실행은 `[실측]`, 왕복 구조는 `[문서]`, tools 목록 제외 효과와 원격 커넥터 실행 주체는 `[추정]` |

## 관련

- [[untrusted-text-to-tool-agent-becomes-instructions]]
- [[prefix-allowlist-is-not-a-capability-boundary]]
