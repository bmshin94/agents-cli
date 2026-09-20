# `agents-cli` 전수조사 분석 & 활용/수익화 정리 (한국어)

> 작성: 카리나 (Claude Code) 💖 · 작성일: 2026-09-20
> 분석 대상 커밋: `26a6520` · 분석 버전: **v1.6.1** · 분석 파일 수: **459개**

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/agents-cli |
| **원본 저장소 (Google 공식)** | https://github.com/google/agents-cli |
| 공식 문서 사이트 | https://google.github.io/agents-cli/ |
| PyPI 패키지 | https://pypi.org/project/google-agents-cli/ |
| 릴리스 노트 | https://github.com/google/agents-cli/blob/main/RELEASE_NOTES.md |
| 이슈 트래커 | https://github.com/google/agents-cli/issues |
| 데모 영상 | https://youtu.be/ECYKo70pPNc |
| ADK (Agent Development Kit) | https://adk.dev |
| ADK Go 예제 | https://github.com/google/adk-go/tree/main/examples |
| 스킬 설치 도구 (`npx skills`) | https://github.com/vercel-labs/skills |
| 확장 스키마 | https://raw.githubusercontent.com/google/agents-cli/main/schemas/agents-cli-extension-v1alpha1.schema.json |

---

## 1. 이게 뭐야? (정체)

> **"AI 에이전트를 만들고 → 평가하고 → Google Cloud에 배포하는 전 과정을,
> 코딩 AI(Claude Code / Antigravity / Codex)가 대신 수행하도록 만드는 CLI + 스킬 패키지"**

- 라이선스: **Apache-2.0** (상업적 이용 가능)
- 저작권: `Copyright 2026 Google LLC`
- **코딩 에이전트의 경쟁자가 아님.** README FAQ 원문:
  > *"`agents-cli` is a tool **for** coding agents, not a coding agent itself."*
- ADK는 "프레임워크", `agents-cli`는 그 주변 전부(스캐폴딩·평가·배포·관측)를 담당.

### 폴더 구조 요약

| 경로 | 역할 |
|---|---|
| `src/google/agents/cli/` | CLI 본체 (Python, Click 기반) |
| `skills/` | 코딩 AI에 주입하는 지식 7종 (`SKILL.md` + `references/`) |
| `src/.../skills/data/` | 위 스킬의 번들 복사본 (wheel 패키징 → 네트워크 없이 설치) |
| `src/.../scaffold/` | 프로젝트 템플릿 (Python / Go / Java / TypeScript) |
| `extensions/langchain/` | 확장 시스템 레퍼런스 구현 (LangChain) |
| `docs/` | MkDocs 공식 문서 사이트 |
| `schemas/` | 확장 manifest JSON 스키마 (v1alpha1) |
| `plugin.json`, `.claude-plugin/plugin.json`, `gemini-extension.json` | 플러그인 매니페스트 3종 |
| `CLAUDE.md` | 페르소나(카리나) 설정 |

### 스킬 7종

| 스킬 | 코딩 AI가 배우는 것 |
|---|---|
| `google-agents-cli-workflow` | 총괄 지휘관 — 0~7단계 라이프사이클, 코드 보존 규칙, 모델 선택 |
| `google-agents-cli-adk-code` | ADK API (Python/Go) — 에이전트, 툴, 오케스트레이션, 콜백, 상태 |
| `google-agents-cli-scaffold` | 프로젝트 스캐폴딩 (`create` / `enhance` / `upgrade`) |
| `google-agents-cli-eval` | 평가 — 데이터셋, 메트릭, LLM-as-judge, 실패 분석 |
| `google-agents-cli-deploy` | 배포 — Agent Runtime / Cloud Run / GKE / CI·CD / 시크릿 |
| `google-agents-cli-publish` | Gemini Enterprise 등록 |
| `google-agents-cli-observability` | Cloud Trace, 로깅, BigQuery 분석, 서드파티 연동 |

핵심 설계: **스킬끼리 서로를 호출**하고, *"각 단계 시작 전에 다시 읽어라"* 를 명시
(컨텍스트 압축 대비) → **프로그레시브 디스클로저** 패턴.

