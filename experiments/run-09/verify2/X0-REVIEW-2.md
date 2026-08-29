# 코드 리뷰

## 요약

- **읽은 파일 28개** — 엔진 소스 22개(`src/Realtime.Assembly/**/*.cs`), 하네스 1개(`harness/Program.cs`, 1,840줄),
  `.csproj` 2개, 문서 3개(`DEVIATIONS.md`, `EXTENSION-NOTES.md`, `ASYMMETRY-NOTES.md`).
  빌드 산출물(`obj/`, `bin/`)은 읽지 않았고, 검증을 위해 이미 빌드되어 있던
  `harness/bin/Debug/net10.0/Realtime.Assembly.Harness.exe`를 **실행만** 했다(21/21 통과).
  대상 디렉터리의 파일은 한 글자도 바꾸지 않았다. 재현 프로그램은 스크래치패드에 따로 만들어
  `Realtime.Assembly.dll`을 참조해 돌렸다.
- **찾은 결점 12개** — 확신 **11개**, 의심 **1개**.
- 가장 심각한 것: `ReassemblyBudget`이 청구하는 바이트와 `BufferPool`이 실제로 잡는 바이트가
  달라, 연결별·전역 되돌리기 예산이 **실제 힙에서는 약 2배까지 초과**될 수 있다(결점 1).

---

## 결점

### 1. 되돌리기 예산이 "실제 메모리"를 세지 않는다 — 등급 반올림만큼 예산이 새어 나간다

