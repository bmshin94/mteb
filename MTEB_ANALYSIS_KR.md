# MTEB 분석 정리 노트 🇰🇷

> 본 문서는 MTEB 레포지토리를 직접 분석하고 정리한 한국어 가이드입니다.
> 설치/사용법, 프로젝트 구조, 활용 전략, 수익화 아이디어까지 한 번에 정리했습니다.

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| 이 레포 (포크) | https://github.com/bmshin94/mteb |
| 원본 레포 (upstream) | https://github.com/embeddings-benchmark/mteb |
| 공식 문서 | https://embeddings-benchmark.github.io/mteb/ |
| 리더보드 (HF Spaces) | https://huggingface.co/spaces/mteb/leaderboard |
| HuggingFace 조직 | https://huggingface.co/mteb |
| PyPI 패키지 | https://pypi.org/project/mteb/ |
| 논문 - MTEB | https://arxiv.org/abs/2210.07316 |
| 논문 - MMTEB (멀티링구얼 확장) | https://arxiv.org/abs/2502.13595 |

---

## 1. MTEB란 무엇인가

**MTEB (Massive Text Embedding Benchmark)** = **임베딩 모델 성능 채점기**

텍스트를 숫자 벡터(임베딩)로 바꾸는 모델들을 객관적인 기준으로 비교·평가하는
오픈소스 벤치마크 프레임워크입니다. RAG(검색 증강 생성) 시스템의 품질은 검색기,
즉 임베딩 모델이 좌우하기 때문에 "어떤 임베딩을 쓸 것인가"를 데이터로 답해줍니다.

### 기본 정보

| 항목 | 값 |
|---|---|
| 버전 | 2.21.0 |
| 라이선스 | Apache-2.0 (상업적 이용 가능) |
| Python 요구사항 | >=3.10, <3.15 |
| 파이썬 파일 수 | 1,739개 |
| 총 코드 라인 | 약 222,000줄 |
| 등록 태스크 클래스 | 1,994개 |
| 등록 모델 메타 | 931개 |
| 벤치마크 세트 | 59개 |

### 임베딩이란?

```
"강아지"  ->  [0.21, -0.88, 0.45, ...]   (예: 768차원 벡터)
"멍멍이"  ->  [0.19, -0.85, 0.43, ...]   <- 벡터 방향이 거의 동일
"자동차"  ->  [-0.7, 0.12, 0.91, ...]    <- 완전히 다른 방향
```

의미가 비슷한 문장은 벡터 공간에서 가까이 위치합니다. 이 성질을 이용해
검색, 분류, 군집화, 유사도 판별을 수행합니다.

---

## 2. 폴더 구조 분석

| 경로 | 역할 |
|---|---|
| `mteb/tasks/` | 평가 태스크 1,994개. 분류(340) / 검색(477) / 클러스터링(119) / STS(68) / 리랭킹(60) / BitextMining(39) 등 |
| `mteb/models/model_implementations/` | 931개 모델 정의 (254개 파일). BGE, E5, Cohere, OpenAI, Voyage, Gemini, Bedrock, ColPali 등 |
| `mteb/benchmarks/benchmarks/` | 벤치마크 세트 59개. `MTEB(eng)`, `BEIR`, `BRIGHT`, `C-MTEB`, **`MTEB(kor, v2)`** 등 |
| `mteb/evaluate.py` | 평가 실행 엔진. 캐시 전략(`always`/`never`/`only-missing`/`only-cache`) 지원 |
| `mteb/_evaluators/` | 지표 계산기 (NDCG, MAP, Recall, Accuracy, Spearman 등) |
| `mteb/leaderboard/` | Gradio 기반 리더보드 앱 |
| `mteb/api/` | FastAPI REST 서버 (React 프론트엔드용 데이터 제공) |
| `mteb/cli/` | `mteb run`, `mteb leaderboard` 등 CLI 명령 |
| `mteb/languages/` | ISO 639 / ISO 15924 언어·문자 코드 매핑 |
| `mteb/cache/` | 결과 캐싱 (재실행 시 비용·시간 절약) |
| `mteb/abstasks/` | 태스크 추상 클래스 (커스텀 태스크 작성의 기반) |
| `.github/workflows/` | CI 15종. 리더보드 자동 갱신, 모델 로딩 테스트, 헬스체크 등 |