### CLI 명령어 (`src/google/agents/cli/main.py`, lazy-load 17개)

```
설치   setup, update, login
생성   create, scaffold (enhance / upgrade)
개발   run, playground, install, lint, build
평가   eval (run / generate / grade / compare / analyze / optimize / dataset synthesize / metric)
배포   deploy, publish, infra (single-project / cicd / show / datastore)
확장   extension (add / list / remove / update)
정보   info
```

### 스캐폴딩이 자동 생성해주는 것

- 4개 언어(Python / Go / Java / TypeScript) × 배포 타겟 4종(Agent Runtime / Cloud Run / GKE / none)
- `deployment/terraform/` — IAM, 스토리지, WIF, 빌드 트리거 등 인프라 코드
- `.github/workflows/` + `.cloudbuild/` — PR 체크 / 스테이징 / 프로덕션 파이프라인
- `tests/eval/`, `tests/integration/`, `tests/load_test/`
- `Dockerfile`, `.env.example`, BigQuery 스키마, OTLP 텔레메트리, `AGENTS.md`

> 사람이 손으로 짜면 2~3주 걸릴 보일러플레이트를 명령 한 줄로 생성.

### 평가(Eval) — Quality Flywheel 4단계

```
1. 데이터 준비 → eval dataset synthesize  (가상 유저로 멀티턴 대화 자동 생성)
2. 평가 실행   → eval run (= generate + grade), HTML/JSON 리포트 생성
3. 실패 분석   → eval analyze  (LLM이 실패 케이스 클러스터링)
4. 최적화      → eval optimize (ADK GEPA 프롬프트 자동 튜닝)
```

> ⚠️ `eval optimize`(GEPA)는 LLM 호출이 매우 많아 비용/시간이 크다 — 스킬에도
> "사용자가 명시적으로 요청하기 전엔 실행 금지" 경고가 있음.

### 확장 시스템 (v1.5.0+)

`agents-cli-extension.yaml` 하나로 CLI 명령어를 오버라이드/추가 → **포크 없이** 다른
프레임워크나 사내 정책을 얹을 수 있음. GitHub / 사내 git / 로컬 경로에서 설치 가능.

---

## 2. 나(사용자)한테 무슨 도움이 되나

**장점**
1. 에이전트 개발의 "정석 루트"를 통째로 흡수 (스캐폴드 → 평가 → 배포)
2. 스킬 설치 시 코딩 AI가 GCP 에이전트 배포에 능숙해짐
3. 대부분 개발자가 건너뛰는 **평가 체계**를 코드+문서로 학습 가능
4. Terraform / CI-CD 템플릿을 다른 프로젝트에 재활용 (Apache-2.0)

**한계**
- Google Cloud에 강하게 종속 (Vertex AI / Agent Runtime / Gemini Enterprise 전제)
- 배포부터는 GCP 프로젝트 + 결제 필요
- Python 중심 (Go는 1.6.1에서 Preview로 공개, Java/TS는 템플릿 수준)
- 멀티클라우드 미지원, 실시간 음성/영상 미지원

---

## 3. 질문 정밀 답변

### 3-1. 설치 및 사용법

```bash
# 설치 (택1)
uvx google-agents-cli setup          # 권장: CLI + 스킬 한 번에
npx skills add google/agents-cli     # 스킬만
pipx install google-agents-cli && agents-cli setup
uv tool install google-agents-cli    # 영구 설치 (스킬 권장 방식)
```

전제조건: **Python 3.11+**, **uv**, **Node.js**
플랫폼: macOS / Linux / Windows(WSL 2) — 네이티브 Windows 공식 미지원

```bash
agents-cli login                 # 또는 export GEMINI_API_KEY="..."
agents-cli create my-agent       # 프로젝트 생성
agents-cli install               # 의존성 설치
agents-cli playground            # 로컬 웹 UI
agents-cli run "테스트 프롬프트"  # 단발 실행
agents-cli eval run              # 평가 → artifacts/grade_results/*.html
agents-cli infra single-project  # 인프라 준비
agents-cli deploy                # 배포
agents-cli info                  # 설정/버전 확인
```

