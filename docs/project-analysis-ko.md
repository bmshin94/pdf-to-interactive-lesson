# PDF to Interactive Lesson — 전수조사 분석 & 수익화 기획 정리

> 작성: Claude Code 세션 대화 정리본
> 정리일: 2026-09-28
> 대상 레포: `bmshin94/pdf-to-interactive-lesson` (브랜치 `claude/intelligent-wozniak-i5iruj`)

## 🔗 GitHub / 링크 모음

| 구분 | 주소 |
|---|---|
| 내 포크 (이 레포) | https://github.com/bmshin94/pdf-to-interactive-lesson |
| 원본 레포 (upstream) | https://github.com/Nutlope/pdf-to-interactive-lesson |
| 원본 제작자 | https://github.com/Nutlope (Hassan El Mghari, Together AI) |
| 원본 서비스 데모 | https://www.pdftolesson.com/ |
| 이 포크 배포본 | https://pdf-to-interactive-lesson.vercel.app |
| 원본 스타 / 포크 (조사 시점) | ⭐ 123 / 🍴 31 |

---

## 1. 이 프로젝트는 무엇인가

**PDF 한 장을 올리면 AI가 3개 모듈짜리 인터랙티브 강의(레슨 + 퀴즈)를 자동 생성하는 Next.js 웹앱 + CLI.**

단순 요약이 아니라 "문제를 풀면서 배우는" 코스 형태로 변환하는 것이 핵심이다.

### 파일 규모 (전수조사 기준, 총 286 파일)

| 폴더 | 내용 | 규모 |
|---|---|---|
| `app/` | Next.js 16 App Router — 페이지 + API 라우트 8개 + UI 컴포넌트 | ~7,500줄 |
| `lib/` | 핵심 로직 — AI 파이프라인, OCR, DB, Zod 스키마 | ~4,300줄 |
| `lib/pipeline/` | 병렬 생성 파이프라인 (`index` / `assign-flows` / `combined-flow` / `dedup-repair`) | 4파일 |
| `bin/course.ts` | CLI 도구 (+ zsh 자동완성 `bin/_course`) | 840줄 |
| `scripts/` | 벤치마크 · 평가 · 감사 하네스 | 25파일 |
| `data/benchmarks-slim/` | 실제 실험 결과 JSON | 80개 / 612KB |
| `docs/course-generation-speedup.md` | 9배 속도개선 실험 리포트 | — |
| `CLAUDE.md` | 이 포크에서 추가한 페르소나 설정 (원본엔 없음) | — |

### 기술 스택

- **프레임워크**: Next.js 16 (App Router), React 19.2
- **AI**: Together AI, 기본 모델 `openai/gpt-oss-120b`, 채점 모델 `openai/gpt-oss-20b`
- **PDF 추출**: MuPDF (WASM) — 로컬 텍스트 추출, API 호출 없음, 최대 100페이지
- **DB**: Neon Postgres + Drizzle ORM
- **스토리지**: Vercel Blob
- **큐 / 레이트리밋**: Vercel Queues (v2beta) + Upstash Redis
- **UI**: Tailwind CSS v4, Radix UI, `@dnd-kit/core`, `@xyflow/react` + `dagre`, lucide-react, vaul
- **검증**: Zod 3

### 실제 동작 흐름 (코드 확인 경로)

```
[브라우저] PDF 선택
  → POST /api/upload-url            Vercel Blob 직접 업로드 (.pdf만, 약 10MB 제한)
  → POST /api/generate-course       Redis에 job 생성 + Vercel Queue push
  → /api/queues/run-generation-job  (백그라운드 워커, maxDuration 800초)
      1. lib/ocr.ts            MuPDF로 로컬 텍스트 추출
      2. lib/pipeline/index.ts ① 모듈 3개 구조 생성 (LLM 1콜)
                               ② assign-flows: 모듈별 서로 다른 프로세스 배정 (LLM 1콜)
                               ③ 모듈 3개 병렬 생성 (Promise.all)
                               ④ dedup-repair: Jaccard 유사도로 중복 탐지 후 재생성
      3. Postgres 저장 → slug 발급
  → 클라이언트가 /api/generate-course/status 폴링
  → /course/[slug] 에서 학습 시작
```

### 생성되는 문제 유형 4가지

- `short-answer` — 서술형. `gpt-oss-20b`가 "정답 일치"가 아니라 "이해했는지"로 LLM 채점
- `true-false` — O/X + 해설
- `multiple-choice` — 4지선다. 모델은 정답을 항상 index 0에 넣게 시키고, 서버에서 Fisher-Yates 셔플 (인덱스 오류 원천 차단)
- `drag-drop` / `flow-diagram` — React Flow + dagre로 프로세스 순서 맞추기 드래그 퀴즈

