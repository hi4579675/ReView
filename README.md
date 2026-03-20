# ReView

> Your codebase has context. Your code reviewer should too.

diff만 보는 리뷰어는 그만. 레포 전체를 이해하는 AI 코드 리뷰어.

![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-Trigger-black?logo=github)
![Python](https://img.shields.io/badge/Python-3.11+-blue?logo=python)
![LangChain](https://img.shields.io/badge/LangChain-LCEL-green)
![pgvector](https://img.shields.io/badge/pgvector-Vector_DB-orange)
![Gemini](https://img.shields.io/badge/Gemini-1.5_Flash-purple?logo=google)

## 왜 만들었나요?
팀 프로젝트를 할 때 PR이 올라오면 코드 리뷰를 남겨야 하는데 diff만 봐서는 맥락을 알기 어려웠습니다.
"이 함수가 어디서 호출되지?", "기존 코드와 충돌이 안나나?"
결국 레포를 뒤지고, 관련 파일을 열어보고 나서야 코멘트를 남기곤 했는데 한 사람의 코드를 리뷰할 때 30분에서 1시간까지 걸렸습니다.
그래서 레포 전체를 이미 읽고 있는 리뷰어가 있다면 어떨까 싶었습니다.


```
// 기존 AI 리뷰
"이 함수 이름이 명확하지 않네요."

// ReView
"이 함수가 auth/middleware.py에서도 호출되는데,
거기선 반환값을 bool로 가정하고 있어요.
이 PR에서 None을 반환할 수 있게 바뀌었으니 거기도 수정이 필요해요."
```
일반적인 AI 코드 리뷰 도구는 PR의 diff만 보지만
**ReView**는 레포지토리 전체를 인덱싱해서, 변경된 코드가 다른 파일에 어떤 영향을 미치는지까지 이해하고 리뷰합니다.

## 주요 기능

- **컨텍스트 인식 리뷰** — 레포 전체를 RAG로 인덱싱해서 의존성까지 파악
- **AST 기반 청킹** — 함수/클래스 단위로 코드를 분리해 검색 정확도 향상
- **Query Decomposition** — 버그 / 컨벤션 / 성능 관점으로 자동 분리해서 검색
- **하이브리드 검색** — 벡터 유사도 + 키워드 매칭 결합
- **Cross-encoder 리랭킹** — 노이즈 제거 후 상위 컨텍스트만 LLM에 전달
- **멀티턴 대화** — PR 리뷰 중 추가 질문 가능
- **SQS 비동기 처리** — 여러 PR 동시 처리 시 안정적인 큐 관리
- **GitHub Actions 연동** — 별도 서버 없이 PR 오픈 시 자동 실행

## 아키텍처

```
PR 오픈 (GitHub Actions)
        ↓
  SQS 메시지 발행
        ↓
  PR diff 파싱
  + 레포 인덱싱 (AST 청킹)
        ↓
  Query Decomposition
  → 버그 / 컨벤션 / 성능 서브쿼리
        ↓
  Hybrid Retrieval (pgvector + BM25)
        ↓
  Cross-encoder 리랭킹
        ↓
  Gemini 1.5 Flash 리뷰 생성
        ↓
  GitHub PR 코멘트 작성
```

## 기술 스택

| 분류 | 기술 |
|---|---|
| LLM | Gemini 1.5 Flash |
| 임베딩 | text-embedding-004 |
| RAG 프레임워크 | LangChain LCEL |
| Vector DB | pgvector |
| AST 파싱 | tree-sitter |
| 리랭킹 | CrossEncoder (sentence-transformers) |
| 메시지 큐 | AWS SQS |
| 배포 | GitHub Actions |
| 평가 | RAGAs |

## 설치

### 1. 시크릿 설정

레포지토리의 Settings → Secrets → Actions에서 추가하세요.

| 시크릿 | 설명 |
|---|---|
| `GOOGLE_API_KEY` | Gemini API 키 |
| `DATABASE_URL` | pgvector 연결 문자열 |
| `AWS_ACCESS_KEY_ID` | AWS 액세스 키 |
| `AWS_SECRET_ACCESS_KEY` | AWS 시크릿 키 |
| `SQS_QUEUE_URL` | SQS 큐 URL |

### 2. 워크플로우 추가

레포지토리에 아래 파일을 추가하면 끝이에요.

```yaml
# .github/workflows/review.yml
name: ReView

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Run ReView
        uses: your-username/ReView@v1
        with:
          google_api_key: ${{ secrets.GOOGLE_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          database_url: ${{ secrets.DATABASE_URL }}
          sqs_queue_url: ${{ secrets.SQS_QUEUE_URL }}
```

### 3. PR 열기

이제 PR을 열면 ReView가 자동으로 리뷰 코멘트를 달아요.

## RAG 성능 실험

ReView는 단순한 서비스가 아니라, RAG 기법을 단계적으로 적용하며 성능 향상을 측정하는 실험 프로젝트이기도 해요.

| 단계 | 기법 | Hit@3 | Hit@5 | MRR |
|---|---|---|---|---|
| Stage 0 | Baseline (코사인 유사도) | - | - | - |
| Stage 1 | 청킹 최적화 (AST) | - | - | - |
| Stage 2 | HyDE + 하이브리드 검색 | - | - | - |
| Stage 3 | Cross-encoder 리랭킹 | - | - | - |
| Stage 4 | 임베딩 파인튜닝 | - | - | - |

> 측정 방법: RAGAs 프레임워크 사용 / `evaluation/` 폴더 참고

## 프로젝트 구조

```
ReView/
├── .github/
│   └── workflows/
│       └── review.yml
├── review/
│   ├── ingestion/
│   │   ├── ast_chunker.py       # tree-sitter 기반 AST 청킹
│   │   └── repo_indexer.py      # 레포 전체 인덱싱
│   ├── pipeline/
│   │   ├── query_decomposer.py  # Query Decomposition
│   │   ├── retriever.py         # 하이브리드 검색
│   │   └── reranker.py          # Cross-encoder 리랭킹
│   ├── queue/
│   │   └── sqs_handler.py       # SQS 비동기 처리
│   ├── github/
│   │   └── pr_handler.py        # PR diff 파싱 + 코멘트 작성
│   └── main.py
├── evaluation/
│   ├── stage0_baseline.ipynb
│   ├── stage1_chunking.ipynb
│   ├── stage2_retrieval.ipynb
│   ├── stage3_reranking.ipynb
│   ├── stage4_embedding_finetune.ipynb
│   └── results/
│       └── performance_comparison.png
├── tests/
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

## 로드맵

- [x] 프로젝트 설계
- [ ] 1단계: GitHub Actions + PR diff 파싱 + 기본 리뷰
- [ ] 2단계: AST 청킹 + 레포 인덱싱 + 하이브리드 검색
- [ ] 3단계: SQS + 리랭킹 + Query Decomposition + 멀티턴
- [ ] 4단계: RAGAs 평가 + 오픈소스 공개

## 기여

PR과 이슈 모두 환영합니다.

## 라이선스

MIT
