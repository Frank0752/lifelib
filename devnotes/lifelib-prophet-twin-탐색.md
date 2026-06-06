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

---

## 17. 트랜스파일 타겟 포맷 해부 (2026-06-05)

modelx 직렬화 포맷을 트랜스파일러 출력 타겟으로 쓸 수 있는지 실측 해부. **결론: 매우 단순·규칙적인 타겟. 저위험.**

### 디스크 포맷 구조
- **모델 = 디렉토리**, **스페이스 = 하위 디렉토리 + `__init__.py`**, 자식 스페이스 = 중첩 디렉토리.
- `_system.json`: `{"modelx_version":[0,30,1], "serializer_version":7}` — **버전 스탬프**(§15 버전 핀과 연결, 트랜스파일러는 고정 버전 타겟).
- 데이터(DataFrame 등)는 스페이스의 `_data/`에 피클로 저장(있을 때만). **TradLife_A는 런타임에 `input.xlsx`를 읽어 `_data` 없음.**

### 스페이스 `__init__.py` 스키마 (고정 틀)
```python
"""docstring → 스페이스 문서"""
from modelx.serialize.jsonvalues import *
_formula = lambda idx, scen_id=1: None   # 파라미터화(없으면 None)
_bases = [".BaseProj", ".PV"]            # 상속(.=형제 스페이스 경로)
_allow_none = None
_spaces = []                             # 자식 스페이스명
# --- Cells ---
def cell_name(t):                        # 셀 = 평범한 def(=수식). docstring 선택
    return ...
# --- References ---
scalar_ref = 0.05                        # 스칼라 참조
M = 1                                    # (enum 멤버도 동일)
input_data = ("Interface", ("..","InputData"), "auto")   # 스페이스→스페이스
pd = ("Module", "pandas")                                # 모듈
ProductID = ("Interface", (".","Enums","ProductID"), "None")  # 자식 인터페이스
```
→ 즉 **셀 1개 = `def` 1개**. Prophet 변수 DEFINE이 그대로 여기로 떨어짐.

### 두 가지 출력 경로 — 둘 다 실측 검증됨
- **(A) 텍스트 파일 직접 생성**: 위 스키마는 사실상 템플릿(헤더 상수 + `def` 블록 + 참조 라인). 기계적.
- **(B) modelx API 구동 후 `write_model`** ⭐ 권장:
  `new_model → new_space(bases=[...]) → new_cells(name, formula="def f(t): ...") → 스칼라/Interface 참조 주입 → space.formula로 파라미터화`.
  - 실측: 문자열 수식(재귀 포함)·상속·파라미터화·스페이스간 참조 모두 조립 성공 → `write_model`→`read_model` **값 완전 동일**.
  - 장점: modelx가 **수식 검증·의존성 그래프 구성을 공짜로** 해줌, 텍스트 따옴표/이스케이프 함정 회피.

### 트랜스파일러 설계 함의 (→ §15 IR 백엔드)
- **IR → 백엔드(B: API 구동)** 가 정석. (A 텍스트는 디버그/검수용 뷰.)
- 매핑 이음새:
  | Prophet | modelx |
  |---|---|
  | 변수 DEFINE | `new_cells(formula="def 변수(t): ...")` |
  | t 루프 | 셀 `(t)` 파라미터 + 재귀 (네이티브) |
  | GLOBAL/파라미터 | 스칼라 참조 또는 Globals 스페이스 |
  | 테이블 | `_data` DataFrame 또는 input 읽는 셀 |
  | 상품/structure | 스페이스 + 상속(`_bases`) |
  | 빌트인 함수 | **shim 라이브러리를 모듈/함수 참조로 주입** |
- **이름 보존**: Prophet 변수명을 셀명으로 그대로 → §7 동형 미러 직접 지원.
- ⚠️ **주의**: modelx 셀명은 **유효한 Python 식별자**여야 함. Prophet 변수명에 비식별자 문자가 있으면 sanitize 필요 + **이름맵 보관**(§13 역번역용).

---

## 18. 기초서류 파싱 설계 (검토 2026-06-05)

프로젝트의 "나머지 절반". 엔진 경로와 **검증 구조가 근본적으로 다름**(아래 §18.3).