- **유형**: 자기모순 / 계산 오류
- **어디**:
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyBudget.cs:7-9` — "묶이는 것은 페이로드 바이트만이 아니라
    건당 부기(A_ctx)를 포함한 **실제 메모리**다"
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyBudget.cs:96-97` — `ChargeFor` = `SealedSizeOf(payloadBytes, fragments) + A_ctx`
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyTable.cs:104` / `:110` — 예약은 `header.TotalLength`로,
    대여는 `_pool.TryRent(header.TotalLength)`로 한다
  - `src/Realtime.Assembly/Stubs/BufferPool.cs:39` — `SizeClasses = [1536, 4096, 16384, 65536]`, `:66` 등급 올림
- **무엇이 어긋나는가**: 주석은 청구액이 "실제 메모리"라고 말한다. 그러나 청구액은 피어가 **선언한 총 길이**에서
  나오고, 실제로 잡히는 힙은 그 길이를 **등급으로 올린** 값이다. 두 값의 차이는 피어가 총 길이를 등급 경계
  바로 위로 고르면 최대가 된다. 건수 축(N_conn = 8 / N_total = 2,048)이 이 차이를 곱한다.
  - 연결별: 8건 × 65,536 B = **512 KiB** 실제 힙 vs `B_conn` 256 KiB → **2배**
  - 전역: 2,048건 × 65,536 B = **128 MiB** 실제 힙 vs `B_total` 64 MiB → **2배**
    (그때 회계가 보고하는 값은 2,048 × 17,383 ≈ 35.6 MiB로, 예산은 "여유 있음"이라고 말한다)
- **재현 방법**: 기본 `EngineOptions`, `PathLimits.Guaranteed`로 표를 만들고
  **총 길이 16,385 B**(= 16,384 등급 바로 위) 메시지의 **첫 조각 하나만** 넣는다.
  실제로 돌려 확인했다:
  ```
  message 16385 B / 15 fragments -> charged 17383 B, BufferPool rents the 65536 size class
                                    (48,153 B unaccounted)
  ```
  `table.Budget.Bytes` = 17,383인데 `BufferPool`이 잡은 배열은 65,536 B다.
  같은 방식으로 한 연결에 8건(서로 다른 메시지 ID)을 열면 회계는 139,064 B(< `B_conn` 262,144)라고 하지만
  힙에는 524,288 B가 잡혀 있다.
- **확신도**: 확신 (수치는 실행으로 확인. 다만 "등급 반올림 회계는 D9 `BufferPool`/`MemoryBudgetLedger`의
  몫"이라는 `BufferPool.cs:34-35`의 면책을 근거로 D7의 결점이 아니라고 볼 여지는 있다. 그렇더라도
  `ReassemblyBudget.cs:7-9`의 "실제 메모리"라는 문장은 코드와 맞지 않는다.)

---

### 2. 마감으로 회수된 메시지의 조각은 `Late`가 아니다 — 두 문서가 서로 다른 말을 한다

- **유형**: 자기모순 / 중복 불일치
- **어디**:
  - `src/Realtime.Assembly/Stubs/DropReason.cs:31-32` — "이미 완성되었거나 **회수된** 메시지의 조각이 뒤늦게 왔다"
  - `src/Realtime.Assembly/MessageAssembly/FragmentAdmission.cs:23-24` — "이미 완성되었거나 **마감으로 회수된** 메시지의 조각이다"
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyTable.cs:166` — `RecordCompleted`를 부르는 곳은 `Retire` 하나뿐
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyTable.cs:174-188` — `ReclaimExpired`는 `ReleaseSlot`만 부르고
    `RecordCompleted`를 부르지 않는다
  - (반대편) `DEVIATIONS.md:53-58` — "**완성된 것만 넣는다.** 만료된 건은 다시 열릴 수 있다"
- **무엇이 어긋나는가**: 두 열거형의 XML 주석은 "마감으로 회수된 것도 `Late`"라고 못박는데, 코드는 완성된 것만
  링에 넣는다. `DEVIATIONS.md`는 코드 쪽 동작을 근거와 함께 적어 두었다 — 즉 **문서 두 벌이 서로 어긋나고**,
  코드는 그중 한쪽만 따른다. 부수 효과로, 마감 뒤 재전송된 조각은 새 건을 열어 예산을 **다시** 청구한다
  (`DEVIATIONS.md`가 "연결별 예산 안에 갇히므로 괜찮다"고 판단한 부분이지만, 열거형 주석은 그 반대를 약속한다).
- **재현 방법**: 기본 설정으로 표를 하나 만들고
  1. 2,500 B 메시지(3조각)의 조각 0을 `t = 0`에 넣는다 → `Accept`, `InProgress = 1`
  2. `ReclaimExpired(new PollTimestamp(12_000))` → 1건 회수, `ReassemblyDeadlinePassed = 1`
  3. 같은 조각 0을 **다시 봉인해서**(새 순번) `t = 12_000`에 넣는다

  실행 결과:
  ```
  A: first fragment      -> Accept   inProgress=1
  A: ReclaimExpired      -> 1 reclaimed, inProgress=0, DeadlinePassed=1
  A: re-sent fragment 0  -> Accept   LateFragment counter=0, inProgress=1, budgetBytes=2802
  ```
  주석이 약속한 `Late` / `LateFragment` 대신 `Accept`가 나오고 예산이 다시 2,802 B 청구된다.
- **확신도**: 확신

---

### 3. `Retire`가 "완성된 건"이라는 전제를 강제하지 않는다 — 미완성 건도 회수되고 ID가 "완성"으로 기록된다

- **유형**: 미강제 불변식
- **어디**:
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyTable.cs:156-168` — 주석: "**완성되어 애플리케이션으로 넘어간
    건**을 회수한다". 코드는 `IndexOf` 결과만 보고 완성 여부는 보지 않는다.
  - 대비: `src/Realtime.Assembly/MessageAssembly/AssemblyGate.cs:8-13` — "강제 수단은 문서가 아니라 접근 제한이다",
    `ReassemblyContext.cs:119-120` — `Reveal`은 미완성이면 예외
- **무엇이 어긋나는가**: 이 코드베이스는 "완성 전에는 아무것도 나가지 않는다"를 **접근 제한으로** 강제한다고
  거듭 말한다(`AssemblyGate`, `ReassemblyContext.Reveal`). 그런데 같은 컨텍스트를 **버리는** 쪽 문(`Retire`)에는
  같은 강제가 없다. `Offer`가 `out ReassemblyContext?`로 컨텍스트를 공개로 돌려주므로,
  애플리케이션이 미완성 건을 `Retire`에 넘길 수 있고 그러면 (a) 조용히 버려지고 (b) 그 메시지 ID가
  "최근 완성" 링에 들어가 (c) **나머지 조각이 도착해도 `Late`로 버려진다**.
