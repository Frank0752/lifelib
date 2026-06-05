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

## 11. 다음 단계 — PoC 플랜 (탐색→파일럿)

1. **파일럿 1상품 선정**: 가장 단순한 정기보험류. 전체 3000변수 말고 의존 서브셋만.
2. **Prophet export 파서**: 변수·타입·DEFINE·의존성 추출 → 의존 그래프 시각화.
3. **builtins shim 최소셋** 구현 + 서브셋 트랜스파일 → modelx 셀.
4. **reconciliation**: 동일 모델포인트로 셀단위 대조, 원단위 일치까지.
5. **round-trip 닫힘 확인**: "엔진 변경 → Prophet 명세" 역번역을 끝까지 1회 — **이게 PoC 진짜 성공 기준.**

### 다음 대화에서 정할 것 (Open Questions)
- [ ] 파일럿 대상 상품/Run 선정
- [ ] Prophet export 포맷 실제 샘플 확인 → 트랜스파일 난이도·견적
- [ ] builtins/관례 목록 1차 인벤토리 (타이밍·보간·반올림·Run 분기…)
- [ ] reconciliation 허용오차 및 sign-off 기준 정의
- [ ] 거버넌스/라이선스 사전 확인 결과
- [ ] 엔진 버전 ↔ Prophet 버전 페어링·회귀검증 운영 방식

### 가장 값진 한 걸음
**Prophet export txt의 변수 정의 샘플 1~2개**(민감정보 제거)를 확보하면:
① modelx 셀 변환이 얼마나 기계적인지, ② 어떤 빌트인/관례를 shim으로 분리해야 하는지를 실제 코드로 매핑 가능 → 트랜스파일러 견적·PoC 범위 확정.