### 18.0 전제 (사용자 확인)
- **문서 포맷: HWP(한글)** — 독점포맷, 표·수식 임베드. 변환 파이프라인 필요.
- **신상품 성격: 반반** — 기존 변형 + 신규구조 혼재 → diff/extract **하이브리드**.
- **요율 소재: 혼재** — ★**위험률은 별도 Excel/DB로 사용자가 제공**, 기초서류엔 **위험률 *명*(참조키)만** 존재(애초에 율 없음). 예정사업비·확정(예정)이율 등은 **문서 본문에 값으로** 있음.
  - → **함의: 가장 어려운 "대량 율표 추출"이 과제에서 빠짐.** 엔진의 기존 패턴(lifelib: MortTable**ID**→MortalityTables 조회)과 동일하게, IR엔 "참조키"만 담고 율표는 엔진 InputData에 별도 주입.

### 18.1 기초서류 3종 → 엔진 기여
| 문서 | 엔진에 주는 것 |
|---|---|
| 사업방법서 | 모델포인트 속성 구조, 상품 분류/세대 |
| 보험약관 | **현금흐름 트리거**(지급사유·면책·해지), 옵션 구조 |
| 산출방법서 | **공식·방법론 + 본문 파라미터**(예정이율·사업비·해지공제) + 위험률 *참조키* |
→ §2의 "파라미터 vs 방법론" 구분 그대로 적용.

### 18.2 파싱 대상 (좁혀짐)
- (a) **참조키**(위험률명 등): 추출 → **외부 제공 율표 키와 매칭**. ← 신상품에도 적용되는 **강한 게이트**(불일치=즉시 에러).
- (b) **본문 스칼라 파라미터**(예정이율·예정사업비·해지공제 등): 값 + provenance 추출. 규칙검증(사업비 합·해지공제 단조 등).
- (c) **방법론/구조**(지급사유→트리거, 산출방식): LLM 제안 + **계리 검수**.

### 18.3 ★ 핵심: 신상품엔 Prophet 골든값이 없다
엔진 경로(§16)는 기존상품이라 Prophet과 reconcile하는 강한 게이트가 있었음. **신상품은 정의상 아직 Prophet 모델이 없음** → 같은 검증 불가.
- 첫 강한 검증이 **맨 끝(Prophet 2차검증)** 에 있음 → 앞단 오류가 늦게 발각.
- 대응: **앞단 sanity 검증 front-load**(스키마·문서간 일관성·합리성) + **twin = 빠른 반복 검토장**(모델→즉시 현금흐름 확인·이상치, Prophet이 폐쇄·느린 걸 보완) + (드문 강한 게이트) **율키 매칭**.

### 18.4 ★ 전략: extract-all 말고 analogy-diff 우선 (반반이므로 혼용)
- 새 상품 → 먼저 **유사 기존상품 매칭**.
- 유사 있음 → **diff 모드**(변경점만 추출·확인, 고신뢰, §13 change-spec 철학과 일치).
- 신규구조 → **extract 모드**(백지 추출, 계리검수·twin sanity 강화).
- 한 상품 안에서도 **컴포넌트 단위로 혼용**(대부분 변형 + 일부 새 옵션).

### 18.5 HWP 처리 전략 — kordoc 채택 검토 (업데이트 2026-06-05)