- **재현 방법**: 3조각 메시지의 조각 0만 넣어 `touched`를 받은 뒤 `AssemblyGate.TryReveal`을 부르지 않고
  바로 `table.Retire(touched!)`를 부른다. 예외 없이 통과하고 `InProgress`가 0이 된다.
  이어서 조각 1·2를 (새로 봉인해) 넣으면 `Accept`가 아니라 `Late`가 나온다 —
  `WasRecentlyCompleted`가 그 ID를 완성으로 알고 있기 때문이다.
- **확신도**: 확신 (코드 경로가 명확하다. `Retire`에 `ObjectDisposedException.ThrowIf(_closed, …)`조차 없다는
  점도 같은 자리다.)

---

### 4. `EngineOptions.Validate()`의 K9 검사는 결코 실패할 수 없다 — 죽은 검증

- **유형**: 죽은 코드 / 항상 참인 조건
- **어디**:
  - `src/Realtime.Assembly/Stubs/EngineOptions.cs:244-246` — `Require(F * FragmentPayloadCeiling >= M, …, "D_max에서 K4가 깨진다 — K9 위반.")`
  - 앞선 `:214` `Require(DatagramMaxBytes >= DatagramFloorBytes, …)`
  - 앞선 `:240` `Require(F * FragmentPayloadFloor >= M, …)` (K4)
  - (주장하는 쪽) `ASYMMETRY-NOTES.md:69` — "`Validate()`의 K4·K9 … 부등식은 맞았지만 **한 번만** 성립했다"
- **무엇이 어긋나는가**: `D_max ≥ D_floor`(:214)이므로 `FragmentPayloadCeiling ≥ FragmentPayloadFloor`이고,
  K4(:240)가 이미 `F × FragmentPayloadFloor ≥ M`을 통과시켰다. 따라서 `F × FragmentPayloadCeiling ≥ M`은
  **항상 참**이다. :245의 `Require`는 어떤 설정으로도 발화하지 않는다.
  실질적인 D_max 쪽 검사는 :252의 `RequireRelationsFor(DatagramMaxBytes, …)`가 하고 있고, :245는 그 앞에
  남아 있는 사문이다.
- **재현 방법**: "K9 위반" 메시지를 내는 `EngineOptions`를 만들 수 없다.
  `DatagramFloorBytes ∈ {200,600,1200,4000}` × `DatagramMaxBytes ∈ {200,600,1200,1452,4000,20000}` ×
  `MaxMessageBytes ∈ {1000,65536,200000}` × `MaxFragmentsPerMessage ∈ {1,8,64}` = **216가지**를
  전수로 `Validate()`에 넣어 보았다:
  ```
  B: 216 configurations tried, 'K9 위반' raised 0 times
  ```
  (전수 실험은 보조 근거이고, 위의 부등식 연쇄가 증명이다.)
- **확신도**: 확신

---

### 5. `NonceSequencer`가 "논스 구성의 소유자"라고 적혀 있으나 아무도 쓰지 않는다 — 구성은 다른 파일에 복제되어 있다

- **유형**: 죽은 코드 / 중복 불일치
- **어디**:
  - `src/Realtime.Assembly/DatagramSecurity/NonceSequencer.cs:6-8`("논스의 **구성**과 … 의 소유자"),
    `:29` `SaltBytes = 4`, `:32` `CounterBytes = 8`, `:35-41` `Write(…)`
  - `EXTENSION-NOTES.md:43` — "§1.2의 … 를 **소유하는 자리**. 구성(소금 ‖ 계수)과 그 이유를 한 곳에 두고"
  - 실제로 구성을 하는 곳: `src/Realtime.Assembly/DatagramSecurity/SealedDatagramHeader.cs:66-67` —
    `WriteUInt32BigEndian(destination[NonceOffset..], NonceSalt)` / `WriteUInt64BigEndian(destination[(NonceOffset + 4)..], NonceCounter)`
    — **`4`가 그대로 박혀 있고** `NonceSequencer.SaltBytes`를 읽지 않는다
  - `src/Realtime.Assembly/DatagramSecurity/DatagramSealer.cs:96-100` — 논스는 머리말 쓰기의 부산물로 만들어진다