### 3-2. 플러그인? 스킬? MCP?

**정답: "스킬"이 본체, "플러그인"으로 포장됨. MCP는 아님.**

| 구분 | 정체 | agents-cli |
|---|---|---|
| Skill | AI에게 주입하는 지식/절차 문서(Markdown) | ✅ **본체 (7개)** |
| Plugin | 스킬·명령어·훅을 묶은 배포 패키지 | ✅ 포장지 (매니페스트 3종) |
| MCP | AI가 외부 시스템에 접속하는 서버 프로토콜 | ❌ 아님 |

소스 전체 검색 결과 **MCP 서버 구현 코드는 0개**. MCP 언급은
`references/adk-python.md` 안의 "ADK 에이전트가 MCP 툴을 쓸 수 있다"는 설명뿐.

동작 방식: 서버를 띄우지 않고, **문서를 읽고 → 터미널에서 CLI를 실행**하는 구조.

### 3-3. API 토큰이 필요한가

`src/google/agents/cli/auth.py` 기준 인증 3종:

| 방식 | 환경변수 | 용도 | 결제 |
|---|---|---|---|
| Gemini API Key (AI Studio) | `GEMINI_API_KEY` | 로컬 개발/실험 | 무료 티어 가능 |
| Express Mode (Vertex AI) | `GOOGLE_API_KEY` | Vertex 빠른 시작 | 부분적 |
| Google Cloud ADC | `gcloud auth ...` | 배포/프로덕션 | 필요 |

| 하고 싶은 것 | 필요한 것 |
|---|---|
| 스킬만 설치해서 학습 | 없음 |
| `create` (프로젝트 생성) | 없음 |
| `run` / `playground` / `eval` | API 키 1개 |
| `deploy` / `infra` / `publish` | GCP 프로젝트 + 결제 활성화 |

> 보안: `auth.py`가 "쉘 설정 파일에 키를 export 하면 그 쉘에서 실행된 모든 프로세스가
> 읽을 수 있다"고 경고. 프로젝트 `.env` + `.gitignore` 사용을 권장.

### 3-4. 왜 GitHub에서 유명한가 (저장소 내용 기반 분석)

1. **구글 공식** (`google/` 오거니제이션, Apache-2.0) → 신뢰도
2. **타이밍** — "에이전트 프로덕션화"가 최대 화두인 시점에 정확히 그 구간을 커버
3. **벤더 중립** — 구글 도구인데 Claude Code / Codex를 공식 지원 (README에 로고 노출)
4. **Agent Starter Pack 후계자** — `docs/src/reference/from-agent-starter-pack.md` 마이그레이션 가이드로 기존 사용자층 승계
5. **스킬 생태계 선점** — `npx skills` + Claude 플러그인 + Gemini 확장 3중 호환
6. **완성도** — 459개 파일, 4개 언어, 4개 배포 타겟, 릴리스 노트에 이슈 번호까지 링크
7. **배포/마케팅** — 데모 영상, 문서 사이트, 애니메이션 랜딩

> 참고: 실시간 스타 수는 이 세션에서 확인하지 않았음(원본 저장소 접근 범위 밖).

### 3-5. 로컬 에이전트 구축에 도움이 되나

**직접 도움 60% + 학습 가치 100%**

로컬에서 가능: `create --deployment-target none`, `playground`, `run`, `eval run`,
in-memory 세션 → **API 키 하나로 로컬 개발/테스트 완결 가능**

한계: Gemini/ADK 중심(Ollama 등 로컬 LLM 네이티브 미지원), 모델 호출은 인터넷 필요,
배포는 GCP 전용

학습 가치가 큰 지점:
- 스킬 설계 패턴 (`SKILL.md` 7종) — AI에 지식 주입하는 정석
- 프로그레시브 디스클로저 (`SKILL.md` → `references/`) 2단 구조
- Lazy CLI 아키텍처 (`main.py`의 `LazyGroup`)
- 확장 시스템 스키마 설계
- **평가 방법론 (4단계 플라이휠)** — 프레임워크 무관하게 재사용 가능
- 템플릿 머지/업그레이드 전략 (`scaffold/utils/merge.py`, `upgrade.py`)

