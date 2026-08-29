# 코드 리뷰

대상: `experiments/run-09/X1/` (net10.0, 단일 프로젝트). `obj/`·`bin/` 산출물은 제외.

검증 방법: `X1/`을 임시 디렉터리로 **복사**해서 빌드·실행했습니다. `X1/` 안의 파일은
한 글자도 바꾸지 않았습니다. 원본 하네스는 **21개 검사가 전부 통과**합니다
(상수 관계 K1–K9 / X1–X8 / D1·D6·D7·D9·D10 도 전부 OK). 아래 결점은 전부
**현재 검사가 잡지 못하는 잠복 결함**입니다.

## 요약

- 읽은 파일: **53개** (소스 45 + 하네스 1 + `X1.csproj` 1 + 문서 3, 그리고 문서 3개는 전문)
- 찾은 결점: **9건**
  - 확신: **7건** (1·2·3·4·5·6·7)
  - 의심: **2건** (8·9)
- 그중 4건(1·2·4·5)은 임시 복사본에서 **실제로 재현**했고, 2건(3·6)은 전수 탐색 또는
  산술로 도달 불가능성을 확인했습니다.

---

## 결점

### 1. 인증되지 않은 데이터그램 하나가 연결별 보안 연관과 AES 키를 영구히 만들어 낸다 — 파이프라인 자신의 E4 주석과 정면으로 어긋난다

- **유형**: 자기모순 / 미강제 불변식
- **어디**:
  - `X1/src/Security/Pipelines/DatagramOpenPipeline.cs:98` — `var receive = _associations.ReceiveAssociationFor(connectionTag);`
  - 같은 파일 `:28-32` (클래스 주석) — *"A datagram that fails to open leaves this pipeline with **every** piece of state exactly as it found it"*
  - 같은 파일 `:74-76` (1b단계 주석) — *"no key is derived and no state of any kind is left behind"* (1b단계에 한정된 진술이지만, 그 아래 단계에는 같은 보장이 없다는 뜻이 된다)
  - `X1/src/Security/Resources/SecurityAssociationTableResource.cs:58-66` — `ReceiveAssociationFor`는 조회가 아니라 **없으면 만들어 넣는** 동사다. 만들면서 `KeyFor(...)` → `EpochKeyResource` → `HKDF.DeriveKey` + `new AesGcm(...)`까지 간다.
- **무엇이 어긋나는가**: 주석은 "열리지 않은 데이터그램은 어떤 상태도 남기지 않는다"고
  말한다. 코드는 3단계(시대 판정)에 들어가기 **전에** 연관 표에 새 행을 만들고 시대 0
  키를 유도한다. 그 데이터그램이 `SealedDatagramEpochUnknown` 또는
  `SealedDatagramAuthenticationFailed`로 거절되어도 행과 키는 남는다.
  `SecurityAssociationTableResource.ReclaimConnection`을 부르는 파이프라인은
  이 코드베이스에 없다(`ASYMMETRY-NOTES.md` §5-6이 그 사실을 인정한다).
  즉 **연결 태그 축은 피어가 몰 수 있는데 아무도 세지 않는 축**이고, 이것은
  `AdmissionVerdictRecord.cs:6-8`이 인용하는 S2("세지 않은 축은 정의상 무한하다")가
  금지하는 바로 그 모양이다.
- **재현 방법** (복사본에서 실행해 확인함):
  ```
  world = 새 엔진 (EngineSecurity.ReceiveAssociationCount == 0)
  200 B 짜리 난수 배열 하나를 만들고, 오프셋 0(시대 필드)만 바꿔 가며
  서로 다른 연결 태그 500개로 Receive.AcceptSealedDatagram(tag, garbage) 500회
  ```
  실측 출력:
  ```
  before: receiveAssociations=0
    first outcome: Dropped / SealedDatagramEpochUnknown
  after 500 forged datagrams on 500 fresh tags: receiveAssociations=500
    arena occupancy zero=True, pool leases=0, reassembly journal=0
  ```
  되돌리기 쪽 상태는 정말로 0이지만(그래서 검사 11·12·18이 통과한다), 보안 연관은
  500개가 남는다 — 각각 32 B 키 + `AesGcm` 인스턴스 + 사전 항목. 검사 11·12·18의
  "leaves no state" 단언은 `Arena`·`Pool`·`Aggregate`만 보고 `EngineSecurity`를 보지 않는다.