- **무엇이 어긋나는가**: `NonceSequencer.Write`는 **어디에서도 호출되지 않는다**(엔진·하네스 전체 grep 0건).
  논스 12 B의 4/8 분할은 `SealedDatagramHeader.TryWrite`가 자기 상수로 다시 한 번 적고 있다.
  "같은 수를 두 번 적지 않는다"(`SealedDatagramHeader.cs:107`)는 이 파일의 선언과 정면으로 어긋나며,
  `SaltBytes`를 4가 아닌 값으로 바꾸면 `Matches`(검사 15가 쓰는 함수)와 실제 배치가 조용히 갈라진다.
- **재현 방법**: `NonceSequencer.SaltBytes`를 예컨대 6으로 바꾸면 `SealedDatagramHeader.TryWrite`는 그대로
  4/8로 쓰고 `NonceSequencer.Matches`는 6/8로 읽으므로 하네스 검사 10·15가 실패한다
  (한 상수가 두 곳에 있고 한쪽만 움직인다). 호출 0건은
  `grep -rn "NonceSequencer.Write" src harness` 로 즉시 확인된다(0줄).
- **확신도**: 확신

---

### 6. `EngineClock`은 통째로 죽은 코드인데, 주석은 "하네스가 이 시각을 전진시킨다"고 말한다

- **유형**: 죽은 코드 / 자기모순
- **어디**:
  - `src/Realtime.Assembly/Stubs/EngineClock.cs:22-24` — "폴 진입 시 단조 시각을 한 번 확정한다.
    **하네스는 결정론을 위해 이 시각을 직접 전진시킨다.**"
  - `src/Realtime.Assembly/Stubs/EngineClock.cs:32` — "`하네스 전용` — 실물에서는 `Stopwatch` 단조 시각이 들어온다"
  - `harness/Program.cs` 전체 — `EngineClock`·`BeginPoll`·`Advance`가 한 번도 나오지 않는다.
    하네스는 `new PollTimestamp(…)`를 직접 만든다(`:342`, `:532`, `:1143` 등).
- **무엇이 어긋나는가**: 주석이 말하는 사용자가 존재하지 않는다. `EngineClock` 클래스 전체(`BeginPoll`, `Advance`)가
  어느 코드에서도 참조되지 않는다. 쓰이는 것은 같은 파일의 `PollTimestamp`뿐이다.
- **재현 방법**: `grep -rn "EngineClock\|BeginPoll\|\.Advance(" src harness` → `EngineClock.cs` 안의 정의 3줄만 나온다.
  `EngineClock` 클래스를 지워도 엔진과 하네스가 그대로 빌드된다.
- **확신도**: 확신

---

### 7. `Realtime.Assembly.csproj`의 주석이 "이 어셈블리에 암호는 없다"고 말한다 — 그 어셈블리가 AES-GCM과 HKDF를 담고 있다

- **유형**: 자기모순
- **어디**:
  - `src/Realtime.Assembly/Realtime.Assembly.csproj:4` — "D7 메시지 조립만. 소켓·**암호**·핸드셰이크·채널은
    이 어셈블리에 없다."
  - 같은 어셈블리: `DatagramSecurity/DatagramSealer.cs:31` `AesGcm _cipher`,
    `DatagramSecurity/SessionKeys.cs:60` `HKDF.DeriveKey`,
    `DatagramSecurity/DatagramOpener.cs:220` `new AesGcm(...)`
- **무엇이 어긋나는가**: 봉인 확장(`EXTENSION-NOTES.md`)이 `DatagramSecurity/` 7개 파일을 이 어셈블리에 넣었는데,
  프로젝트 파일의 범위 선언은 확장 전 그대로다. 같은 주석의 다른 절반(`InternalsVisibleTo`를 두지 않는다)은
  지금도 맞다 — 즉 이 주석은 부분적으로만 낡았고, 그래서 더 헷갈린다.
- **재현 방법**: 정적인 모순이라 실행으로 드러나지 않는다. `Realtime.Assembly.csproj:4`와
  `src/Realtime.Assembly/DatagramSecurity/` 디렉터리 목록을 나란히 보면 끝난다.
- **확신도**: 확신

---

### 8. 재생 창 폭 256의 근거 주석에 틀린 산수가 있다 ("폭이 64였다면 58칸 늦은 조각이 창 밖")