### 3-6. React / PHP로 만들 수 있나

| 접근 | 판정 |
|---|---|
| agents-cli 자체를 PHP로 포팅 | ❌ 비추 (ADK/Vertex SDK 부재, 생태계 재구현 비용 과다) |
| Node/TS로 포팅 | 🟡 가능하나 중복 |
| React로 포팅 | ❌ React는 UI 라이브러리, CLI 아님 |
| **React/PHP로 "주변 도구" 제작** | ✅✅ **강력 추천** |

React로 만들 만한 것:
1. **평가 대시보드** (점수 추이 / 히트맵 / 실패 드릴다운 / A/B 비교) — 현재 단일 HTML뿐이라 빈틈
2. 스캐폴드 위저드 (웹 GUI) — CLI 플래그가 많아 진입장벽 존재
3. 스킬 에디터 / 마켓플레이스
4. 에이전트 운영 콘솔 (로그·비용·트레이스·채팅 테스트)

PHP로 만들 만한 것:
- 워드프레스 플러그인(배포된 에이전트를 챗봇으로 삽입), 라라벨 SDK,
  기존 PHP 백오피스 통합, 그누보드/카페24 등 국내 특화 연동

연동 지점(이미 열려 있음):
`--json` 출력 · `fast_api_app.py`(REST) · A2A 모드 · 확장 시스템 · BigQuery 로그

---

## 4. 수익화 아이디어

### 먼저, 비어 있는 자리(기회)

| 빈틈 | 근거 | 기회 |
|---|---|---|
| 평가 결과가 1회용 | `results_<ts>.html` 단일 파일, 히스토리 없음 | 평가 SaaS |
| GCP 전용 | "Multi-cloud: Not yet supported" 명시 | 국내 클라우드 확장 |
| 영어 100% | 모든 스킬/문서 영어 | 한국어 스킬팩 |
| 템플릿이 기술 기준 | `adk`, `adk_go` 등 | 업종별 템플릿 |
| GUI 없음 | 순수 CLI | 웹 콘솔 |
| 도메인 지식 없음 | 프레임워크만 다룸 | 교육/컨설팅 |

### 아이디어 1 — 에이전트 평가 SaaS "EvalHub" ⭐1순위

`agents-cli eval` 결과 JSON을 수집해 **시간축 추적·회귀 감지·팀 공유**를 제공.

- 기능: 커밋별 점수 추이, 회귀 시 Slack 알림, 버전 A/B 비교, 실패 클러스터링 뷰
- 스택: React + Next.js / Node·Python API / Postgres + ClickHouse
- 가격: Free / Pro $29 / Team $99 / Enterprise
- 진입 전략: `agents-cli extension add` 로 `eval submit` 오버라이드 → 자동 전송
- 경쟁: LangSmith, Braintrust, Arize (단 ADK/agents-cli 특화는 공백)
- 난이도 ⭐⭐⭐⭐

### 아이디어 2 — 국내 특화 확장 `agents-cli-kr` ⭐2순위 (해자 깊음)

```yaml
schema: agents-cli-extension/v1alpha1
name: agents-cli-kr
commands:
  override:
    deploy: { run: ["python", "deploy_ncloud.py"] }     # 네이버클라우드
  add:
    "publish.kakao":    { ... }   # 카카오톡 채널
    "publish.naver":    { ... }   # 네이버웍스
    "compliance.check": { ... }   # 개인정보보호법 점검
```

- 타겟: 공공기관, 금융, 망분리 환경 (GCP 사용 불가 조직)
- 수익: 라이선스 연 500만~3,000만원 + 구축비
- 해자: 규제 = 진입장벽. 해외 업체 진입 곤란
- 난이도 ⭐⭐⭐

### 아이디어 3 — 업종별 에이전트 템플릿 마켓플레이스