- **확신도**: 확신

---

### 2. 완성된 메시지가 배달되기 전에 "마감 만료"로 회수되어 조용히 사라진다

- **유형**: 자기모순 / 계산(순서) 오류
- **어디**:
  - `X1/src/Assembly/Pipelines/ReassemblyExpirySweepPipeline.cs:8-9` (주석) —
    *"what is left **unfinished** is reclaimed once enough time has passed"*
  - 같은 파일 `:44-50` — 실제로는 `OpenSlots()` 중 마감이 지난 것을 **완성 여부와 무관하게** 전부 회수한다. 배달이 이미 적립되었는지 묻는 코드가 없다.
  - `X1/harness/Program.cs:1646-1653` (`AssemblyWorld.Poll`) — `Sweep.SweepExpired()`가 `Delivery.DrainToApplication()`보다 **먼저** 돈다.
  - 배경: `DEVIATIONS.md` #5가 "트리거는 적립만 하고 슬롯 반납은 배달 파이프라인이 한다"고 못 박았기 때문에, 완성된 슬롯은 다음 폴까지 `Open` 상태로 남는다.
- **무엇이 어긋나는가**: 마지막 조각이 도착해 완성 판정이 나고 배달이 적립된 뒤,
  애플리케이션이 다음 폴을 `ReassemblyDeadlineMilliseconds` 이후에 하면, 그 폴의 쓸기가
  완성된 슬롯을 먼저 회수한다. 그러면 (a) 완성된 메시지가 애플리케이션에 **한 번도**
  전달되지 않고, (b) `ReassemblyDeadlineExpired`(등급 `InvestigatePeer`)로 집계되어
  운영자에게 "피어가 메시지를 끝내지 못했다"고 잘못 보고된다.
  배달 큐의 항목은 다음 폴에 `slot.State != Open`으로 조용히 버려진다
  (`ReassemblyDeliveryPipeline.cs:50`).
- **재현 방법** (복사본에서 실행해 확인함):
  ```
  Poll(1_000)
  40,000 B 메시지를 계획(37조각) → 37개 조각을 전부 봉인해 전부 수신
    (여기서 완성 판정이 나고 배달이 적립됨: deferred work = 1, open slots = 1)
  Poll(1_000 + 20_000)          // 즉 Poll(21_000) — 마감 정각
  ```
  실측:
  ```
  fragments=37; deferred work queued=1; open slots=1
  after Poll(21000): deliveries=0 deadlineExpired=1 occupancyZero=True poolLeases=0
  after another poll: deliveries=0
  ```
  대조군 — 같은 것을 `Poll(20_999)`로 하면 `deliveries=1, deadlineExpired=0`.
  또한 폴 안의 두 호출 순서만 바꿔(배달 먼저, 쓸기 나중) 돌리면
  `deliveries=1, deadlineExpired=0`이 되므로, 결함은 **쓸기가 완성된 슬롯을 걸러내지
  않는다는 것**과 **폴의 순서** 두 자리 중 어느 쪽을 고쳐도 사라진다.
- **확신도**: 확신

---

### 3. 관계 X7은 어떤 값에서도 발화할 수 없다 — "`SealKeyScheduleExhausted`는 산술적으로 도달 불가능하다"는 두 곳에 적혀 있고 아무 데서도 강제되지 않는다