### 코드에서 확인한 뛰어난 설계 포인트

1. **`lib/hint-answer-leak.ts` (546줄)** — 힌트에 정답이 섞이는 것을 막는 전용 모듈. `severity: none | partial | direct` 판정 + 자동 세정.
2. **`docs/course-generation-speedup.md`** — Together AI 서버리스 모델 15개를 실측 비교해 `MiniMax-M2.7 → gpt-oss-120b` 전환. 결과: **8.7배 빠름**, 정확도 92%→100%, 그라운딩 85%→99%.
3. **`benchmark_runs` 테이블** — 실험 추세 관리. `judgeStatus: 'real' | 'fake-100%' | 'no-judge' | 'none'` 컬럼으로 "심판이 조용히 실패해 100% 찍은 가짜 데이터"를 구분.
4. **Zod 기반 관용적 구조화 출력** — 배열 순서는 아무렇게나 받고 `sortLessonsByType`으로 정렬, `info`는 optional로 두고 없으면 채움. 모델의 변덕을 코드로 흡수.
5. **동시성 캡 자체 구현** — Vercel Queue에 max-concurrency가 없어 Upstash 카운터로 구현 (`generation-concurrency.ts`). 크래시 누수는 TTL로 자연 감소.
6. **레이트리밋** — IP당 무료 강의 3개 / 채점 50회 (평생). 자기 Together 키(BYOK) 사용 시 무제한.
7. **중복 처리 철학** — 재생성 2회 실패 시 해당 레슨을 `success:false`로 숨김. "12개 중 11개가, 중복 보이는 12개보다 낫다".

---

## 2. 쉬운 비유 설명

**"PDF 넣으면 문제집이 나오는 자동 학원 선생님 공장"**

1. **스캔실** — MuPDF가 PDF 안의 글자를 그대로 긁어온다. 공짜, 빠름. 단 **스캔 사진 PDF는 못 읽는다(최대 약점)**.
2. **커리큘럼실** — AI가 "3개 챕터로 나누자" 하고 목차를 만든다.
3. **문제 제작실** — 3챕터를 동시에 만든다(병렬, 빠름). 하지만 서로 안 보고 만들면 같은 문제를 낸다 → 그래서 **반장(assign-flows)이 미리 주제를 나눠줘서 충돌을 원천 차단**.
4. **중복 검사실** — 단어 겹침(Jaccard) 50% 이상이면 재생성, 두 번 실패하면 숨김.

결과물:

```
강의: "트랜스포머 아키텍처 이해하기"
├─ 모듈 1
│   ├─ 설명 3줄 → 서술형 (AI가 이해도로 채점)
│   ├─ 설명 → O/X + 해설
│   ├─ 설명 → 4지선다 + 해설
│   └─ 순서 맞추기 (드래그앤드롭 다이어그램)
├─ 모듈 2 …
└─ 모듈 3 …
```

진도는 브라우저 `localStorage`, 강의는 DB. 로그인 없음(익명 세션 ID). 기본 비공개, 공개 토글로 링크 공유.

---

## 3. 핵심 질문 정리

### 3-1. 설치 및 사용법

**요구사항**: Node 20+ (Next 16 기준 22 권장), **pnpm 9.5.0 고정** (npm/yarn 사용 시 lock 꼬임)

```bash
git clone https://github.com/bmshin94/pdf-to-interactive-lesson.git
cd pdf-to-interactive-lesson
pnpm install
cp .env.example .env.local     # 키 입력
pnpm db:push                   # Neon 테이블 생성
pnpm dev                       # http://localhost:3000
```

`.env.local`:

```bash
# 필수
TOGETHER_API_KEY=...
DATABASE_URL=postgresql://...
BLOB_READ_WRITE_TOKEN=...
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
NEXT_PUBLIC_APP_URL=http://localhost:3000
# 선택
OPENROUTER_API_KEY=...          # openrouter/ 심판
ANTHROPIC_API_KEY=...           # anthropic/ 심판
OLLAMA_BASE_URL=http://127.0.0.1:11434   # 로컬 모델 심판
GENERATION_MAX_CONCURRENT=2
```

> ⚠️ 큐 워커(`/api/queues/run-generation-job`)는 `vercel.json`의 Vercel Queues v2beta 트리거라서 **로컬 `pnpm dev`에서는 자동 실행되지 않는다.** 로컬 생성 테스트는 CLI를 쓰는 것이 정석.