### 멀티모달 확장

텍스트를 넘어 이미지(CLIP, BLIP, SigLIP, ColPali), 오디오(CLAP, Dasheng, Encodec),
비디오(InternVideo2)까지 지원합니다. `scripts/` 에는 로보틱스 데이터셋 빌드
스크립트(`build_droid_dataset.py`, `build_libero_datasets.py`)도 포함되어 있습니다.

---

## 3. 태스크 6가지 종목

### Retrieval (검색) - 477개, 가장 중요
```
질문: "아이폰 배터리 교체 비용은?"
-> 문서 100만 건 중 정답 문서를 상위로 끌어올림
지표: NDCG@10
```
RAG를 만든다면 이 점수를 최우선으로 봅니다.

### Classification (분류) - 340개
임베딩을 뽑아 로지스틱 회귀를 학습시킨 뒤 정확도를 측정합니다.
고객 문의 자동 분류, 스팸 필터 등에 직결됩니다.

### Clustering (군집화) - 119개
정답 라벨 없이 뉴스/문서를 자동으로 주제별 덩어리로 묶습니다.
데이터 탐색, 중복 제거에 사용됩니다.

### STS (문장 유사도) - 68개
사람이 매긴 유사도 점수와 모델 점수의 상관계수(Spearman)를 측정합니다.
한국어 `KorSTS`, `KLUE-STS`가 여기 속합니다.

### Reranking (재정렬) - 60개
1차 검색 결과 100개를 더 정밀한 모델로 다시 줄 세웁니다.
RAG 2단계(검색 -> 리랭크) 파이프라인의 리랭커에 해당합니다.

### PairClassification / BitextMining
NLI(함의/모순 판별), 번역쌍 찾기 등을 평가합니다.

---

## 4. 한국어 벤치마크 - MTEB(kor, v2)

레포에 한국어 벤치마크가 이미 내장되어 있습니다.

```python
MTEB_KOR_V2 = Benchmark(
    name="MTEB(kor, v2)",
    display_name="Korean",
    tasks=get_tasks(languages=["kor"], tasks=[
        "KLUE-TC",                       # 뉴스 주제 분류
        "SIB200ClusteringS2S",           # 클러스터링
        "KlueMrcDomainClustering",
        "KlueYnatMrcCategoryClustering",
        "KLUE-NLI", "KorNLI",            # 자연어 추론
        "PawsXPairClassification",       # 패러프레이즈 판별
        # 검색/STS 태스크 추가 포함
    ]),
)
```

v1에는 `KLUE-TC`, `MIRACLReranking`, `MIRACLRetrieval`, `Ko-StrategyQA`,
`KLUE-STS`, `KorSTS`가 포함되어 있으며 v2로 대체(superseded)되었습니다.

> 참고: 한국어 태스크는 영어권 대비 수가 적어 확장 여지가 큽니다.
> (아래 수익화 아이디어 6번 참고)

---

## 5. 설치 및 사용법

### 설치

```bash
# 기본
pip install mteb

# uv 사용 (더 빠름)
uv add mteb

# 개발/기여용 (레포 클론 후)
pip install -e ".[dev]"
```

### 선택 설치 (extras)

```bash
pip install "mteb[image]"        # 이미지 임베딩 (CLIP 등)
pip install "mteb[audio]"        # 오디오
pip install "mteb[leaderboard]"  # Gradio 리더보드 UI
pip install "mteb[api]"          # FastAPI 서버
pip install "mteb[openai]"       # OpenAI 임베딩 평가
pip install "mteb[cohere]"       # Cohere
pip install "mteb[voyageai]"     # Voyage AI
pip install "mteb[bm25s]"        # BM25 베이스라인
pip install "mteb[codecarbon]"   # 탄소배출량 측정
```

### Python API

```python
import mteb

# 1) 모델 로드 (MTEB에 정의가 없으면 SentenceTransformer로 폴백)
model = mteb.get_model("intfloat/multilingual-e5-large")

# 2-A) 태스크 직접 선택
tasks = mteb.get_tasks(tasks=["KLUE-STS", "KorSTS"])

# 2-B) 언어·유형으로 필터링
tasks = mteb.get_tasks(languages=["kor"], task_types=["Retrieval"])

# 2-C) 벤치마크 통째로
benchmark = mteb.get_benchmark("MTEB(kor, v2)")

# 3) 평가 실행
results = mteb.evaluate(model, tasks=tasks)
print(results)
```