- **유형**: 미강제 불변식 / 항상 거짓인 조건
- **어디**:
  - `X1/src/Stubs/EngineConstantsRecord.cs:357-360`
    ```csharp
    // X7 — the byte axis rolls an epoch long before the 2^64 sequence space can run out,
    // so SealKeyScheduleExhausted is unreachable by arithmetic (the reason stays, S2).
    if (EpochByteLimit >= long.MaxValue / MinSealedDatagramBytes)
        return "X7: the epoch byte limit no longer bounds the sequence count";
    ```
  - `X1/src/Stubs/DropReasonIndex.cs:64` — *"Unreachable by arithmetic while relation X7 holds."*
  - `X1/EXTENSION-NOTES.md:191-195` — *"관계 X7이 `EpochByteLimit`이 순번 공간 2^64보다 훨씬 먼저 시대를 넘긴다는 것을 고정합니다"*
- **무엇이 어긋나는가**: 주석이 말하는 성질은 "한 시대 안의 데이터그램 수
  (`EpochByteLimit / MinSealedDatagramBytes`)가 순번 공간 2^64보다 작다"이다.
  코드가 실제로 비교하는 것은 `EpochByteLimit`과 `long.MaxValue / MinSealedDatagramBytes`
  이고, 이는 **2^64와 아무 관계가 없다**. 실측값은
  `lhs = 16,777,216`, `rhs = 174,025,887,487,825,958` — 1천만 배 차이다.
  게다가 바로 위의 X6이 `EpochByteLimit ≤ 2^32 = 4,294,967,296`을 이미 고정하므로,
  **X6을 통과하는 어떤 값에서도 X7은 발화할 수 없다.** 즉 항상 거짓인 분기다.
  더 나아가, `SealKeyScheduleExhausted`에는 도달 경로가 둘 있는데
  (`SealedDatagramSendPipeline.cs:107-109`의 `IsEpochSpaceExhausted`,
  `:115-118`의 `IsSequenceSpaceExhausted`), X7은 후자에 대해서만 말하고
  `DropReasonIndex`의 주석은 "the epoch counter **or** the sequence counter"라며 둘 다를
  X7에 걸어 놓았다. 시대 카운터(`Epoch == uint.MaxValue`)의 도달 불가능성을 고정하는
  관계는 어디에도 없다.
- **재현 방법**: 런타임 실패로 재현되지 않습니다 — **결함의 내용이 "이 검사는 절대
  실패할 수 없다"는 것**이기 때문입니다. 확인은 두 가지로 했습니다.
  (a) 상수를 그대로 두고 `EpochByteLimit`과 `long.MaxValue / MinSealedDatagramBytes`를
  찍으면 위 두 수가 나오고, (b) X6이 허용하는 최댓값 2^32를 넣어도
  `4,294,967,296 < 174,025,887,487,825,958`이므로 여전히 통과합니다.
  의도한 성질을 고정하려면 비교가 `EpochByteLimit / MinSealedDatagramBytes`와
  `ulong.MaxValue` 사이여야 합니다.
- **확신도**: 확신

---

### 4. 송신 파이프라인이 길이를 확인하기 **전에** 스크래치 버퍼에 쓴다 — 사유를 붙인 거절 대신 예외가 난다

- **유형**: 자기모순 / 계산(순서) 오류
- **어디**:
  - `X1/src/Security/Pipelines/SealedDatagramSendPipeline.cs:19-27` (주석) — 레인의 순서를
    *"1. write the fragment header and its payload into the scratch buffer / 2. does it fit
    the caller's buffer, and this connection's outgoing limit"* 로 적어 놓았다.
  - 같은 파일 `:77-78` — 1단계가 `_sealedAreaScratch`(`MaxSealedAreaBytes` = **1,374 B**)에
    먼저 쓴다.
  - 같은 파일 `:86-90` — 2단계의 `datagramLength > MaxSealedDatagramBytes` 검사는 그 뒤에 온다.
  - 관련: `X1/src/Assembly/Pipelines/MessageFragmentationPipeline.cs:49` — `public` 과부하
    `PlanSplit(길이, 조각페이로드, 태그)`는 조각 페이로드에 상한을 걸지 않는다
    (`MessageSplitFragment.Split`은 조각 **수** ≤ 64와 메시지 길이 ≤ 65,536만 본다).