**CLI (Together 키 하나만 필요, DB/Blob/Redis 불필요)**

```bash
pnpm course generate  data/문서.pdf
pnpm course modules   data/문서.pdf
pnpm course benchmark data/문서.md --runs 5
# 플래그: --model --output --verbose --max-retries --no-validate --save-text-auto
```

**검사 / 벤치마크**

```bash
pnpm test:hint-leak      # 힌트 답 유출 테스트
pnpm audit:hint-leaks    # 전체 감사
pnpm test:grade          # 서술형 채점 테스트
pnpm db:studio           # Drizzle Studio
tsx scripts/bench/speed-bench.ts --variants=... --iterations=3
tsx scripts/bench/measure-cost.ts
```

**배포**: Vercel 전용 설계(Blob + Queues + 800초 함수). 타 플랫폼 이전 시 큐 레이어를 BullMQ 등으로 교체 필요.

### 3-2. 플러그인? 스킬? MCP?

**셋 다 아니다. 독립 제품(Next.js 웹앱 + CLI)이다.**

| 분류 | 해당 | 근거 |
|---|---|---|
| Claude Code 플러그인 | ❌ | `.claude-plugin/`, `plugin.json` 없음 |
| Skill | ❌ | `SKILL.md`, `.claude/skills/` 없음 |
| MCP 서버 | ❌ | `@modelcontextprotocol/sdk` 의존성·서버 엔트리 없음 |
| Next.js 웹앱 | ✅ | `app/` + `next.config.ts` |
| CLI | ✅ | `package.json`의 `bin: { course: ./bin/course.ts }` |

혼동 요소: `CLAUDE.md`(이 포크에서 추가한 페르소나 파일), `docs/`의 `--judge=claude`(Claude CLI를 *심판 모델*로 쓰는 벤치마크 옵션).

> 다만 `lib/pipeline/generateCourse()`가 순수 함수라 **MCP 서버로 감싸기 매우 쉬움** → 차별화 기회.

### 3-3. API 토큰이 필요한가

| 서비스 | 필수도 | 용도 |
|---|---|---|
| Together AI | 필수 | 강의 생성 + 서술형 채점 |
| Neon Postgres | 웹앱 필수 | 강의 저장 |
| Vercel Blob | 웹앱 필수 | PDF 업로드 |
| Upstash Redis | 웹앱 필수 | 큐 + 레이트리밋 |
| OpenRouter / Anthropic | 선택 | 벤치마크 심판 |
| Ollama | 선택 | 로컬 심판 (키 불필요) |

- **CLI만 쓰면 Together 키 하나로 충분.**
- 웹앱엔 BYOK 구현됨: 사용자 키 → `localStorage` → `X-Together-API-Key` 헤더 → 레이트리밋 우회.
- ⚠️ 보안: 사용자 키가 Redis `job-store`에 TTL 1시간 저장됨. 상용화 시 암호화/구조 개선 권장.
- 비용: `gpt-oss-120b` = 입력 $0.15 / 출력 $0.60 per 1M 토큰 → 강의 1개당 수 센트 수준.

### 3-4. 왜 GitHub에서 유명한가

제작자 Nutlope(Hassan El Mghari)는 레포 92개를 보유한 오픈소스 스타다.

| 레포 | ⭐ |
|---|---|
| hallmark | 29,217 |
| roomGPT | 10,682 |
| aicommits | 9,099 |
| logocreator | 8,787 |
| llamacoder | 7,135 |
| llama-ocr | 2,439 |
| llamatutor | 2,052 |
| **pdf-to-interactive-lesson** | **123** |

이유:
1. **저자 브랜드** — Together AI 개발자 릴레이션, 팔로워 대규모
2. **바로 쓸 수 있는 프로덕션 템플릿** — 큐, 레이트리밋, BYOK, 공유링크, OG 이미지, Plausible까지 완비
3. **실제 엔지니어링이 존재** — 모델 15개 벤치마크 + 5개 품질 지표 + 실험 로그 80개 (일반 AI 데모 레포와 결정적 차이)
4. **Together AI 쇼케이스** — 오픈소스 모델로 상용급 품질을 증명하는 레퍼런스
5. **보편적 주제** — 학생/강사/HR/인강업체 모두의 니즈

> 단, ⭐123은 이 저자 기준으론 조용한 편 = **아직 안 터진 작품**. 경쟁 포크가 적고 로컬라이즈 여지가 크다는 기회.

### 3-5. 로컬 에이전트 구축에 도움이 되는가

