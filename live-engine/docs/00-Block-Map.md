# Live Engine — 원본 모듈 → SBA 블록 매핑

> 원본: `M:/Projects/Live/Live DTS` (.NET 10, 46파일 / 11.6k라인).
> 근거 문서: `redesign.md`, `LIVE DTS/architecture.md`, `audit-findings.md`, `session-summary.md`.
> **소스 코드는 열지 않았습니다.** 문서에 명시된 와이어 레이아웃·상수·검증 순서만 따랐고,
> 문서에 없는 세부(HKDF 라벨, 다이제스트 계산식 등)는 자체 정합성으로 재도출했습니다.

## 매핑의 원리

원본은 **기술 계층**으로 나뉘어 있습니다(Channels / Protocol / Security / Sessions / Live).
SBA는 계층을 유지하되 각 계층 안에서 **타입 하나가 여러 책임을 지는 것**을 해체합니다.
원본의 큰 타입 하나가 SBA에서 파이프라인 1 + 파편 N + 레코드 M으로 분해되는 것이 기본형입니다.

| 원본 타입 | SBA 분해 | 분해 사유 (G1: 요약문에 접속사가 필요한가) |
|---|---|---|
| `WireFormat` | 파편 ~18개 + 레코드 ~9개 | "모든 TCP **그리고** UDP 프레임을 인코딩 **그리고** 디코딩" — 접속사 3개 |
| `FrameReader` | `ReadTcpFramePipeline` + 파편 3 | 길이 파싱 / 쿼터 검사 / 조립이 각각 독립 실패 지점 |
| `EngineConfig` | 레코드 1 + 검증 파편 ~12 | "설정을 담고 **그리고** 상호 검증" — 담기와 검증은 별개 |
| `OutboundSequencer` | 자원 1 + 파편 2 | 카운터 상태(자원) vs. 논스 구성·소진 판정(파편) |
| `UdpReplayWindow` | 자원 1 + 파편 3 | 슬라이딩 윈도우 상태 vs. RFC1982 비교·비트맵 판정 |
| `TcpChannel` | 자원 1 + 파이프라인 3 + 파편 4 | 소켓 수명(자원) vs. 읽기/쓰기/종료 유즈케이스 |
| `Reassembler` | 자원 1 + 파이프라인 2 + 파편 6 | 캡·중복·타임아웃·커버리지가 전부 독립 거부 사유 |
| `ReliableSender` | 자원 1 + 파편 7 | 윈도우 상태 vs. RTO 계산·빠른 재전송 판정·혼잡 창 산술 |
| `ConnectionRegistry` | 파이프라인 4 + 원장 1 | Add / CompleteFresh / TryCompleteResume / Release가 각각 유즈케이스 |
| `ITelemetry` + `DropReason` | `DropReasonRecord` + `RefusalLedger` + `Verdict<T>` | 규약 §6 — 거부가 사유를 **타입으로** 들고 다니게 전환 |

---

## 단계별 구성 (원본 7단계와 동형)

| 단계 | 도메인 | 원본 대응 | 상태 |
|---|---|---|---|
| 1 | `Wire/` | Protocol_Shared | 진행 중 |
| 2 | `Memory/`, `Transport/` | Memory_Shared, Channels_Shared | 대기 |
| 3 | `Fragmentation/` | Fragmentation_Shared | 대기 |
| 4 | `Security/` | Security_Shared | 대기 |
| 5 | `Sessions/`, `Reliability/` | Sessions_Shared | 대기 |
| 6 | `Facade/` | Live_Shared | 대기 |
| 7 | `tests/` 하네스 | Tests | 대기 |

각 단계는 독립적으로 컴파일되고 하네스로 검증된 뒤 다음 단계로 넘어갑니다.
단계별 파편 목록은 그 단계를 시작할 때 이 문서에 확정 기록합니다 — 미리 적어둔 목록은
반드시 실제와 어긋나므로, 지키지 못할 계획을 문서에 남기지 않습니다.