- **무엇이 어긋나는가**: 2단계 검사는 "이 데이터그램은 만들 수 없다"를 `Verdict`로
  돌려주기 위해 있는데, 그보다 큰 계획이 오면 1단계가 먼저 스팬 밖으로 복사하려다
  예외를 던진다. 그 결과 호출자가 받는 것은 `SealedDatagramLengthOutOfRange`가 아니라
  `ArgumentOutOfRangeException`이다. 두 과부하 모두 `public`이므로 엔진 사용자가
  정상적인 API 사용만으로 도달할 수 있다.
- **재현 방법** (복사본에서 실행해 확인함):
  ```csharp
  var plan = fragmentation.PlanSplit(65_536, 65_536, tag);   // 받아들여진다: 1조각 × 65,536 B
  var big  = new byte[200_000];                              // 호출자 버퍼는 충분히 크다
  seal.WriteSealedFragmentDatagram(tag, plan.Value, 1, 0, new byte[65_536], big);
  ```
  실측:
  ```
  PlanSplit(65536, 65536) accepted=True count=1 uniform=65536
  THREW ArgumentOutOfRangeException: Specified argument was out of the range of valid values.
  ```
  (`MessageFragmentationPipeline.WriteFragmentCarrier`의
  `carrier.Slice(12, 65536)`가 1,374 B 스크래치에서 터진다.)
- **확신도**: 확신

---

### 5. `MessageSplitPlanRecord`는 네 개의 불변식을 선언해 놓고 하나도 강제하지 않는다

- **유형**: 미강제 불변식
- **어디**:
  - `X1/src/Assembly/Records/MessageSplitPlanRecord.cs:12-16` — *"Stated invariants: 1 ≤
    FragmentCount ≤ MaxFragmentsPerMessage / FragmentCount == ceil(Total / Uniform) /
    Uniform ≤ the path's per-fragment payload / every boundary lies inside [0, Total)"*
  - 같은 파일 `:24-29` — 생성자는 세 정수를 그대로 대입한다. 검사 함수가 없다.
  - 대조: 같은 규약(§3.7, "값의 불변식은 값을 든 클래스에 산다")을 따르는 다른 두 값은
    자기 검사 함수를 가진다 —
    `X1/src/Assembly/Records/FragmentationHeaderRecord.cs:64` (`FindDeclaredInvariantViolation`),
    `X1/src/Path/Records/PathDatagramLimitsRecord.cs:77` (같은 이름).
- **무엇이 어긋나는가**: `AdmissionVerdictRecord`(생성자에서 throw)와
  `ReassemblyProgressRecord`(생성자에서 throw)와 `ReassembledMessageRecord`(`TryCreate`)는
  모두 "진술한 것을 강제한다"를 실제로 지키는데, 이 값만 진술만 하고 아무 것도 지키지
  않는다. 실제 강제는 `MessageSplitFragment.Split`에만 있고, 타입은 그것을 요구하지 않는다.
  `MessageSplitPlanRecord`는 `public`이고 `SealedDatagramSendPipeline.WriteSealedFragmentDatagram`
  의 인자이므로 조립되지 않은 계획이 그대로 흘러 들어간다.
- **재현 방법** (복사본에서 실행해 확인함):
  ```csharp
  var a = new MessageSplitPlanRecord(40_000, 0, 0);
  // 실측: count=0 uniform=0 LengthOf(0)=0 header.count=0
  var b = new MessageSplitPlanRecord(40_000, 200, 5_000);   // 200 × 5,000 = 1,000,000 B
  // 실측: OffsetOf(199)=995000  (총 길이는 40,000),  LengthOf(199)=-955000
  ```
  즉 "모든 경계는 [0, TotalLength) 안에 있다"가 깨진 값이 아무 저항 없이 만들어지고,
  `LengthOf`는 **음수**를 돌려준다. (4번과 결합하면 이 값이 그대로 송신 파이프라인의
  `Slice(offset, length)`로 들어간다.)
