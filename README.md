# Re:View

> RAG-based AI code reviewer that understands your entire codebase — not just the diff.

![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-Trigger-black?logo=github)
![Python](https://img.shields.io/badge/Python-3.11+-blue?logo=python)
![LangChain](https://img.shields.io/badge/LangChain-LCEL-green)
![pgvector](https://img.shields.io/badge/pgvector-Vector_DB-orange)
![Gemini](https://img.shields.io/badge/Gemini-2.5_Flash-purple?logo=google)

## 왜 만들었나요?

일반적인 AI 코드 리뷰 도구는 PR의 diff만 봐요.
`reviewbot`은 **레포지토리 전체를 인덱싱**해서, 변경된 코드가 다른 파일에 어떤 영향을 미치는지까지 이해하고 리뷰합니다.

```
// 기존 AI 리뷰
"이 함수 이름이 명확하지 않네요."

// reviewbot
"이 함수가 auth/middleware.py에서도 호출되는데,
거기선 반환값을 bool로 가정하고 있어요.
이 PR에서 None을 반환할 수 있게 바뀌었으니 거기도 수정이 필요해요."
```

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
| API 서버 | FastAPI |
| 배포 | GitHub Actions |
| 평가 | RAGAs |

## 빠른 시작

### 1. 설치

```bash
git clone https://github.com/your-username/ReView
cd Review
pip install -r requirements.txt
```

### 2. 환경 변수 설정

```bash
cp .env.example .env
```

```env
GOOGLE_API_KEY=your_gemini_api_key
GITHUB_TOKEN=your_github_token
DATABASE_URL=postgresql://localhost/reviewbot
AWS_ACCESS_KEY_ID=your_aws_key
AWS_SECRET_ACCESS_KEY=your_aws_secret
SQS_QUEUE_URL=your_sqs_queue_url
```

### 3. GitHub Actions 워크플로우 추가

레포지토리에 아래 파일을 추가하세요.

```yaml
# .github/workflows/reviewbot.yml
name: reviewbot

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run reviewbot
        env:
          GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          pip install reviewbot
          reviewbot run --pr ${{ github.event.pull_request.number }}
```

## 검색 정확도

vanilla Gemini 대비 RAG 파이프라인 적용 후 성능 개선 (측정 예정)

| 지표 | baseline | reviewbot |
|---|---|---|
| Hit@3 | - | - |
| Hit@5 | - | - |
| MRR | - | - |

> 측정 방법: RAGAs 프레임워크 사용, 코드 리뷰 품질 데이터셋 기반

## 프로젝트 구조

```
reviewbot/
├── .github/
│   └── workflows/
│       └── reviewbot.yml
├── reviewbot/
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
│   └── ragas_eval.ipynb         # RAGAs 평가 노트북
├── tests/
├── .env.example
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