- **유형**: 계산 오류
- **어디**:
  - `src/Realtime.Assembly/DatagramSecurity/ReplayWindow.cs:9-10` — "폭이 64였다면 조각 하나가 **58칸 늦게 오는
    것만으로 창 밖이 되어**, 정상적인 재정렬이 거절로 보이게 된다."
  - 판정 코드: `ReplayWindow.cs:44-46` `IsBelowWindow` = `_top - sequence >= Width`
- **무엇이 어긋나는가**: `IsBelowWindow`의 기준은 `거리 ≥ 폭`이다. 폭이 64라면 거리 0~63이 창 **안**이므로,
  거리 58은 창 안이다. 도달 가능한 최대 조각 수가 58이라는 것은 한 메시지의 순번 폭이 최대 57(0~57)이라는
  뜻이므로, 폭 64는 "한 메시지가 통째로 뒤섞여 도착"하는 경우를 **덮는다**. 주석은 덮지 못한다고 말한다.
  (폭 256이라는 선택 자체는 넉넉해서 해롭지 않다. 틀린 것은 그 근거다.)
- **재현 방법**: 산술로만 드러난다 — 58 < 64이므로 폭 64에서도 `IsBelowWindow(top-58)`은 `false`다.
  실제로 창 밖이 되려면 거리 ≥ 64여야 한다. 코드 동작에는 영향이 없다.
- **확신도**: 확신 (산술은 명백. "결점"으로 셀지는 산문의 무게를 어떻게 보느냐에 달렸다.)

---

### 9. 스트림 ID의 문서화된 범위(0 또는 1~32)를 아무도 강제하지 않는다