- **확신도**: 확신

---

### 6. 도달 불가능한 방어 분기 넷 — 셋은 주석이 "이 경로가 있다"고 말하고 있다

- **유형**: 죽은 코드 / 항상 거짓인 조건
- **어디와 무엇이 어긋나는가**:

  **(a) `FragmentOffsetBoundsFragment`의 두 번째 경계 검사**
  `X1/src/Assembly/Fragments/FragmentOffsetBoundsFragment.cs:54-57` —
  `if (offset + expected > total)`.
  바로 위 `:48-52`가 `arrivedPayloadLengthBytes != expected`를 이미 걸렀고,
  `expected`는 `FragmentationHeaderRecord.ExpectedPayloadLengthBytes`
  (`FragmentationHeaderRecord.cs:49-58`)로 `min(uniform, total − offset)`이므로
  `offset + expected ≤ total`이 항상 성립한다.
  *재현*: 실패로 재현되지 않습니다 — 전수로 확인했습니다. `FindDeclaredInvariantViolation()`
  을 통과하는 (TotalLength 1…65,536) × (FragmentCount 1…64) × (모든 인덱스) 조합
  **134,196,400개**를 전부 돌려 이 분기가 참이 된 횟수는 **0**이었습니다.

  **(b)·(c) `FragmentReceivePipeline.OpenSlot`의 §5-3 보상 경로와 슬롯 고갈 경로**
  `X1/src/Assembly/Pipelines/FragmentReceivePipeline.cs:162-165` (주석) —
  *"on a failure the earlier steps are cancelled in reverse"* 라고 그 경로가 있다고 말한다.
  `:198-203` — `_pool.Borrow(...)` 뒤의 `if (!_pool.IsValid(lease))` → `CancelReservation`.
  그런데 `X1/src/Stubs/BufferPoolResource.cs:66-90`의 `Borrow`는 **유효하지 않은 대여를
  돌려주지 않는다**: 상한을 넘으면 `:69`에서 `ArgumentOutOfRangeException`을 던지고,
  그 밖에는 언제나 `TierBytes ≥ 1,024`인 유효한 대여를 돌려준다.
  따라서 `CancelReservation`(=`ReassemblyArenaResource.cs:124`)을 부르는 코드 경로가
  이 코드베이스에 **하나도 없다**.
  같은 함수 `:192-195`의 `if (slot is null)` → `ReassemblyTotalSlotBudgetExhausted`도
  마찬가지다: 자유 목록 길이는 항상 `1,024 − _totalSlotCount`이고 승인이
  `total.RemainingSlots ≥ 1`을 이미 확인했으므로 `Reserve`가 null을 돌려줄 수 없다.
  *재현*: 실패로 재현되지 않습니다. `Borrow`를 1·1,023·1,024·1,025·65,535·65,536 B로 부르면
  6번 모두 유효한 대여가 나오고(무효 0회), 65,537 B는 `ArgumentOutOfRangeException`을 던집니다.

  **(d) `MessageSplitFragment`의 고정점 반복과 사후 재검증**
  `X1/src/Assembly/Fragments/MessageSplitFragment.cs:49-63`.
  `c = ceil(L/P)`, `u = ceil(L/c)`이면 `L/u ≤ c`이므로 `ceil(L/u) ≤ c`이고,
  `u ≤ P`이므로 `ceil(L/u) ≥ ceil(L/P) = c` — 즉 항상 첫 회에 `settled == c`로 끊긴다.
  루프의 `fragmentCount = settled` 줄과 8회 가드, 그리고 `:57-63`의 재검증(거절 사유
  `MessageAboveFragmentCountCap`)은 전부 도달 불가능하다.
  *재현*: 실패로 재현되지 않습니다. 조각 페이로드 1…2,000 × 메시지 길이 1…65,536의
  모든 쌍(조각 수 ≤ 64인 것)에 대해 두 번째 반복이 필요한 경우의 수는 **0**이었습니다.

