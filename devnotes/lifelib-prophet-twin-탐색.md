# lifelib × Prophet 쌍둥이 엔진 — 탐색 대화 정리

> 작성일: 2026-06-05
> 목적: lifelib 프로젝트를 탐색하며 "Prophet 동등 현금흐름엔진 + 신상품 모델링 자동화" 구상을 점검한 대화 기록.
> 다음 단계: 이 문서를 토대로 심도 있는 실행 플랜 수립.

---

## 0. 한 줄 요약

폐쇄망의 **Prophet(원장)** 은 그대로 두고, 인터넷 환경에 **Prophet과 값이 동일한 쌍둥이(twin) 현금흐름엔진**을 `modelx` 기반으로 구축한다. 신상품이 나오면 **기초서류를 LLM으로 파싱**해 이 엔진에서 모델링을 자동화하고, 거기서 도출된 **변경사항을 Prophet 규격으로 정리**해 사람이 Prophet에 수작업 반영 후 2차 검증한다. → **기술적으로 가능. 단, "리팩토링 금지·Prophet 동형 미러" 가 핵심 설계 원칙.**

---

## 1. lifelib 프로젝트 파악

- **lifelib** = Python 오픈소스 생명보험 계리모델 모음 (MIT, https://lifelib.io).
- 핵심 토대는 [`modelx`](https://modelx.io): 객체지향·셀 단위 모델링 패키지.
- `lifelib/libraries/` 아래 독립 라이브러리들:
  - 전통형: `simplelife`, `fastlife`, `basiclife`, `annuallife`
  - 중첩/확률론: `nestedlife`, `cluster`
  - 저축/변액: `savings`, `appliedlife`(변액연금 GMXB/GLWB)
  - 회계/규제: `ifrs17sim`, `ifrs17a`, `solvency2`, `economic`, `economic_curves`, `smithwilson`, `assets`

### 작업 브랜치 초점: `annuallife/TradLife_A`
- 연납 신계약 전통형(정기 TERM / 종신 WL / 양로 ENDW) 투영 모델.
- **modelx 구조** (폴더=Space, `__init__.py`의 함수=Cells=수식):

```
InputData ──┬─> PolicyAttrs ─┐
            ├─> Assumptions ─┤
            ├─> Economic ────┼─> Projection  (= BaseProj + PV 상속)
            └─> CommTable ───┘
```

- **InputData**: `input.xlsx`의 named range(PolicyData, ProductSpecTable, AssumptionTable, MortalityTables, Scenarios, ConstParams 등) → pandas 로드
- **PolicyAttrs / Assumptions**: 모델포인트별 속성·가정을 numpy 배열로 매핑 (`Utilities` 상속)
- **CommTable**: 커뮤테이션 함수(Dx, Nx, Cx, Mx, Ax, AnnDuenx…) → 영업보험료율
- **BaseProj**(~959줄): 기간별 현금흐름 투영
- **PV**: 현가 계산
- **Projection**: 자체 수식 없이 BaseProj+PV 상속, `idx`(모델포인트)·`scen_id`(시나리오)별 동적 실행
- 사용 예: `m = mx.read_model("TradLife_A"); m.Projection[0].pv_net_cf(0)`
- 순수 Python(nomx) export 지원.

**핵심 통찰: lifelib는 "계산 로직(코드)" 과 "회사·상품 데이터(input.xlsx)" 를 의도적으로 분리해 둠.** → 외부 입력을 갈아끼우는 자리가 자연스러운 통합 지점.

---

## 2. 구상과의 적합성 (1차)

### 잘 맞는 점
- 계산엔진 ↔ 파라미터 분리 구조가 이미 존재 → 파싱한 정보를 input으로 주입하는 그림이 자연스러움.
- modelx의 강점: **셀 단위 값 추적·캐싱·의존성 자동추적** → 검증에 유리.

### 자동화되는 것 vs 막히는 것
- **① 파라미터 성격**(위험률표 ID, 예정이율, 사업비율, 해지공제, 수수료율, 세율 등) → input 테이블에 그대로 매핑 (자동화 잘 됨)
- **② 방법론·공식 성격**(보험료/준비금 산출식) → modelx **Cells에 하드코딩**. 예: `gross_prem_rate()`가 상품유형별 `if TERM/WL … elif ENDW` 분기. → 회사 방식과 다르면 셀을 새로 작성해야 함.

### 한국 상품 갭
- lifelib 기본 상품군은 단순. 갱신형/재가입, 무·저해지, 유니버셜, 변액, 다수 특약, 금리연동형 등은 추가 모델링 필요.

---

## 3. 라이브러리 코드 리팩토링이 필요한가? → 아니오 (오히려 금지)

1. `__init__.py`는 손으로 관리하는 소스가 아니라 **modelx가 직렬화한 산출물**(`# modelx: pseudo-python`). 일반 코드식 DRY 리팩토링은 **라운드트립을 깨뜨림**.
2. 라이브러리 간 중복(`pols_if`가 15개 모델에 존재)은 버그가 아니라 **교육용 점진적 변형(standalone 예제)**.
3. 모델 내부 재사용은 이미 **Space 상속**으로 구현됨 (BaseProj+PV→Projection, Utilities→Assumptions/PolicyAttrs).
4. 성능 최적화 불필요: modelx가 **파라미터 기준 메모이즈** → 순진한 t-재귀도 1회만 계산. 손 최적화는 가독성만 해침.
5. 눈에 띄는 잔재: `TradLife_A` 옆 `TradLife_A_mx30`(구 modelx 0.3 포맷 사본) — 리팩토링 대상이 아니라 "어느 직렬화 버전 기준" 선택의 문제.

> 함의: 노력은 lifelib 안쪽이 아니라 **본인이 얹을 어댑터 레이어**(파싱→중간표현→input 빌더)에 투입.

---

## 4. 회사 모델과 "동일 값"이 가능한가? → 가능 (조건부)

- modelx 셀 = 편집 가능한 Python 공식 → 회사 로직대로 다시 쓰면 됨.
- 관점 전환: "lifelib 수정"이 아니라 **"reconciliation(일치화)"**, 그리고 **"lifelib 골격 위에 회사 공식을 얹기"**.
- 재사용 가치 = 프레임워크(modelx 엔진, Space 구조, 입력처리, PV·투영 스켈레톤). 구체적 공식은 대체 대상.

### 진짜 난관은 공식이 아니라 "관례(convention)"
- 연령(보험나이/만나이), 시점(기시/기말), 위험률 보간·조정 순서, **반올림/절사**, 준비금 방식, 사업비 부가, 납입면제·특약.
- lifelib 기본값 ≈ 교과서 관례라 회사와 거의 다름 → **이 매핑이 작업의 80%.**

### modelx가 유리한 이유
- 최종값만이 아니라 **모든 중간 셀값 대조** 가능 → "어느 시점·어느 항목에서 처음 갈라지는지" 추적해 수렴.

---

## 5. 실제 상황: Prophet 화이트박스 export

- 회사 모델 = **Prophet**(FIS), 로직을 **txt로 export**. **변수 ~3000개, 약 20MB.**
- 화이트박스 → "블랙박스 리버스 엔지니어링"이 아니라 **"번역(translation)"** 문제로 격하 (난이도 질적 하락).

### Prophet ↔ modelx 구조 대응 (거의 1:1)

| Prophet | modelx | 비고 |
|---|---|---|
| Variable (DEFINE 수식) | **Cell** | 둘 다 "이름=수식", t 재귀 기본 |
| 시간 인덱스 `t` (암묵 루프) | Cell `(t)` 파라미터 | `f(t)`가 `f(t-1)` 호출 동형 |
| Table / Parameter | **Reference / DataFrame** | InputData 자리 |
| Model Point 필드 | 입력 Cell / Reference | PolicyData |
| Product / Run structure | **Space / 상속** | TradLife_A Space와 동형 |
| Result variables | 최종 Cell | net_cf, pv_* |

> 두 시스템 패러다임이 같음(변수/셀 단위, t 재귀, lazy 평가, 의존성 자동추적) → 자동변환 친화적. (거대 Excel이었다면 더 어려웠을 케이스.)

### 일이 되는 지점
1. **규모**: 3000변수·20MB = 실제 프로덕션 엔진 → 손번역 비현실적, **반자동 트랜스파일러** 필요 (= 원래 목표 "자동화"와 동일 방향).
2. **Prophet 빌트인·평가 의미론 재현**: `POL_DUR`, 보간, 테이블 조회, `MIN/MAX/IF`, `GLOBAL`, indicator, t=0 처리, Run/시나리오 분기, 반올림 → **Python builtins shim 라이브러리로 한 번 정확히 구현**해 공유.
3. **변환 ≠ 일치**: 1차 트랜스파일 후에도 의미론 차이로 값 안 맞음 → **셀 단위 reconciliation** 필수.

---

## 6. 최종 목표 (사용자 확정)

1. **Prophet은 그대로** — 폐쇄망, LLM 불가(현재 Prophet도 LLM 미지원). = system-of-record.
2. 인터넷 환경에 **Prophet과 동일 값의 현금흐름엔진(twin)** 구축.
3. 신상품 → **기초서류 파싱** → twin 엔진에서 상품 모델링을 **자동화 수준**으로 완료.
4. 모델링으로 도출된 **변경사항(로직·속성)을 Prophet 규격으로 정리** → 사람이 Prophet에 **수작업 반영** → **2차 검증**.

> 평가: 폐쇄망·LLM 불가 제약을 우회하는 합리적 사이드카 설계. round-trip(엔진→명세→Prophet→재검증)으로 계리 책임 경계를 지킴.

---

## 7. 이 목표가 강제하는 설계 원칙

### (A) 재사용 대상 재정의
- **진짜 핵심 = `modelx`(엔진)** — Prophet과 패러다임 동일, 쌍둥이 토대로 이상적.
- **lifelib(라이브러리) = 참고서 + 스캐폴딩** — 작법/입력처리/PV·투영 골격. **공식은 안 씀.**
- 기대치: "lifelib 코드 상당부분 재사용"이 아니라 **"modelx 위에 Prophet 미러링 + lifelib 작법 차용"**.

### (B) ★ 리팩토링 금지, Prophet 동형(isomorphic) 미러 ★
- 역방향 round-trip(엔진 변경 → Prophet 수작업 반영) 때문에 **엔진 구조가 Prophet과 1:1**이어야 함.
- Prophet 변수 1개 = modelx 셀 1개, **같은 이름**, 같은 테이블 구조, 같은 분기.
- 공식을 합치거나 추상화하면 역번역이 사람이 대조 불가능해짐.
- **의도적으로 똑같이, 장황하게** 작성. **추적성(traceability) > 코드 미학.**

### (C) 양방향 정확성
- 정방향(Prophet→엔진): 번역 + 셀단위 reconciliation.
- 역방향(엔진→Prophet 명세): 동형이면 "변경된 셀 = 변경할 변수" diff로 기계적 도출.

---

## 8. 핵심 리스크

1. **쌍둥이 표류(drift)**: Prophet 업데이트 시 자동 동기화 안 됨 → **버전 페어링 + 정기 재검증** 규율 필수. 엔진을 Prophet 버전에 태깅하고 회귀검증 결합.
2. **의미론 충실도**: 값 차이는 공식보다 빌트인·관례(타이밍/보간/반올림/t=0/Run)에서 발생 → **builtins shim**으로 일원화.
3. **검증 정의**: "동일 값"의 허용오차·대상 출력·sign-off 기준을 상품별로 명문화. 2차 검증 매치 기준도 정의.
   - **[결정 2026-06-05] 허용오차 정책**: **bit 단위(부동소수점) 일치는 비목표.** Prophet(C++)과 Python은 연산순서·초월함수 구현이 달라 끝자리가 갈리는 게 정상. → **상대오차 + 중요성(materiality) 임계치** 기반으로 판정(예: 항목별 상대오차 < 1e-6, 절대금액이 무의미하게 작은 셀은 면제). 절대오차 단독은 금액 스케일 차이로 부적합. 출력 항목별 차등 기준 허용(보험료=빡빡, 먼 미래 CF=느슨).
4. **거버넌스/라이선스** (착수 전 확인):
   - FIS Prophet 로직 외부 재구현이 라이선스 약관에 저촉되는지.
   - 회사 계리로직·기초서류를 인터넷 환경 반출하는 게 정보보안/규제(기밀·개인정보)상 허용되는지.
   - 폐쇄망 사유가 보통 이 때문 → 사이드카를 인터넷에 두면 경계를 넘게 됨.

---

## 9. 자동화(LLM) 적용 경계

- **LLM 고가치**: 기초서류 파싱·구조화 / Prophet export 파싱·트랜스파일 초안 / 변경 diff를 Prophet 명세 문서로 정리.
- **사람 필수**: reconciliation 사인오프 / Prophet 수작업 반영 / 2차 검증. (계리 책임 영역)

---

## 10. 권장 아키텍처

```
[폐쇄망]  Prophet (원장)
   │ logic export (.txt, ~3000 vars, 20MB)
   ▼
[인터넷]  ① 파서: 변수명·타입·DEFINE·의존성 추출
   ▼
        중간표현 (AST / JSON)
   ▼
        ② 트랜스파일러: → modelx 셀 코드  (+ Prophet builtins shim)
   ▼
        쌍둥이 엔진 (modelx; lifelib 골격 차용, Prophet 동형 구조)
   ▼
        ③ reconciliation 하네스: Prophet 변수값과 셀단위 대조
   ▲────────────────────────────────────────────────┐
   │ 신상품: 기초서류 파싱(LLM) → 모델링 자동화        │
   ▼                                                  │
        변경사항 → "Prophet 변경 명세" 역번역          │
   ▼                                                  │
[폐쇄망]  사람이 Prophet에 수작업 반영 → 2차 검증 ──────┘
```

- 차용: 입력처리·PV·투영 골격, modelx 사용 패턴
- 신규 IP: Prophet 파서 + 트랜스파일러 + builtins 런타임 + reconciliation/round-trip 하네스

---

## 11. 성능 & GPU

- **GPU 지원: 없음.** 코드베이스에 `cupy/numba/torch/cuda/jax/tensorflow` 흔적 전무. modelx=순수 Python(CPU), 수치=numpy(CPU). 벡터화 셀의 numpy를 cupy로 바꾸는 식은 이론상 가능하나 modelx 오버헤드·Python 재귀가 GPU 이득을 막아 **실익 작음·비지원**.
- **modelx 성능 특성**: 셀 단위 메모이즈(재계산 회피, 단 의존성 추적 오버헤드), `f(t)→f(t-1)` 재귀(`setrecursionlimit` 필요), 모든 중간값 캐시 누적(대규모 런 메모리↑). → **투명성·증분재계산용 엔진**이지 raw throughput 엔진 아님. Prophet(C++·멀티스레드)보다 순수 속도는 느림.
- **lifelib 두 관용구**:
  - 모델포인트 **벡터화**(`BasicTerm_M/_ME`, `CashValue_ME`): 셀이 전체 모델포인트를 numpy 배열로 반환 → 호출 수가 건수와 무관, 대량에 빠름.
  - 모델포인트 **스칼라**(`BasicTerm_S`, `TradLife_A`의 `Projection[idx]`): 건당 스칼라, 읽기 쉽고 추적 명확, 대량은 느림.
- **성능 레버**(필요 시): 벡터화(_M 패턴) + `m.export()`(nomx, 오버헤드 제거) + `cluster`(모델포인트 압축). **GPU 아님.**
- ⚠️ **설계 충돌**: 벡터화(배열 반환)는 "Prophet 동형·변수당 스칼라·추적성 우선" 원칙과 부딪힘. → **reconciliation/round-trip 코어는 스칼라 동형**으로, 대량 런이 정말 필요할 때만 **별도 벡터화 변형(2-트랙)**. 단일 코드로 둘 다 만족 ✗.
- twin 목적(대표 모델포인트 reconciliation)에선 throughput 사실상 비이슈.

---

## 12. twin으로 Prophet 로직 최적화 확인 가능한가?

"병목/최적화"를 **두 종류로 분리**해야 함.

### (A) 로직·구조 최적화 — ✅ 가능 (엔진 무관, twin 강력)
실행엔진과 무관한 공식·의존성 구조 문제 → modelx의 **의존성 그래프 추적**(precedents/dependents) + `networkx`로 분석. LLM이 공식 추론 담당.
- **죽은 변수**(dependents 없음 — lifelib에도 "future use, not consumed" 사례 존재), **중복 로직/공통 부분식**, **불필요한 의존성**, **의존성 깊이·팬아웃 핫스팟**, **t-불변인데 매 t 재계산**, **대수적 단순화**.
- "Prophet 동형 미러" 덕분에 발견이 **Prophet 변수에 1:1 매핑** → 바로 actionable.

### (B) Prophet 런타임 성능 병목 — ⚠️ 직접 측정 신뢰 불가
twin wall-clock ≠ Prophet 프로파일(C++·멀티스레드 vs Python·재귀·메모이즈). twin을 cProfile로 재면 modelx 오버헤드를 재는 것.
- 엔진 무관하게 무거운 패턴(중첩 루프, 확률론 곱, `O(t²)` 누적, 거대 테이블 조회)은 **구조로 추정 가능**, 보통 Prophet에서도 무거움.
- **실제 런타임 프로파일은 Prophet 자체 도구(폐쇄망)** 로. twin은 지도이지 스톱워치 아님.

### LLM 역할 + 검증 루프
입력(공식 텍스트 + 그래프 지표) → LLM이 단순화/통합/죽은가지 **수정안 생성** → **twin에 적용해 원본 출력/골든값과 reconcile(원단위 일치)** → 통과분만 **Prophet 변경명세**로. (= round-trip 재사용)
- 규율: 발견은 **별도 "최적화 제안"으로 기록**, mirror는 그 자리에서 리팩토링 ✗(동형성 보존).

---

## 13. 하네스 / 오케스트레이션 / 서브에이전트 구성

### 대원칙
안전성은 에이전트 수가 아니라 **단계 사이 검증 게이트(reconciliation)** 에서 나옴.
> **검증 게이트 없이 LLM 단계를 연쇄하지 말 것.** 파서·트랜스파일러·reconciliation = 결정론 코드. LLM/에이전트 = "모호한 판단"에만. 각 에이전트 출력은 **기계적으로 검증 가능**(등가 reconcile + 구조화 스키마).

### 두 레이어 분리
- **빌드타임(1회성)**: Prophet 3000변수 → twin 구축(파서·트랜스파일·shim·초기 reconcile). 한 번 제대로.
- **런타임(반복)**: 신상품마다 기초서류 파싱 → 모델링 → reconcile → Prophet 변경명세. ← 반복 오케스트레이션 투자 대상.

### 서브에이전트 분해 (이음새 = 에이전트 경계)

| 단계 | 형태 | 분리 이유 / 컨텍스트 | 검증 게이트 |
|---|---|---|---|
| Prophet export 파싱 | 결정론 코드 (에이전트 ✗) | 20MB → 모듈/상품별 청크 fan-out | 파싱 라운드트립(재출력=원본) |
| DEF→modelx 트랜스파일 | 결정론 + LLM 폴백 | 까다로운 구문만 서브에이전트 | 컴파일/문법 통과 |
| builtins shim 구현 | 사람+LLM 코딩(런타임 ✗) | 한 번 만들어 공유 | 단위테스트 |
| **reconciliation** | 결정론 하네스 + 진단 에이전트 | 모델포인트별 fan-out, 첫 divergence 추적 | 원단위 일치 |
| 기초서류 파싱 | 에이전트(LLM 강점) | 문서이해·추출, 상품별 격리 | 사람 검수 + 스키마 |
| 상품 모델링 자동화 | 에이전트 | 추출 스펙→엔진 변경 매핑 | reconcile + 계리 sign-off |
| Prophet 변경명세 역번역 | 에이전트 + 결정론 diff | 동형성 → 셀diff→변수diff | 사람 반영 후 2차검증 |
| 최적화 분석(§12) | 에이전트 | 그래프+공식 추론 | twin 등가성 증명 |

### 오케스트레이션 패턴
- **상위 오케스트레이터**: 순차 실행 + **게이트 통과 시에만 다음**. 실패 시 사람 호출. 계리 sign-off·Prophet 수작업 반영은 **무조건 human-in-the-loop**.
- **Fan-out(병렬) 이득**: ① 20MB 파싱(모듈/상품 청크), ② reconciliation(모델포인트별), ③ 신상품 동시 다건. — 컨텍스트 윈도우 한계상 **상품/모듈 단위 분할은 필연**.
- **순차 유지**: 트랜스파일→reconcile→명세(의존 강함, 병렬 무의미).

### 안티패턴
- ❌ 코어가 1상품에서도 안 도는데 멀티에이전트부터.
- ❌ LLM 단계 2개를 검증 없이 연결(오류 누적).
- ❌ 에이전트가 mirror를 그 자리에서 "개선"(동형성 깨짐).
- ❌ 에이전트 출력이 자유텍스트 → 반드시 구조화 산출물(스키마).

### 단계적 도입
1. **PoC: 1상품·단일스레드·순차** — 파서→트랜스파일→reconcile 손으로 한 바퀴, 오케스트레이션 없음.
2. round-trip 닫히면 → reconciliation·파싱을 fan-out 서브에이전트로.
3. 신상품 파이프라인 가치 입증 → 상위 오케스트레이터 + 게이트 정식화.

---

## 14. 다음 단계 — PoC 플랜 (탐색→파일럿)

1. **파일럿 1상품 선정**: 가장 단순한 정기보험류. 전체 3000변수 말고 의존 서브셋만.
2. **Prophet export 파서**: 변수·타입·DEFINE·의존성 추출 → 의존 그래프 시각화.
3. **builtins shim 최소셋** 구현 + 서브셋 트랜스파일 → modelx 셀.
4. **reconciliation**: 동일 모델포인트로 셀단위 대조, 원단위 일치까지.
5. **round-trip 닫힘 확인**: "엔진 변경 → Prophet 명세" 역번역을 끝까지 1회 — **이게 PoC 진짜 성공 기준.**

### 다음 대화에서 정할 것 (Open Questions)
- [ ] 파일럿 대상 상품/Run 선정
- [ ] Prophet export 포맷 실제 샘플 확인 → 트랜스파일 난이도·견적
- [ ] builtins/관례 목록 1차 인벤토리 (타이밍·보간·반올림·Run 분기…)
- [x] reconciliation 허용오차: **bit 일치 비목표, 상대오차+materiality 기반** (§8-3). 항목별 임계치·sign-off 기준은 PoC에서 구체화.
- [ ] 거버넌스/라이선스 사전 확인 결과
- [ ] 엔진 버전 ↔ Prophet 버전 페어링·회귀검증 운영 방식

### 가장 값진 한 걸음
**Prophet export txt의 변수 정의 샘플 1~2개**(민감정보 제거)를 확보하면:
① modelx 셀 변환이 얼마나 기계적인지, ② 어떤 빌트인/관례를 shim으로 분리해야 하는지를 실제 코드로 매핑 가능 → 트랜스파일러 견적·PoC 범위 확정.

---

## 15. ADR — 코어 엔진: modelx 차용 vs 백지 자작

- **상태**: 제안(Proposed) — PoC에서 Prophet 샘플 검증 후 확정(Accepted) 예정
- **날짜**: 2026-06-05

### 맥락
"lifelib 의존 안 하고 코어엔진을 백지부터 만들까?"의 결정. 용어를 3층위로 분리해야 함:
1. **lifelib 라이브러리(예제 모델)** — 이미 비의존 결정(참고/스캐폴딩만, 공식은 Prophet 출처).
2. **modelx(엔진)** — 셀 그래프, lazy 평가, 의존성 자동추적, 메모이즈, 직렬화. ← **실제 쟁점.**
3. **계산 엔진까지 백지 자작.**
→ 진짜 질문 = "**modelx 차용 vs 엔진 자작**".

### 고려한 선택지
- **A. 백지 자작**: 의미론 완전 통제(Prophet 동형을 설계로 보장), lock-in 회피, 트랜스파일 타겟 소유. **단점**: 의존성그래프·lazy·메모이즈·순환검출·캐시무효화를 정확히 재구현 = 별도 프로젝트, 버그 미묘 → **버그 엔진은 "Prophet 동일" 보장 자체를 붕괴**시킴.
- **B. modelx 완전 종속**: 빠른 PoC, 검증된 도메인 정합(변수=셀·t재귀·lazy가 Prophet과 동형, 우연 아님), §12·§13에 필요한 의존성그래프를 공짜로. **단점**: 단일 메인테이너 bus factor, 패러다임 임피던스.
- **C. 하이브리드(채택)**: 엔진은 빌리되 결혼하지 않음.

### 결정 — C. 하이브리드 ("엔진은 빌려 쓰되 결혼하진 마라")
```
Prophet export → [내 IR/AST: 소유] → 백엔드: modelx 코드 생성 (지금)
                       │                         ↘ (나중) 자작 엔진 백엔드로 교체 가능
                 builtins shim(엔진무관) · reconciliation(엔진무관)
```
- **소유(IP)**: 파서 + **IR** + builtins 런타임 + reconciliation 하네스 (엔진 무관 설계).
- **빌림**: modelx를 1차 백엔드, 버전 핀 고정.
- **교체 시드**: IR를 엔진 중립으로 → 구체적 한계 입증 시에만 자작 백엔드 추가.

### 근거
- modelx **lock-in이 약함**: 모델을 읽기 쉬운 Python 텍스트로 직렬화 → **자산(공식)은 이식 가능 텍스트**, 의존은 런타임뿐.
- 두 극단(거대 자작=엔진 재발명 / 완전 종속) 동시 회피.
- 신뢰성이 twin의 존재 이유 → 검증된 엔진 위에서 시작이 안전.

### 자작(A) 재검토 트리거
- PoC에서 **Prophet 구문이 modelx 셀로 깨끗이 안 맞음**이 입증될 때(다차원 배열, 특정 평가순서, GLOBAL/structure 스코핑 충돌 등).
- 제로 외부 런타임 의존이 요구되는 장기 제품화.
- (GPU/대규모 병렬은 §11에서 불필요 결론.)

### 선행조건
**Prophet export 실제 샘플 → modelx 매핑 검증**이 이 ADR 확정의 전제(§14 "가장 값진 한 걸음"과 동일).

---

## 16. 가정검증 실측 결과 (2026-06-05, modelx 0.31.1 / TradLife_A)

이 환경에서 modelx+deps 설치 후 `annuallife/TradLife_A`로 §11·§12·§13·§15 가정을 실측. **모두 가정대로 확인됨.**

### 환경
- `pip install modelx pandas openpyxl networkx` → modelx **0.31.1**, pandas 3.0.3, networkx 3.6.1. (modelx가 pandas를 강제의존하지 않아 별도 설치 필요.)
- modelx/lifelib 둘 다 순수 Python으로 설치·실행됨(컴파일·GPU 불필요).

### 결과
| 가정 | 결과 | 비고 |
|---|---|---|
| 모델 읽기 | ✅ 0.5s | `read_model("TradLife_A")` |
| 계산 + **메모이즈** | ✅ | `pv_net_cf(0)=8954.0183`; cold 0.35s → **cached 46µs**(≈7500배) |
| **의존성 그래프 API** (§12·§13) | ✅ | `node.preds/succs`(런타임, 값캐시 필요) + `node.precedents`(정적) |
| **networkx 추출** | ✅ | pv_net_cf(0) 도달 노드 **1,813 / 엣지 3,404, DAG=True**(순환 없음) |
| **fan-out 핫스팟 분석** | ✅ | `disc()` 161곳, `mortality_rates()` 110곳 참조 → 최적화 핫스팟 식별 실현성 입증 |
| **직렬화 라운드트립** (§15) | ✅ | `write_model→read_model` 후 값 **완전 동일**(`identical=True`). 단 `input.xlsx`(데이터)는 모델 외부라 동반 필요 |
| **nomx export** (§11·§15) | ✅ | 순수 Python 패키지로 export, **modelx 없이 실행해도 값 동일** → 이식성·lock-in 약함 실증 |
| **성능 감** (§11) | ✅ | 300 MP **스칼라 방식** cold 런 **20.0s = 66.8ms/MP** |

### 함의 (계획 업데이트)
- **§12·§13 핵심 가정 입증**: 의존성 그래프가 실제로 쿼리·networkx화 가능 → 최적화 분석·셀단위 reconciliation 토대 확실. (`preds`는 **값이 캐시된 뒤**라야 조회됨 = 런타임 실제 의존; 정적 의존은 `precedents`.)
- **§15 ADR 보강**: 텍스트 직렬화 라운드트립 무손실 + nomx 이식성 확인 → "엔진은 빌리되 lock-in 약함" 실증. **modelx 차용으로 PoC 시작 결정의 신뢰도↑.**
- **§11 성능 확증**: 스칼라 방식 ~67ms/MP는 twin PoC(대표 MP 수십~수백 건)엔 충분. **대량(수만~수십만 MP)이면 벡터화(_M)·nomx·cluster 필요** — GPU 아님.
- **주의(검증 절차)**: 직렬화/nomx 모델은 `input.xlsx`를 모델 경로 기준으로 읽음 → reconciliation 하네스에서 **데이터 파일 동반·경로 관리**를 절차에 포함해야 함.