### CLI

```bash
# 평가 실행
mteb run -m sentence-transformers/all-MiniLM-L6-v2 \
         -t "Banking77Classification.v2" \
         --output-folder results

mteb available-tasks        # 태스크 목록
mteb available-benchmarks   # 벤치마크 목록
mteb leaderboard            # 로컬 리더보드 UI 실행
mteb create-model-results   # HF 업로드용 모델 카드 생성
mteb mock-run               # 실제 모델 없이 파이프라인만 검증
```

### API 서버

```bash
pip install -e ".[api]"
uvicorn mteb.api.app:app --reload --port 8000
```

| 메서드 | 경로 | 반환 |
|---|---|---|
| GET | `/health` | 헬스체크 |
| GET | `/v1/benchmarks` | 벤치마크 목록 |
| GET | `/v1/benchmarks/menu` | 중첩 메뉴 트리 |
| GET | `/v1/benchmarks/{name}` | 벤치마크 메타데이터 |
| GET | `/v1/benchmarks/{name}/scores` | 순위표 전체 (camelCase JSON) |
| GET | `/v1/tasks/{name}/descriptive_statistics` | 태스크 통계 |
| GET | `/metrics` | Prometheus 메트릭 |

`PRELOAD=1` 환경변수를 주면 시작 시 백그라운드로 캐시를 미리 채웁니다.

---

## 6. 플러그인 / 스킬 / MCP - 무엇인가?

**셋 다 아닙니다. MTEB는 순수 파이썬 라이브러리 + CLI 입니다.**

| 구분 | 정체 | 실행 주체 | MTEB 해당? |
|---|---|---|---|
| 플러그인 | Claude Code 확장팩 | Claude Code | 아니오 |
| 스킬 | `SKILL.md` 지침 파일 | Claude가 읽음 | 아니오 |
| MCP | AI <-> 외부툴 연결 프로토콜 | MCP 클라이언트 | 아니오 |
| **MTEB** | **pip 패키지 + CLI + 평가 프레임워크** | 파이썬 런타임 | **예** |

근거: 레포에 `SKILL.md`, `.mcp.json`, `plugin.json`이 존재하지 않으며,
`pyproject.toml`에 `[project.scripts] mteb = "mteb.cli:main"`으로 CLI만 등록되어 있습니다.

### 다만, 셋 다로 감쌀 수 있음

1. **스킬로**: `.claude/skills/mteb-eval/SKILL.md` 작성 -> "임베딩 선택 시 mteb run 실행" 지침화
2. **MCP로**: `get_korean_ranking`, `compare_models` 등을 툴로 노출하는 MCP 서버 제작
3. **플러그인으로**: 위 둘 + 커맨드를 묶어 배포

현재 "MTEB MCP 서버"로 널리 알려진 구현이 없어 선점 기회가 있습니다.

---

## 7. API 토큰이 필요한가?

**기본적으로는 불필요합니다.**

| 상황 | 토큰 | 비고 |
|---|---|---|
| 오픈소스 모델 평가 (BGE, E5, MiniLM) | 불필요 | |
| 공개 데이터셋 다운로드 | 불필요 | |
| 리더보드 조회 | 불필요 | |
| `mteb mock-run` | 불필요 | |
| 게이트/비공개 HF 데이터셋 | `HF_TOKEN` | |
| OpenAI 임베딩 평가 | `OPENAI_API_KEY` | 유료 |
| Cohere 평가 | `COHERE_API_KEY` | 유료 |
| Voyage AI | `VOYAGE_API_KEY` | 유료 |
| Google Vertex / Gemini | GCP 인증 | 유료 |
| AWS Bedrock | AWS 크리덴셜 | 유료 |
| 리더보드 결과 제출 | HF 계정 + PR | |

코드 근거 (`mteb/models/model_implementations/cohere_v.py`):
```python
# do `export COHERE_API_KEY=<Your_Cohere_API_KEY>` before running eval scripts.
api_key = os.getenv("COHERE_API_KEY")
```

### 비용 주의

상용 API로 대형 검색 벤치마크를 돌리면 수백만 건 임베딩으로 요금이 급증합니다.