**도구: [chrisryugj/kordoc](https://github.com/chrisryugj/kordoc)** (사용자 발견, 산출방법서 1건 파싱 성공 — 정확성 미검증).
- MIT, **Node.js 18+**(TS). HWP3·HWP5·HWPX·HWPML·PDF·XLS·XLSX·DOCX **→ Markdown** 통합. 병합셀·누락선 표 복원. library/CLI/MCP 3인터페이스.
- **코어 파싱 완전 로컬·오프라인**(필수 외부 API 없음). 의존성 전부 로컬(`pdfjs-dist`/`cfb`=HWP5 OLE/`JSZip`=HWPX/`rhwp` 내부포트). **한글 설치 불필요.**

**함의 (설계 변경):**
- ★ **HWP 직접 파싱** → 기존 "HWP→HWPX 변환 1단계" **불필요해질 수 있음.**
- ★ 모든 포맷 → **Markdown 정규화** → 포맷 이질성 해소, 파서 뒷단을 단일 입력으로.
- ★ **폐쇄망·인터넷 동일 도구** 사용 가능(코어 로컬·MIT).

**Caveat (폐쇄망/거버넌스):**
1. **오프라인 설치**: npm 설치는 인터넷 필요 → `npm pack`/오프라인 미러로 패키지+node_modules 반입(설치 후 오프라인 실행).
2. **OCR은 선택·외부연동 가능** → 캡처 이미지 수식이면 **로컬 OCR 연결 또는 경로 제외**.
3. **MCP(LLM연동) 인터페이스는 폐쇄망 미사용** → library/CLI만(파싱 자체 LLM 불필요).
4. **텔레메트리 무전송 명시 없음** → 반입 전 **코드/네트워크 감사**(아웃바운드 호출 없음 확인).
5. **정확성 미검증 + 신생·1인 저자** → 파이프라인 최전단이라 오류 전파 → **알려진 문서로 정확성 벤치마크** 필수.

**출력 형태 주의**: kordoc 출력은 **Markdown**(rich XML AST 아님). 표·텍스트는 양호하나 **provenance(페이지·셀 좌표)** 메타는 약할 수 있음 → "kordoc(MD) → 구조화 추출(규칙+LLM) → product-spec IR" 단계로 두고 provenance 보존도 검증.

**보조/대안**: `pyhwp`(HWP5, Python), `rhwp`/`rhwp-python`(Rust+PyO3, HWP/HWPX), `openhwp` — kordoc 정확성·유지보수 리스크 대비 백업 후보.

### 18.6 product-spec IR (엔진 IR과 연결되는 상품 레이어)
목표 = "문서→완성모델" 자율화가 **아니라** "문서→**사람 검토 가능한 구조화 스펙**".
- 필드: 분류키(상품/PolType/Gen), 위험률 **참조키**, 예정이율·예정사업비·해지공제(본문값), 납입/보장기간 구조, 지급 트리거, 보험료식·준비금방식 식별.
- 각 필드 메타: `value + source(문서/외부율/diff기준상품) + provenance(페이지·표) + confidence + needs_review`.
- diff 메타: 기준상품 + 변경필드 목록.

### 18.7 파이프라인/서브에이전트 (§13 게이트 적용)
1. kordoc: HWP/HWPX/PDF → **Markdown** 정규화 (결정론, 로컬·오프라인)
2. 구조 추출: Markdown의 표/문단/수식 → 구조화 (규칙 + LLM)
3. 상품 분류 · 유사상품 매칭 (검색/에이전트)
4. diff/extract 추출 — **컴포넌트별 fan-out** (LLM 에이전트)
5. 소스 resolution(문서값 + 외부 율키 + diff기준) → product-spec IR
6. 검증 게이트: 스키마 · **율키 매칭(강)** · 문서간 일관성 · 산술 sanity · **사람 검수**
7. → (다음) IR→엔진 모델 변경(§7) → twin sanity → Prophet 변경명세(§13)

### 18.8 검증 도구 분담
- 표/숫자: 가능하면 LLM 말고 **결정론 표 파서** + 규칙검증. (LLM이 표 숫자 "읽어 옮기기"를 검증 없이 신뢰 ❌)
- 서술→의미: LLM 강점(약관 지급사유→트리거, 산문 공식→수식 후보).

### 18.9 Open Questions
- [~] HWP 처리: **kordoc 채택 후보**(§18.5). 남은 확인 → 정확성 벤치마크, 오프라인 반입(npm vendoring), 텔레메트리 감사, 수식이 객체/이미지인지(OCR 필요 여부), Markdown provenance 보존도.
- [ ] 외부 위험률 Excel/DB의 키 체계 = 문서의 위험률명과 어떻게 매칭되나(명명 규칙)?
- [ ] 유사상품 매칭 기준(상품유형·구조 시그니처) 정의
- [ ] product-spec IR 스키마 v0 확정 (엔진 IR과의 접합면)
- [ ] 본문 파라미터(예정이율·사업비)의 표기 일관성(상품마다 다른가)