- **유형**: 미강제 불변식
- **어디**:
  - `src/Realtime.Assembly/MessageAssembly/FragmentHeader.cs:13` — "`1  스트림 ID  1  0 = 재전송 없는 갈래(예약),
    **1~32 = 신뢰 갈래**`"
  - `src/Realtime.Assembly/MessageAssembly/FragmentValidator.cs:35-135` — `Validate`는 총 길이·조각 수·색인·
    조각 길이만 본다. 스트림 ID는 보지 않는다.
  - `src/Realtime.Assembly/MessageAssembly/FragmentValidator.cs:151-158` — `MatchesContext`도 **같은지만** 비교하고
    범위는 보지 않는다.
- **무엇이 어긋나는가**: 배치 주석이 값역을 명시하는데 어느 판정도 그 값역을 집행하지 않는다.
  피어가 스트림 ID 33~255를 실어 보내면 그대로 되돌리기 건이 열린다. (`FragmentHeader.UnreliableStreamId`
  상수도 정의만 되어 있고 호출자가 없다.)
- **재현 방법**: 기본 설정의 연결에 2,500 B 메시지의 조각 0을 **`streamId = 200`으로 봉인해** 넣는다.
  실행 결과:
  ```
  E: streamId=200 (documented range is 0 or 1..32) -> Accept
  ```
  `Inconsistent`가 아니라 `Accept`가 나오고 건이 열린다.
- **확신도**: 확신 (동작은 확인됨. 이 값역의 소유가 D6인지 D7인지는 이 범위에서 정해져 있지 않다.)

---

### 10. `SealedDatagramIntake.Offer`가 D7 층의 사유를 지워서 돌려준다 — 하네스가 그 값을 실패 원인으로 찍는다

- **유형**: 자기모순 (진단 경로)
- **어디**:
  - `src/Realtime.Assembly/DatagramSecurity/SealedDatagramIntake.cs:87-88` — `reason = DropReason.None;` 직후
    `return _table.Offer(...)`
  - `harness/Program.cs:1356-1357` — `near.Intake.Offer(…, out DropReason r) != Accept` 이면
    `return $"들어오는 조각 {i} 거절: {r}"`
  - `harness/Program.cs:715-716` — 검사 8에서도 같은 패턴
- **무엇이 어긋나는가**: 봉인 층에서 거절되면 `reason`에 진짜 사유가 담기지만, 봉인이 열린 뒤 D7이 거절하면
  `reason`은 항상 `None`이다(사유는 `CounterBank`에만 남는다). 하네스 검사 17·8은 그 `out` 값을
  "거절 사유"로 출력하므로, 그 갈래가 실제로 실패하면 **`거절: None`**이라는 무의미한 진단이 나온다.
  즉 검사가 깨졌을 때 원인을 말해 주지 못한다.
- **재현 방법**: 검사 17의 `incomingFrames[i]` 하나를 D7이 거절할 조각(예: 총 길이 필드를 0으로 만든 조각)으로
  바꿔 넣으면 `[FAIL] … 들어오는 조각 0 거절: None`이 출력된다. 지금의 통과 경로에서는 드러나지 않는다.
- **확신도**: 확신

---

### 11. `EXTENSION-NOTES.md` §3의 자체 셈이 어긋난다

- **유형**: 자기모순 (문서)
- **어디**:
  - `EXTENSION-NOTES.md:96` — "읽어 낼 것 **두 가지**." 뒤에 항목이 `1.`, `2.`, `3.` **세 개** 온다(:98, :103, :107)
  - `EXTENSION-NOTES.md:92` — 다섯째 행의 세 곳은 스스로 "`FragmentPlan.FrameLengthAt`(**주석**이 …라고 말함),
    `MessageSplitter.WriteFragment` **주석**, 하네스 검사 1"이라고 적는데, 행 머리는 "실행/문서", 개수는 "3곳"
  - `EXTENSION-NOTES.md:94` — "합계 21곳. 그중 **실행되는 곳은 9곳**이고 나머지 12곳은 주석·산문이다"
- **무엇이 어긋나는가**: (a) "두 가지"라고 하고 셋을 적는다. (b) 21 = 2+7+5+4+3, 9 = 2+4+3으로 계산되는데,
  마지막 3곳 중 둘은 그 행이 **스스로 주석이라고 밝힌** 것이다. 그러면 실행되는 곳은 7곳이고 산문은 14곳이다.
  이 문서의 요지가 "실행되는 중복과 산문 중복을 나눠 센다"인 만큼, 그 셈이 어긋난 것은 요지에 직접 걸린다.
- **재현 방법**: 문서 안에서만 확인된다 — :92의 행 내용과 :94의 합계를 대조하면 된다. 실행으로는 드러나지 않는다.
- **확신도**: 확신

---

### 12. `ReassemblyTable.Budget`은 "읽기 전용 스냅숏"이라고 적혀 있으나, 변경 메서드가 공개된 가변 구조체를 값으로 돌려준다

- **유형**: 미강제 불변식
- **어디**:
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyTable.cs:51-52` — "연결별 예산의 **읽기 전용 스냅숏**."
    `public ReassemblyBudget Budget => _budget;`
  - `src/Realtime.Assembly/MessageAssembly/ReassemblyBudget.cs:104` `public bool TryReserve(...)`,
    `:131` `public void Release(...)` — 둘 다 `readonly`가 아니다
- **무엇이 어긋나는가**: `Budget`은 구조체를 **복사**해 돌려준다. `table.Budget.Release(2500, 3)` 같은 호출은
  컴파일되고, 임시 복사본만 바뀌어 **조용히 아무 일도 하지 않는다**. 같은 타입의 읽기 속성들은
  `readonly`로 표시되어 있어(`:72`, `:75`, `:78`, `:87`) 이 두 메서드만 예외인 것이 눈에 띈다.
  "읽기 전용"은 주석의 약속일 뿐 타입이 강제하지 않는다.
- **재현 방법**: 하네스 어디에서든 `peer.Table.Budget.Release(2_500, 3);` 한 줄을 넣으면 컴파일되고
  실행되며, 그 뒤 `peer.Budget.Bytes`는 **변하지 않는다**(임시 복사본이 바뀌었을 뿐).
  경고도 나오지 않는다(`TreatWarningsAsErrors`가 켜져 있는데도).
- **확신도**: 의심 (C# 언어 규칙상 확실히 컴파일·무시되지만, 현재 코드에 그런 호출은 없다.
  "지금 틀린 동작"이 아니라 "약속이 강제되지 않는 자리"다.)

---

## 결점이 아니라고 판단한 것