```python
# 캐시 활용: 이미 계산된 결과는 재사용
results = mteb.evaluate(model, tasks=tasks, overwrite_strategy="only-missing")

# 소규모 벤치마크부터 시작 (BEIR의 축소판)
tasks = mteb.get_benchmark("NanoBEIR")
```

---

## 8. 왜 GitHub에서 유명한가

1. **타이밍** - ChatGPT 이후 RAG 열풍과 정확히 겹침. "임베딩 뭐 쓰지?"의 유일한 답이었음
2. **HuggingFace 후원** - 저자에 HF 소속이 포함, 리더보드가 HF Spaces 공식으로 운영됨
3. **"1등" 마케팅 효과** - 모델 회사들이 앞다퉈 등록(931개) -> 네트워크 효과로 표준화
4. **논문 인용 폭발** - 임베딩 논문의 필수 레퍼런스 (arXiv 2210.07316, 2502.13595)
5. **커뮤니티 협업 설계** - `citation.cff`가 7KB에 달할 정도로 저자가 다수.
   각국 연구자가 자국어 벤치마크를 직접 기여하는 구조 -> 기여자가 곧 홍보대사
6. **엔지니어링 품질** - CI 15종, pre-commit, 타입체킹, Docker, mkdocs, 캐시, mock-run

---

## 9. 로컬 에이전트 구축에 도움이 되는가

MTEB는 에이전트 프레임워크가 아니라 **평가 도구**입니다. 하지만 로컬 에이전트의
가장 약한 고리인 **검색 품질**을 해결해 줍니다.

```
로컬 에이전트 = LLM + 임베딩(검색) + 벡터DB + 툴
                        ^
                 성능의 상당 부분을 좌우 -> MTEB 담당 영역
```

### 구체적 활용

1. **임베딩 모델 선택을 감이 아닌 데이터로**
```python
tasks = mteb.get_tasks(languages=["kor"], task_types=["Retrieval"])
for name in ["BAAI/bge-m3", "intfloat/multilingual-e5-large", "nlpai-lab/KURE-v1"]:
    print(name, mteb.evaluate(mteb.get_model(name), tasks=tasks))
```

2. **자사 도메인 데이터로 커스텀 태스크 제작** (가장 강력)
   공개 벤치마크 1위가 자사 문서에서는 3위일 수 있습니다.
```python
from mteb.abstasks import AbsTaskRetrieval

class MyCompanyDocsRetrieval(AbsTaskRetrieval):
    metadata = TaskMetadata(name="MyCompanyDocs", languages=["kor"], ...)
```

3. **비용/속도/성능 트레이드오프 근거** - `n_parameters`, `max_tokens`, `embed_dim` 메타데이터 활용
4. **임베딩 차원 축소 검토** - `mteb/models/compression_wrappers/` (Matryoshka 등)
5. **파인튜닝 효과 검증** - 튜닝 전후 점수 비교

### 주의사항

전체 벤치마크 실행에는 GPU와 수십 시간이 필요합니다.
**리더보드에서 후보 3~5개를 추린 뒤, 자사 데이터로만 소규모 평가**하는 방식을 권장합니다.

---

## 10. React / PHP로 만들 수 있는가

### 코어 재구현은 비현실적

PyTorch / Transformers / sentence-transformers 생태계에 완전히 종속되어 있고,
`pytrec-eval-terrier`(C++ 바인딩), scikit-learn, polars 등 대체재를 찾기 어렵습니다.

### 프론트엔드/대시보드는 100% 가능 (MTEB 자체가 그렇게 구성됨)

`mteb/api/README.md` 원문:
> "FastAPI surface that **powers the leaderboard frontend**"
> "JSON keys are emitted in camelCase to match the frontend types in `leaderboardv2/src/lib/types.ts`"

즉 TypeScript 프론트엔드가 이미 별도 레포로 존재합니다.

### 권장 아키텍처

```
React / Next.js  (직접 구현 영역)
  - 모델 비교 대시보드, 한국어 특화 순위표, 비용 계산기
        |  REST (JSON)
FastAPI  (mteb.api - 이미 구현되어 있음)
  uvicorn mteb.api.app:app --port 8000
        |
MTEB 코어 (Python) + 결과 JSON 캐시
```

### React 예시