- **확신도**: 확신 (네 자리 모두 "동작이 틀렸다"가 아니라 "선언한 경로가 존재하지 않는다"입니다. (b)·(c)는 주석이 있다고 말하는 경로여서 문서-코드 불일치이기도 합니다.)

---

### 7. 하네스가 자기 검사 수를 세 곳에서 다르게 말한다

- **유형**: 중복 불일치 / 자기모순
- **어디**:
  - `X1/harness/Program.cs:22` — *"Fifteen checks, each printing PASS or FAIL"* — 실제로는 `:42-65`에 **21개**가 등록되어 있다.
  - `X1/harness/Program.cs:90` — 성공 시 `$"all {checks.Length} checks passed"` → "all **21** checks passed"
  - `X1/harness/Program.cs:91` — 실패 시 `$"{failed} of {checks.Length + 1} checks failed"` → "… of **22** checks failed"
  - `X1/EXTENSION-NOTES.md:225` — *"열다섯 검사가 전부 통과하면 종료 코드 0"*
  - `X1/ASYMMETRY-NOTES.md:181` — *"스물한 검사가 전부 통과하면 종료 코드 0"*
- **무엇이 어긋나는가**: 같은 사실(검사 개수)이 다섯 곳에 있고 15 / 21 / 22 세 값이 공존한다.
  `Program.cs:22`의 "Fifteen"은 명백히 낡았고, `ASYMMETRY-NOTES.md` §2가 검사 16–21을
  덧붙였다고 적으면서 그 파일의 클래스 주석은 고치지 않았다.
  분모 21과 22의 불일치는 상수 관계 검사를 성공 문구에서는 세지 않고 실패 문구에서는
  세기 때문이다 — 하나라도 실패하면 "1 of 22 checks failed"가 출력되는데,
  성공하면 "all 21 checks passed"가 출력된다.
  `EXTENSION-NOTES.md`가 낡은 것에 대해 `ASYMMETRY-NOTES.md` §5-7이 면책을 적어 두었지만,
  그 면책은 **수치**(1,142·1,362·58조각·36조각)에 대한 것이고 검사 개수는 언급하지 않는다.
- **재현 방법**: `dotnet run` — 헤더 주석은 15를 말하는데 출력은 21줄의 `[PASS]`와
  `all 21 checks passed`가 나온다. 단언 하나를 뒤집으면(예: 검사 1의
  `Require(plan.Value.FragmentCount > 1, …)`) `1 of 22 checks failed`가 나온다.
- **확신도**: 확신 (문서-코드 불일치이며 동작에는 영향이 없다)

---

### 8. 검사 7이 데이터그램이 실제로 다니는 연결이 아닌 다른 연결의 상한으로 쪼갠다 — `ASYMMETRY-NOTES` §2의 설명과 어긋난다

- **유형**: 중복 불일치 (문서 ↔ 코드)
- **어디**:
  - `X1/harness/Program.cs:405` — `var plan = world.RequirePlan(report, message.Length, 1);`
    → 세 번째 인자는 **연결 태그** (`RequirePlan(CheckReport, int, ulong)`,
    `Program.cs:1669`). 즉 태그 `1`의 나가는 쪽 상한으로 계획한다.
  - 같은 검사 `:412-416` — 그런데 봉인·수신은 태그 `0x7000_0000_0000_0000 | peer`로 한다.
  - `X1/ASYMMETRY-NOTES.md:49` — 검사 목록에 **7**을 포함해서, 이 교체가
    *"두 번째 인자를 지운 것이 아니라, **그 연결의** 나가는 쪽 상한에서 읽게 한 것"* 이라고
    설명한다.