**된다. "그대로 쓰기"보다 "패턴 차용"으로 큰 도움.**

| 훔쳐올 것 | 위치 | 가치 |
|---|---|---|
| 프로바이더 추상화 | `lib/utils/judge-model.ts` | 접두사(`anthropic/`,`openrouter/`,**`ollama/`**)로 스위칭. **로컬 Ollama 이미 지원** |
| Zod 구조화 출력 + 관용 파싱 | `lib/schemas.ts`, `lib/utils/json.ts` | LLM JSON 깨져도 복구 |
| Plan → Fan-out → Repair 3단 구조 | `lib/pipeline/index.ts` | 멀티에이전트 오케스트레이션 정석 |
| 충돌 사전 방지 | `pipeline/assign-flows.ts` | 병렬 에이전트 중복 작업을 사전 배정으로 원천 차단 |
| 자기 수리 루프 | `pipeline/dedup-repair.ts` | 실패 → 이유 피드백 → 재생성 → 우아한 포기 |
| LLM-as-judge 평가 하네스 | `scripts/eval-all.ts`, `scripts/bench/*` | 에이전트 품질을 숫자로 관리 |
| 토큰/비용 계측 | `lib/utils/together.ts`의 `__usageTracker` Proxy | 모델 객체를 Proxy로 감싸 전 호출 집계 |
| 긴 작업 큐 + 동시성 캡 | `job-store.ts`, `generation-concurrency.ts` | 장시간 에이전트 작업 처리 |
| 로컬 문서 파싱 | `lib/ocr.ts` | API 없이 오프라인 PDF 추출 |

**로컬 전환 로드맵**

```
1) Together → Ollama 전면 교체 (judge-model.ts의 ollama 분기를 생성부에도 적용) → 오프라인·비용 0
2) Neon → 로컬 Postgres / SQLite
3) Upstash → 로컬 Redis / 인메모리 큐
4) Vercel Blob → 로컬 파일시스템
5) MCP 서버로 감싸 Claude Code / Cursor에서 호출
```

**한계**: 툴 콜링·ReAct 루프·메모리·플래너가 없는 **고정 워크플로우(DAG)**. 자율 에이전트 프레임워크가 아니다. 다만 실무에서는 고정 파이프라인이 더 안정적·저렴하므로 학습 가치는 충분.

### 3-6. React / PHP로 만들 수 있는가

**React → 이미 React다.** (React 19.2 + Next 16 + Tailwind v4 + Radix + dnd-kit + React Flow)

순수 React(Vite)로 이전 시:
- 그대로 이동: `app/components/*` 대부분 (`drag-drop-question` 686줄, `lesson-screen` 573줄, `flow-diagram` 등)
- 교체: Server Component → 일반 컴포넌트, `next/image` → `<img>`, App Router → react-router
- 별도 필요: API 라우트 8개 → Express/Hono/Fastify 백엔드
- 예상 공수: 2~3일

**PHP → 가능. 라이브러리 매핑 필요.**

| 원본 | PHP 대안 |
|---|---|
| Next API 라우트 | Laravel 11 (권장) / Slim / Symfony |
| MuPDF (WASM) | `smalot/pdfparser` 또는 `pdftotext`(poppler) — **품질 차이 주의** |
| Vercel Queue + Redis job | Laravel Queue + Horizon (PHP가 오히려 편함) |
| Upstash 레이트리밋 | Laravel `RateLimiter` 기본 제공 |
| Drizzle ORM | Eloquent |
| Vercel Blob | S3 / MinIO / 로컬 storage |
| Zod | `opis/json-schema` / Laravel Validator |
| ai-sdk (Together) | Guzzle로 Together REST 직접 호출 (OpenAI 호환) |
| React UI | **PHP로 대체 불가** (드래그앤드롭·플로우차트는 JS 필수) |

**권장 하이브리드**: 프론트는 React(기존 컴포넌트 재사용), 백엔드는 Laravel(큐·인증·결제·관리자). 완전 PHP 전환은 `lib/` 4,300줄 재작성(약 2주) 비용이 발생하므로, 기존 PHP 자산이 없다면 Next.js 유지가 압도적으로 빠르다.

---

## 4. 수익화 아이디어

**강점 3가지**
1. 한계비용 거의 0 (강의 1개 = 수 센트, MuPDF는 무료)
2. BYOK + 무료 크레딧 구조가 이미 구현됨 (`lib/utils/rate-limiter.ts`)
3. 공유 링크 + 공개/비공개 토글이 이미 있음 = 바이럴 루프 준비 완료