1. **`MessageSplitter.TryPlan`의 `totalFragments > F` 갈래와 `ReassemblyTable.Offer`의 `FreeSlot() < 0` 갈래가
   도달 불가능하다** (`MessageSplitter.cs:128`, `ReassemblyTable.cs:119`).
   둘 다 실제로 도달 불가능하다 — 전자는 `PathLimits` 생성자가 `RequireRelationsFor`로 이미 막았고,
   후자는 슬롯 수 = `N_conn` = 건수 축 상한이기 때문이다. 그러나 **두 자리 모두 주석이 "도달하지 않는다,
   설정이 바뀌어도 조용히 넘치지 않도록 남겨 둔다"고 명시**하고 있다. 의도된 방어이므로 죽은 코드로 세지 않았다.

2. **`DatagramOpener.TryOpen`이 예외를 던지는 자리**(`DatagramOpener.cs:131-132`, 평문 목적지가 작을 때).
   E3의 "예외를 던지지 않는다"는 **여는 데 실패하는 경로**에 대한 약속이고, 이쪽은 호출자 오류다.
   게다가 `ASYMMETRY-NOTES.md:108-111`(남은 결함 2)이 이 자리를 이미 이름 붙여 기록해 두었다.

3. **`PathLimits`가 스레드 안전하지 않다**(`_outbound`가 가변 필드).
   `ASYMMETRY-NOTES.md:113-116`(남은 결함 3)이 정확히 이것을 적어 두었다. 새로 찾은 것이 아니다.

4. **`DatagramOpener`가 `지금 + 2` 이상의 시대를 영구히 거절한다**.
   `EXTENSION-NOTES.md:145-149`(남은 결함 1)에 근거와 함께 있다. 의도된 맞바꿈이다.

5. **`SealedDatagramHeader.TryRead`가 `TryOpen` 안에서 실패할 수 없다**(`DatagramOpener.cs:111`에서 이미
   길이 40 초과를 보장하므로 24 B 부족이 나올 수 없다).
   파싱과 판정을 분리한다는 이 코드베이스의 규약(`FragmentHeader.cs:65-68`)에서 나온 형태이고,
   `TryRead`는 다른 호출자(하네스 검사 10·14·15)도 있으므로 죽은 코드가 아니다.

6. **`FragmentValidator`의 P 역산이 조각 수를 부풀리는 입력을 놓칠 수 있는가**.
   마지막 조각 갈래(`FragmentValidator.cs:113-131`)와 중간 조각 갈래(`:91-112`)를 각각 따라가
   `(n-1)P < L ≤ nP`가 두 갈래 모두에서 성립함을 확인했다. `MatchesContext`가 이후 모든 조각에 같은
   `(L, n, P, streamId)`를 요구하므로 두 갈래가 서로 다른 P를 만들 수 없다. 구멍을 찾지 못했다.
   `DEVIATIONS.md`의 "도달 가능한 최대 조각 수 58, 59~64는 불가능"도 코드로 다시 유도해 맞았다
   (n=59면 58 × 1,142 = 66,236 > M = 65,536).

7. **`ReplayWindow.ShiftUp`의 `stackalloc` 결과 배열이 0으로 채워지지 않을 수 있는가**(`ReplayWindow.cs:113`).
   `SkipLocalsInit`이 없으므로 `.locals init`가 붙어 0으로 채워진다. `src < 0`인 워드가 0으로 남는 것이 맞다.

8. **`BufferPool`의 세대 토큰이 되감기는가**. `TryRent`와 `Return`이 각각 1씩 올리고 0을 건너뛰므로
   (`BufferPool.cs:76-77`, `:96-97`) 실무 범위에서 반복되지 않는다. 반납 후 접근은 실제로 예외가 된다.

9. **하네스의 숫자들**. 사전 확인이 찍는 H = 58(24+18+16), 바깥 덧붙이 40, P_floor 1,142, P_ceiling 1,394,
   W_msg 69,028, 도달 가능 최대 조각 수 58, 나가는 쪽 D_max 계산과의 차 580 B, 전수 253개 값 —
   전부 문서(`ASYMMETRY-NOTES.md:48-53`, `DEVIATIONS.md:69-85`)와 코드와 실행 출력이 일치했다.
   21/21이 통과한다.