- **무엇이 어긋나는가**: 검사 7에서 "그 연결"은 계획에 쓰인 태그 1이고, 데이터그램이
  다니는 400개 연결과 다르다. 지금은 어느 태그에도 상한이 진술되어 있지 않아
  `LimitsFor`가 모두 `Conservative`(바닥 1,200)를 돌려주므로 값이 우연히 같고,
  그래서 검사가 통과한다. 검사 7이 나중에 피어 상한을 진술하게 되면 계획은 여전히
  태그 1에서 읽히고 단언은 조용히 다른 것을 뜻하게 된다.
- **재현 방법**: 현재 상태로는 **관측되는 잘못된 결과가 없습니다** — 값이 같기 때문입니다.
  드러나게 하려면 검사 7 안에서 임의의 피어 하나에 대해
  `world.PeerPathUpdates.ConfigureConnection(tag, 1_452, 1_452)`를 부른 뒤
  `world.SealFragment(...)`의 결과 길이를 보면 됩니다: 그 피어의 데이터그램도 여전히
  1,110 B 페이로드(= 태그 1의 상한)로 쪼개집니다. 하네스를 고치지 않고는 재현되지 않으므로
  "의심"으로 둡니다.
- **확신도**: 의심 (동작은 지금 옳고, 문서의 설명과 호출 형태가 어긋난 자리)

---

### 9. `EXTENSION-NOTES` §7의 `EpochByteLimit` 근거 수치 1,164 B가 어떤 시점의 값과도 맞지 않는다

- **유형**: 중복 불일치 (문서 ↔ 코드)
- **어디**:
  - `X1/EXTENSION-NOTES.md:80` — *"2^24는 **하한 경로 데이터그램(1,164 B)** 기준 약 14,400통"*
  - `X1/src/Stubs/EngineConstantsRecord.cs:24,29,54,63-64,137,146,149` — 현재의 하한 경로
    데이터그램은 `SealedDatagramCeilingFor(1200)` = 40 + 12 + 1,110 = **1,162 B**.
- **무엇이 어긋나는가**: 확장 시점의 띠 바닥은 1,232 B였고 그때의 조각 페이로드는
  1,142 B였으므로 그때의 하한 경로 데이터그램은 40 + 12 + 1,142 = **1,194 B**다.
  비대칭 변경 뒤의 값은 1,162 B다. 문서의 1,164 B는 둘 중 어느 것도 아니다.
  (`ASYMMETRY-NOTES.md` §5-7이 인정한 "낡은 수치" 목록 — 1,142·1,362·58조각·36조각 —
  에도 이 수는 없다.) 16,777,216 / 1,164 ≈ 14,413이라 "약 14,400통"이라는 결론은
  이 수에 붙어 있으므로, 올바른 값(1,194 → 약 14,050 / 1,162 → 약 14,438)으로 다시
  계산하면 근거 문장이 달라진다.
- **재현 방법**: 코드로 재현되는 결함이 아니라 산술 대조입니다.
  `EngineConstantsRecord.SealedDatagramCeilingFor(EngineConstantsRecord.OutboundDatagramLimitFloorBytes)`
  를 찍으면 1,162가 나오고(실측 확인), 확장 시점 값 1,232로 같은 식을 손으로 풀면 1,194가
  나옵니다. 1,164를 내는 정의를 찾지 못했습니다 — 그래서 "의심"입니다.
- **확신도**: 의심

---

## 결점이 아니라고 판단한 것

- **`FragmentDuplicateFragment.Evaluate`의 `1UL << fragmentIndex`가 인덱스 ≥ 64에서 감싼다.**
  결점이 아닙니다. 호출 전에 `FragmentationHeaderRecord.FindDeclaredInvariantViolation`이
  `FragmentCount ≤ 64`와 `FragmentIndex < FragmentCount`를 모두 강제하므로
  (`FragmentReceivePipeline.cs:101-105`가 6단계보다 앞) 인덱스는 항상 0…63입니다.