### Tier 1 — 가장 빨리 수익화 가능

#### ① 국가고시·자격증 기출 자동 문제집 (최우선 추천)
- **타겟**: 공무원, 토익, 산업기사, 간호사, CPA, 의사국시
- **근거**: 한국 수험시장의 지불의사 높음 + PDF 자료 풍부 + 문제풀이 학습이 표준
- **개조**: 한국어 프롬프트 튜닝 / 모듈 3개 고정 → N개 동적 / 오답노트 + 망각곡선 복습 / 스캔 PDF OCR 추가
- **가격**: 무료 3개 · 라이트 ₩9,900 · 프로 ₩19,900 · 평생 ₩99,000
- **시뮬**: 유료 300명 × ₩15,000 ≒ 월 450만원, AI 원가 약 30만원 → 마진 90%+

#### ② 기업 교육 / 컴플라이언스 자동화 (B2B, 단가 최고)
- **타겟**: HR, 안전보건, ISO/개인정보 교육 운영사
- **근거**: 법정의무교육이 매년 반복되고 대부분 수동 제작
- **개조**: 조직/부서 관리(`createdBy` 확장) / **이수율 대시보드 + 수료증 PDF** / SSO + 감사로그 / 온프레미스(Ollama) 옵션으로 데이터 유출 0 보장
- **가격**: 시트당 ₩3,000~5,000/월, 최소 50석 → 계정당 월 15~25만원
- **시뮬**: 기업 20곳 ≒ 월 300~500만원, LTV가 B2C의 약 10배

#### ③ 강사·인강 제작자용 SaaS
- **타겟**: 온라인 강사, 학원, 교육 유튜버, 뉴스레터 운영자
- **개조**: 화이트라벨 / **생성 후 문제 편집 에디터(필수)** / SCORM·xAPI 내보내기 / Notion·Docs·유튜브 자막 입력
- **가격**: ₩29,000~99,000/월 + 화이트라벨 ₩199,000/월

### Tier 2 — 차별화 아이디어

4. **논문 스터디 도우미** — arXiv 링크 → 퀴즈. 벤치마크가 이미 논문 기반이라 성능 검증됨. 학생 ₩4,900/월, 랩 ₩49,000/월
5. **사내 온보딩 자동화** — 신입 교육자료 200p → 퀴즈 코스. ROI 명확
6. **MCP 서버로 판매** — `generateCourse()`를 MCP 툴로 노출. 공수 작고 선점 효과 큼
7. **API 판매** — `POST /v1/courses { pdf_url }` → 강의당 ₩200~500, 볼륨 할인
8. **부모용 아이 학습지 생성기** — 사교육 시장 지불의사 최상위. 단 저작권 정책 필수

### 리스크 & 대응

| 리스크 | 대응 |
|---|---|
| 저작권 (타인 교재 업로드) | 약관에 "본인 보유 자료만", DMCA 절차, 공개 링크 신고 기능 |
| **스캔 PDF 미지원 (현재 최대 약점)** | 클로바 OCR / Google Vision 폴백 — **수익화 전 필수 작업** |
| ChatGPT·NotebookLM 무료 경쟁 | 구조화된 학습 흐름 + 진도관리 + 채점 + 공유로 차별화 ("질문 1회"가 아닌 "코스") |
| AI 사실 오류 | 그라운딩 99% 측정값 보유 → **출처 페이지 표시** 기능 추가로 신뢰도 강화 |
| 라이선스 | 원본 레포 LICENSE 확인 필요 (현재 이 포크에 LICENSE 파일 없음) |

### 실행 로드맵

```
1주차: 한국어 프롬프트 튜닝 + 스캔PDF OCR 폴백      → 한국 시장 진입 최소요건
2주차: 문제 편집 에디터 + 모듈 개수 동적화           → 실사용 품질
3주차: 로그인(Clerk/Supabase) + 결제(토스페이먼츠)   → 과금 가능 상태
4주차: 수험생 커뮤니티 무료 배포 → 반응 측정          → PMF 검증
이후:  B2B 대시보드 + 수료증 발급                    → 단가 10배 확대
```

**권장 순서: ① 수험생(B2C)로 제품 검증 → ② 기업교육(B2B)로 단가 확보**

---

## 5. 한 줄 결론

이 레포는 **"문서 → AI 구조화 산출물" 파이프라인의 프로덕션급 교본**이며, 한국어 프롬프트 튜닝 + 스캔 PDF OCR + 문제 편집 에디터 3가지만 붙이면 **바로 판매 가능한 제품**이 된다.