```jsx
// app/leaderboard/page.tsx
async function getScores(benchmark = "MTEB(kor, v2)") {
  const res = await fetch(
    `http://localhost:8000/v1/benchmarks/${encodeURIComponent(benchmark)}/scores`,
    { next: { revalidate: 3600 } }
  );
  return res.json(); // camelCase
}

export default async function Page() {
  const { rows } = await getScores();
  return (
    <table>
      <tbody>
        {rows.map((r) => (
          <tr key={r.modelName}>
            <td>{r.rank}</td>
            <td>{r.modelName}</td>
            <td>{r.mean?.toFixed(2)}</td>
            <td>{r.numberOfParameters}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

### PHP 예시 (Laravel)

```php
class MtebClient {
    public function scores(string $benchmark = 'MTEB(kor, v2)'): array {
        return Cache::remember("mteb:$benchmark", 3600, fn() =>
            Http::timeout(30)
                ->get(config('mteb.url') . "/v1/benchmarks/" . rawurlencode($benchmark) . "/scores")
                ->json()
        );
    }
}
```

PHP는 평가를 직접 실행할 수 없으므로, 파이썬 워커를 큐(Redis/RabbitMQ)로 호출하는 구조를 사용합니다.

### 정리

| 목표 | 가능 여부 |
|---|---|
| 리더보드 웹앱 / 대시보드 | 가능 (API 기구현) |
| 모델 비교·필터·차트 | 가능 |
| 비용 계산기, 추천 위저드 | 가능 |
| 평가 실행 트리거 | 가능 (파이썬 워커 경유) |
| 브라우저 내 임베딩 추론 | 제한적 (transformers.js / ONNX, 소형 모델만) |
| MTEB 코어 재구현 | 비현실적 |

---

## 11. 수익화 아이디어

> 라이선스 체크: MTEB는 **Apache-2.0**으로 상업적 이용·수정·재배포가 허용됩니다.
> 단 (1) 라이선스/저작권 고지 유지, (2) MTEB 공식 사칭 금지,
> (3) **개별 데이터셋은 각자 라이선스가 별도**(비상업 제한 존재)이므로 반드시 확인해야 합니다.

### Tier 1 - 즉시 시작 가능 (초기 비용 거의 없음)

#### 아이디어 1. 한국어 임베딩 리더보드 + 선택 가이드 사이트
- **내용**: Next.js 사이트. MTEB(kor, v2) 데이터 + 한국어 해설 + 필터(무료/유료, 모델 크기, 차원)
- **차별점**: 공식 리더보드는 영어권 중심이며 정보가 과다. 한국어로 결론부터 제시
- **수익 경로**: 광고 / 임베딩 API 제휴 / 컨설팅 리드 수집 / 유료 뉴스레터
- **난이도**: 하 / **기간**: 2~3주
- **실행**: `uvicorn mteb.api.app:app` -> Next.js 연결 -> Vercel 배포

#### 아이디어 2. MTEB MCP 서버 (선점 기회)
- **내용**: `search_models`, `compare_models`, `get_korean_ranking`, `recommend_for_rag` 툴을 노출하는 MCP 서버
- **차별점**: 널리 알려진 구현이 아직 없음. MCP 생태계 확장기라 선점 가치가 큼
- **수익 경로**: OSS로 인지도 확보 -> 컨설팅/강의/채용 기회, Pro 버전 구독, 스폰서십
- **난이도**: 하 / **기간**: 1~2주
- **투자 대비 임팩트가 가장 큼 (최우선 추천)**

#### 아이디어 3. 콘텐츠 / 교육
- **내용**: "MTEB로 배우는 RAG 임베딩 선택법" 강의, 전자책, 블로그, 유튜브
- **수익 경로**: 온라인 강의 플랫폼, 전자책 판매, 기업 사내교육
- **부수효과**: 아이디어 1, 2로 트래픽을 유입시키는 선순환
- **난이도**: 하 / **기간**: 4~6주

### Tier 2 - B2B 본격 수익 (2~4개월)

#### 아이디어 4. 자사 데이터 기반 임베딩 벤치마크 SaaS (최우선 유망)
- **문제**: 공개 벤치마크 1위 모델이 자사 문서에서는 최적이 아닐 수 있음
- **해결 흐름**:
  1. 고객이 문서 업로드 (PDF / Notion / Confluence)
  2. LLM으로 질문-정답 쌍 자동 생성 (합성 평가셋)
  3. `AbsTaskRetrieval` 상속으로 커스텀 태스크 생성
  4. 후보 모델 10종 자동 평가 (NDCG@10, Recall@5 등)
  5. 리포트 산출: "귀사에는 X 모델이 최적. 상용 API 대비 성능 -1.2%, 비용 -94%"
- **가격 모델**: 1회 리포트 / Pro 구독 / Enterprise(온프레미스)
- **세일즈 포인트**: 상용 API 비용 절감의 객관적 근거 제공
- **난이도**: 상 / **기간**: 3~4개월

#### 아이디어 5. RAG 최적화 컨설팅
- **내용**: 임베딩 진단 + 청킹 전략 + 리랭커 튜닝 패키지
- **형태**: 프로젝트 단위 수주 또는 월 리테이너
- **강점**: MTEB 평가 리포트가 제안서의 신뢰도 근거로 작동
- **난이도**: 중 (영업력 필요) / 현금화가 가장 빠름

#### 아이디어 6. 한국어 벤치마크 자체 구축 (K-MTEB)
- **배경**: 현재 MTEB(kor, v2)는 태스크 수가 적어 확장 여지가 큼
- **내용**: 금융/의료/법률/이커머스 도메인별 한국어 평가셋 구축
- **수익 경로**: 데이터셋 상용 라이선스, 인증 마크, 정부 R&D 과제
- **부수효과**: 본가 MTEB에 기여 시 국제적 인지도 확보
- **난이도**: 상 (데이터 구축 공수)

### Tier 3 - 장기 / 고위험

#### 아이디어 7. 한국어 특화 임베딩 모델 개발 + API 판매
- MTEB로 성능 검증 -> "한국어 MTEB 최상위" 마케팅 -> API 과금
- GPU 비용이 크고 경쟁이 치열 (KURE, KoE5 등 기존 플레이어 존재)
- **난이도**: 최상

#### 아이디어 8. 임베딩 비용 최적화 툴 (FinOps)
- 차원 축소(Matryoshka), 양자화, 캐싱으로 벡터DB 비용 절감
- `mteb/models/compression_wrappers/` 활용
- 절감액 셰어 모델 가능

### 권장 로드맵

| 시점 | 실행 항목 | 목표 |
|---|---|---|
| 1개월차 | 한국어 임베딩 리더보드 사이트 오픈 (1번) | 트래픽·신뢰도 확보 |
| 2개월차 | MTEB MCP 서버 오픈소스 공개 (2번) | 개발자 커뮤니티 인지도 |
| 3개월차 | 블로그/강의 콘텐츠 발행 (3번) | 첫 현금 흐름 |
| 4~6개월차 | 컨설팅 수주 (5번) | 1~3번에서 확보한 리드 전환 |
| 6개월차~ | SaaS 제품화 (4번) | 컨설팅에서 검증된 니즈로 제품화 |

**핵심 전략**: 무료 툴로 신뢰 확보 -> 콘텐츠로 리드 수집 -> 컨설팅으로 현금 창출
-> 그 경험을 바탕으로 SaaS 제품화

**가장 현실적인 첫걸음**: `uvicorn mteb.api.app:app`을 띄우고 Next.js로 한국어 순위표
한 페이지를 만들어 보는 것.

---

## 12. 한눈에 보는 요약

| 질문 | 답 |
|---|---|
| 무엇인가? | 임베딩 모델 평가 벤치마크 프레임워크 (pip 패키지) |
| 언제 쓰나? | RAG 임베딩 선택, 모델 성능 검증, 논문 작성, 비용 의사결정 |
| 플러그인/스킬/MCP? | 전부 아님. 파이썬 라이브러리 + CLI (단, 셋 다로 감쌀 수 있음) |
| API 토큰 필요? | 오픈소스 모델은 불필요. 상용 API 평가 시에만 필요 |
| 왜 유명한가? | RAG 붐 + HF 공식 리더보드 + 논문 표준 + 커뮤니티 협업 설계 |
| 로컬 에이전트에 도움? | 검색 품질 결정에 직접 기여. 커스텀 태스크 제작이 핵심 |
| React/PHP 가능? | 프론트엔드/대시보드는 가능, 코어 재구현은 비현실적 |
| 수익화? | 리더보드 사이트 -> MCP 서버 -> 콘텐츠 -> 컨설팅 -> SaaS |