병원 예약상담 / 법률 계약검토 / 이커머스 CS / 부동산 상담 / 학원 상담 / 제조 품질 등
업종별 템플릿을 프롬프트 + 툴 + **한국어 평가 데이터셋** + 배포 설정 + 매뉴얼로 판매.

- 구현 근거: `agents-cli create --agent <원격 템플릿 URL>` 지원 (`remote_template.py`)
- 가격: 건당 ₩100,000~300,000 또는 구독 ₩29,000/월
- 핵심 자산: **도메인 평가 데이터셋** (코드는 복제 가능하나 평가셋은 어려움)
- 난이도 ⭐⭐ (가장 빠른 착수)

### 아이디어 4 — 구축 대행 + 컨설팅 (현금흐름 1순위)

| 상품 | 가격 | 기간 |
|---|---|---|
| POC 패키지 | ₩300만~500만 | 2주 |
| 프로덕션 구축 | ₩2,000만~5,000만 | 2~3개월 |
| 월 운영/튜닝 | ₩150만~300만/월 | 지속 |
| 사내 교육(2일) | ₩500만 | - |

`agents-cli`가 보일러플레이트를 대폭 줄여주므로 **같은 단가에 원가 절감**.
동시에 아이디어 1~3의 실제 고객 니즈를 수집하는 채널로 활용.

### 아이디어 5 — 한국어 스킬팩 `agents-cli-skills-ko`

`npx skills add bmshin94/agents-cli-skills-ko`

- 난이도 ⭐ (1~2주), 직접 수익은 낮으나 **브랜딩 → 강의/컨설팅 유입**
- Apache-2.0이라 합법. 단 출처 표기 + NOTICE 유지 필수

### 아이디어 6 — 교육 콘텐츠

- 온라인 강의 ₩99,000 × 300명 ≈ ₩2,970만 (에버그린)
- 전자책 ₩29,000 / 기업 출강 ₩200만/일 / 유튜브 시리즈
- 차별점: **"에이전트 품질 측정 방법론"** — 국내 강의가 거의 없는 영역

### 아이디어 7 — 에이전트 운영 콘솔 (React)

배포된 에이전트 목록 · 상태 · 점수 · 비용 · 트레이스 · 채팅 테스트 · 알림.
데이터 소스(BigQuery Agent Analytics, Cloud Trace, 빌링 API)는 스캐폴드에 이미 구성됨.
SaaS ₩49,000/월 또는 온프레미스 라이선스. 난이도 ⭐⭐⭐

### 실행 로드맵

```
1개월차     아이디어 5 (한국어 스킬팩, 무료)      → 인지도
2~3개월차   아이디어 6 (교육) + 4 (POC 대행)      → 300만~1,000만
4~6개월차   아이디어 3 (템플릿) 또는 1 (평가 SaaS MVP) → 1,000만~3,000만
7~12개월차  아이디어 2 (국내 특화 확장)           → 5,000만+
```

### 법적 체크포인트

| 항목 | 상태 |
|---|---|
| Apache-2.0 | ✅ 상업적 이용 / 수정 / 재배포 가능 |
| 의무 | ⚠️ LICENSE·NOTICE 유지, 변경사항 명시 |
| "Google" 상표 | 🔴 제품명 사용 금지 (`google-agents-cli-pro` ❌) |
| 공식 인증 사칭 | 🔴 금지 ("구글 공식 파트너" 등) |
| 스킬 복제/번역 | ✅ 합법 (출처 표기 조건) |

---

## 5. 최종 결론

```
단기 (현금)   → 아이디어 4 (구축 대행) + 6 (교육)
중기 (제품)   → 아이디어 1 (평가 SaaS, React)
장기 (해자)   → 아이디어 2 (국내 특화 확장)
상시 (마케팅) → 아이디어 5 (한국어 스킬팩, 무료)
```

> 구글이 **"만드는 법"** 을 무료로 공개했으니,
> 우리는 **"잘 만들었는지 확인하는 법(평가)"** 과 **"한국에서 쓰는 법(현지화)"** 을 판다.