- **`ReplayWindowFragment.Advance`의 `<< (int)stepsForward` / `<< (int)stepsBack`.**
  결점이 아닙니다. 두 자리 모두 앞에서 `stepsForward >= Width`(→ 비트맵을 `1UL`로 리셋),
  `stepsBack < Width`(`Evaluate`가 이미 걸렀음)로 막혀 있어 64 이상 시프트가 나올 수
  없습니다.

- **다음 시대 키를 인증 전에 유도하고, 열리지 않으면 버린다.**
  결점이 아닙니다. `DatagramOpenPipeline.cs:135`(재생 거절)와 `:152`(인증 실패) 두 경로
  모두 `key.Dispose()`를 부르고, 열린 경우에는 `receive.Promote(key, …)`가 소유권을
  가져갑니다. 세 경로가 모두 덮여 있어 누수가 없습니다. 유도 자체의 비용은
  `EXTENSION-NOTES.md` §5-3에 이미 결함으로 적혀 있습니다.

- **`ReceiveSecurityAssociationResource`가 유예 지난 옛 키를 다음 승격까지 들고 있는 것.**
  결점이 아닙니다 — `EXTENSION-NOTES.md` §5-2가 그 선택과 대가를 명시적으로 적었고,
  `EpochAdmissionFragment`가 `SealedDatagramEpochGraceExpired`로 거절하므로 열리지는
  않습니다. 연결당 키 두 개로 유계입니다.

- **`ReassemblyDeliveryPipeline`이 배달 항목을 `(SlotIndex, ReassemblyId)`로만 대조하고
  `ConnectionTag`는 보지 않는 것.**
  결점이 아닙니다. 슬롯이 회수되어 다른 연결이 같은 `ReassemblyId`로 재사용하는 시나리오를
  따라가 보면, 낡은 항목이 새 주인의 메시지를 한 번 배달하고 슬롯을 놓기 때문에
  그 뒤에 오는 새 주인의 항목은 `State != Open`으로 걸러집니다 — 중복 배달도, 잘못된
  수신자도 생기지 않습니다(`ReassembledMessageRecord`의 `ConnectionTag`는 슬롯의 현재
  값에서 옵니다).

- **`ReassemblyArenaResource.ReleaseAxes`에서 전체 카운터 감소가 연결 행 존재 여부의
  `if` 밖에 있는 것.**
  결점이 아닙니다. 행은 `Reserve`가 항상 만들고, 행 삭제 조건이 세 축 모두 0일 때이므로
  열린 슬롯을 가진 연결의 행이 사라지는 경우가 없습니다.

- **`DEVIATIONS.md`의 수치 열일곱 자리가 낡은 것.**
  결점이지만 **이미 신고된 것**입니다 — `EXTENSION-NOTES.md` §5-1과
  `ASYMMETRY-NOTES.md` §5-7이 그 사실과 이유를 적어 두었고, `DEVIATIONS.md` 머리에
  경고 줄도 있습니다. 새로 세지 않았습니다.

- **`MaxFragmentsPerMessage` 64가 방향 공용인 것, 나가는 상한이 올라가지 못하는 것,
  들어오는 상한을 낮추는 동사가 없는 것, 연결 종료가 보안/경로 표에 배선되지 않은 것.**
  전부 `ASYMMETRY-NOTES.md` §5-1·§5-4·§5-5·§5-6이 이미 결함으로 적어 둔 것이라
  새로 세지 않았습니다. (다만 §5-6의 "표에 남는 것은 연결당 정수 두 개입니다"는
  보안 연관 표에 대해서는 사실이 아닙니다 — 결점 1을 보십시오. 거기에는 키 재료와
  `AesGcm` 인스턴스가 남습니다.)

- **하네스의 두 끝점이 계수기를 공유하는 것.** `ASYMMETRY-NOTES.md` §5-8에 이미 있습니다.
