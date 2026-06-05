# RAG 아키텍처 (기본)

## 1. 정의

RAG(Retrieval-Augmented Generation)는 LLM이 답변을 생성할 때,  
외부 지식 베이스에서 관련 문서를 **검색(Retrieve)**해 컨텍스트에 **주입(Augment)**하고 **생성(Generate)**하는 아키텍처.

```
사용자 질문 → [검색] 관련 문서 찾기 → [주입] LLM에게 문서 전달 → [생성] 답변
```

LLM 혼자 답하는 게 아니라, 먼저 관련 자료를 찾아서 보여준 뒤 그걸 바탕으로 답하게 만드는 구조.

## 2. 등장 배경

| 연도 | 사건 |
|---|---|
| ~2020 | LLM은 학습 데이터에만 의존 → 최신 정보·내부 문서 불가 |
| 2020 | Meta(Facebook AI) "RAG for Knowledge-Intensive NLP Tasks" 논문 발표 |
| 2022+ | ChatGPT 등장 후 기업 내부 문서 활용 수요 폭발 |
| 2023+ | LangChain, LlamaIndex 등 RAG 프레임워크 대중화 |

LLM에게 모든 걸 학습시키는 건 너무 비싸고 느려 → 필요한 정보만 그때그때 찾아서 주입하는 방식으로 전환.

## 3. 해결하는 문제

- **Hallucination 감소**: 실제 문서 기반으로 답변하므로 LLM이 없는 정보를 지어내는 현상 억제
- **최신 정보 반영**: 학습 데이터 cutoff 이후 정보도 문서만 추가하면 즉시 반영
- **내부 문서 활용**: 기업 내부 보고서, 매뉴얼, 코드베이스 등 공개되지 않은 데이터 활용 가능
- **파인튜닝 없이 지식 업데이트**: 문서만 추가하면 되므로 재학습 비용 없음
- **출처 추적 가능**: 어떤 문서를 참고했는지 제공 가능

## 4. 내부 동작 원리

### 전체 파이프라인

**[사전 작업 — Indexing]**
```
문서 수집 → 청크 분할(Chunking) → 임베딩 생성 → 벡터 DB 저장
```

**[실시간 — Retrieval]**
```
사용자 쿼리 → 쿼리 임베딩 → 벡터 DB 유사도 검색 → 관련 문서 Top-K 추출
```

**[실시간 — Generation]**
```
[쿼리 + 검색된 문서] → LLM 프롬프트 구성 → LLM 답변 생성
```

### 청킹(Chunking) 전략

문서를 통째로 넣으면 너무 길어서 LLM이 처리 불가 → 적당한 크기로 잘라야 함.

| 전략 | 방법 | 특징 |
|---|---|---|
| Fixed-size | 일정 토큰 수로 분할 (예: 512 tokens) | 단순, 문맥이 끊길 수 있음 |
| Sentence | 문장 단위 분할 | 자연스러움 |
| Recursive | 단락 → 문장 → 단어 순으로 재귀 분할 | LangChain 기본 방식, 가장 많이 사용 |
| Overlap | 청크 간 겹치는 구간 설정 | 문맥 끊김 방지 |

```
[문서]
"FastAPI는 빠른 프레임워크다. 비동기를 지원한다. Python으로 작성됐다."

↓ chunk_size=20, overlap=5

청크1: "FastAPI는 빠른 프레임워크다. 비동기를"
청크2: "비동기를 지원한다. Python으로"
청크3: "Python으로 작성됐다."
```

### 프롬프트 구성 예시

```
[시스템]
당신은 주어진 문서를 바탕으로만 답변하는 AI입니다.
문서에 없는 내용은 "모른다"고 하세요.

[검색된 문서]
문서1: FastAPI는 Python 웹 프레임워크로 ...
문서2: FastAPI의 비동기 처리는 ...

[사용자 질문]
FastAPI의 특징은 무엇인가요?
```

## 5. 관련 기술

| 기술 | 역할 |
|---|---|
| LangChain | RAG 파이프라인 구성 프레임워크 |
| LlamaIndex | 문서 인덱싱 특화 프레임워크 |
| Supabase pgvector | 벡터 저장 + 검색 (프로젝트 사용) |
| ChromaDB | 로컬 경량 벡터 DB |
| sentence-transformers | 임베딩 모델 |
| Gemini API | LLM 생성 단계 (프로젝트 사용) |
| RecursiveCharacterTextSplitter | LangChain 청킹 도구 |

## 6. 실제 서비스 사례

- **Notion AI**: 워크스페이스 문서 검색 → AI 답변 생성
- **GitHub Copilot**: 코드베이스 RAG로 관련 코드 컨텍스트 주입
- **법률 AI**: 판례·법령 문서 RAG → 법률 Q&A
- **의료 AI**: 의학 논문 RAG → 임상 질의응답
- **기업 내부 챗봇**: 사내 문서·매뉴얼 기반 Q&A 시스템

## 7. 구현 예시

### 기본 RAG 파이프라인 (Gemini + Supabase pgvector)

```python
from sentence_transformers import SentenceTransformer
from supabase import create_client
import google.generativeai as genai

supabase = create_client(SUPABASE_URL, SUPABASE_KEY)
embed_model = SentenceTransformer("snunlp/KR-SBERT-V40K-klueNLI-augSTS")
genai.configure(api_key=GEMINI_API_KEY)
llm = genai.GenerativeModel("gemini-1.5-flash")

# 1. Indexing: 문서 청크 저장
def index_document(text: str):
    chunks = split_into_chunks(text, size=300, overlap=50)
    for chunk in chunks:
        embedding = embed_model.encode(chunk).tolist()
        supabase.table("documents").insert({
            "content": chunk,
            "embedding": embedding
        }).execute()

# 2. Retrieval: 관련 문서 검색
def retrieve(query: str, top_k: int = 3) -> list[str]:
    query_embedding = embed_model.encode(query).tolist()
    result = supabase.rpc("match_documents", {
        "query_embedding": query_embedding,
        "match_count": top_k
    }).execute()
    return [row["content"] for row in result.data]

# 3. Generation: LLM 답변 생성
def rag_answer(query: str) -> str:
    docs = retrieve(query)
    context = "\n\n".join(docs)
    prompt = f"""다음 문서를 바탕으로 질문에 답하세요.
문서에 없는 내용은 모른다고 하세요.

[문서]
{context}

[질문]
{query}"""
    response = llm.generate_content(prompt)
    return response.text
```

### LangChain으로 구성하는 방법

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import SupabaseVectorStore
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.chains import RetrievalQA

# 청킹
splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=50)
chunks = splitter.split_text(document_text)

# 벡터 저장 + 검색
vectorstore = SupabaseVectorStore.from_texts(chunks, embedding=embed_model)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# RAG 체인
llm = ChatGoogleGenerativeAI(model="gemini-1.5-flash")
chain = RetrievalQA.from_chain_type(llm=llm, retriever=retriever)
answer = chain.invoke("FastAPI의 특징은?")
```

## 8. 장단점

### 장점
- Hallucination 억제 → 실제 문서 기반 답변
- 최신 정보 즉시 반영 (문서 추가만 하면 됨)
- 파인튜닝 불필요 → 비용 절감
- 답변 출처 추적 가능 → 신뢰성 확보

### 단점
- 검색 품질이 나쁘면 답변도 나쁨 (Garbage In, Garbage Out)
- 청킹 전략이 성능에 큰 영향 → 튜닝 필요
- Retrieval 단계 레이턴시 추가 (임베딩 + 검색)
- 문서가 많아지면 벡터 DB 관리 복잡

## 9. 면접 질문

**Q. RAG와 파인튜닝의 차이는?**
> 파인튜닝은 모델 가중치를 직접 업데이트하므로 비용이 높고 데이터 변경 시 재학습이 필요합니다. RAG는 모델은 그대로 두고 외부 문서를 검색해 주입하므로, 최신 정보 반영이 즉시 가능하고 비용이 낮습니다. 도메인 지식이 자주 바뀌거나 데이터가 많을 때는 RAG가 유리합니다.

**Q. RAG에서 청킹이 중요한 이유는?**
> 청크가 너무 크면 LLM 컨텍스트 한계를 초과하고 노이즈가 많아집니다. 너무 작으면 문맥이 끊겨 의미 있는 검색이 어렵습니다. 적절한 크기와 overlap 설정이 검색 품질과 직결됩니다.

**Q. Hallucination이란?**
> LLM이 학습 데이터에 없거나 사실과 다른 내용을 자신 있게 생성하는 현상입니다. RAG는 실제 문서를 컨텍스트로 주입해 LLM이 문서 범위 내에서만 답하게 유도함으로써 Hallucination을 억제합니다.

## 10. 나의 프로젝트 적용 아이디어

주말 프로젝트(AI 멀티툴 에이전트 플랫폼)에서:

- **문서 기반 Q&A 기능**: 사용자가 PDF·텍스트 업로드 → 청크 분할 → Supabase pgvector 저장 → 질문 시 관련 청크 검색 → Gemini에 주입 → 답변 생성
- **청킹 전략**: `RecursiveCharacterTextSplitter(chunk_size=300, overlap=50)` 로 시작, 성능 보며 조정
- **Embedding 연결**: 이전 주제(Embedding)에서 공부한 KR-SBERT + pgvector를 그대로 Retrieval 단계에 적용
- **AI Agent 연동**: Agent가 Tool로 RAG 검색 함수를 호출하는 구조로 확장 가능 (다음 주제에서 이어짐)

## 11. 나의 생각
- 문서를 청크마다 임베딩 벡터가 생기는 것인가? 그렇다면 문서는 임베딩 벡터가 엄청 많은건가? 아니면 1대1 인가?
- llama는 무료 오픈소스로 알고 있는데 요새 가장 많이 쓰는 RAG 프레임워크는 무엇인가? qwen을 어디선가 들어봤는데 이거와 무관한가?
- 임베딩하는 모델이 여러개인데 google 임베딩 모델, opeani 임베딩 모델, meta 임베딩 모델이 있을텐데 어떤 방식으로 모델을 선정하는가?
- RAG는 결국 임베딩해서 문서를 찾는 것 외에는 다른 목적은 없는가?
- 구현 예시를 보니 모르는 건 모른다고 하세요. 라고 있던데 과연 진짜 모른다고 말하는가? 지어낼 것 같다.
- 결국 문서들도 임베딩화 해야되는데 예를 들어 ERP DB면 양이 너무 많을텐데 임베딩화하는데 너무 오래걸리거나 어려운 것이 아닌가?

## 12. 추가 공부

### 청크마다 임베딩 벡터가 생기는가? 1대1인가?

**청크 1개 = 벡터 1개.** 문서가 50청크로 나뉘면 벡터 DB에 50개 벡터가 저장된다.

```
문서 1개
  └─ 청크1 → 벡터[0.12, -0.34, ...] ← DB 저장
  └─ 청크2 → 벡터[0.55, 0.21, ...]  ← DB 저장
  └─ 청크3 → 벡터[-0.03, 0.87, ...] ← DB 저장
  ...
  └─ 청크50 → 벡터[...]              ← DB 저장
```

그래서 문서가 많아질수록 벡터 수도 비례해서 늘어남.  
100페이지 PDF → 약 300~500청크 → 300~500개 벡터.

---

### LLaMA, Qwen은 RAG 프레임워크인가?

**아니야. 둘 다 LLM 모델이야.** RAG에서 "Generation(생성)" 단계에 쓰이는 재료.

| 구분 | 예시 | 역할 |
|---|---|---|
| LLM 모델 | LLaMA(Meta), Qwen(Alibaba), Gemini(Google) | RAG의 Generation 단계에서 답변 생성 |
| RAG 프레임워크 | LangChain, LlamaIndex | RAG 파이프라인 전체를 연결하는 도구 |

- **LLaMA**: Meta의 오픈소스 LLM. 무료로 로컬 실행 가능. RAG의 LLM 부분으로 사용 가능.
- **Qwen**: Alibaba의 오픈소스 LLM. 중국어·영어 강점. 마찬가지로 Generation 단계에 사용.
- **LangChain**: 현재 가장 많이 쓰이는 RAG 프레임워크. 검색→주입→생성 파이프라인을 코드로 연결.
- **LlamaIndex**: 문서 인덱싱에 특화된 프레임워크. 이름에 Llama가 들어가지만 LLaMA 모델과 무관.

---

### 임베딩 모델 선정 기준

| 기준 | 고려 사항 |
|---|---|
| 언어 | 한국어 포함이면 KR-SBERT 또는 multilingual-e5 |
| 비용 | 로컬 무료(sentence-transformers) vs API 유료(OpenAI ada-002) |
| 차원 수 | 높을수록 표현력↑, 저장비용↑ (768 vs 1536) |
| 성능 벤치마크 | MTEB 리더보드에서 태스크별 순위 확인 |
| 속도 | 실시간 Indexing이 필요하면 경량 모델 선택 |

실무 선택 기준 요약:
- **무료 + 한국어**: `snunlp/KR-SBERT-V40K-klueNLI-augSTS`
- **무료 + 다국어**: `intfloat/multilingual-e5-base`
- **유료 + 고성능**: OpenAI `text-embedding-ada-002`, Google `text-embedding-004`

---

### RAG는 문서 검색 외에 다른 목적은 없나?

기본 목적은 "관련 문서 검색 → LLM 주입"이지만, 응용 범위는 넓다.

| 활용 | 설명 |
|---|---|
| 문서 Q&A | 기본 용도. 문서 기반 질의응답 |
| 출처 인용 | 어떤 문서에서 답했는지 함께 제공 |
| 팩트체킹 | LLM 답변을 실제 문서와 대조해 검증 |
| Multi-hop RAG | 여러 문서를 순차 검색해 복잡한 질문 해결 |
| Agent Tool | AI Agent가 필요할 때만 RAG 검색 함수를 호출하는 도구로 사용 |

---

### 모른다고 진짜 말하는가?

**반반이야.** 모델 성능과 프롬프트 강도에 따라 다르다.

- **GPT-4, Gemini 1.5 Pro** 수준: 지시를 잘 따라서 문서에 없으면 "모릅니다"라고 함
- **작은 모델(7B 이하)**: 프롬프트를 무시하고 학습 데이터 기반으로 지어낼 가능성 있음

완전히 막으려면 추가 장치가 필요하다:

```python
# 프롬프트에 강하게 제한
"문서에 없는 내용은 절대 답하지 말고 '해당 정보를 찾을 수 없습니다'라고만 하세요."

# temperature 낮추기 (창의성 억제)
llm = genai.GenerativeModel("gemini-1.5-flash", 
      generation_config={"temperature": 0.1})
```

RAG가 Hallucination을 **억제**하는 거지 **완전 차단**은 아님. 이게 RAG의 한계 중 하나.

---

### ERP같은 대용량 문서 임베딩, 너무 오래 걸리지 않나?

맞아. 실제로 대기업 ERP 데이터는 수백만 건이라 초기 Indexing이 수 시간~수 일 걸리기도 해.

**해결 방법:**

| 방법 | 설명 |
|---|---|
| Batch 처리 | 한 번에 수백 청크씩 묶어서 임베딩 (API 호출 횟수 감소) |
| 비동기 처리 | FastAPI background task로 백그라운드에서 진행 |
| 증분 업데이트 | 변경된 문서만 재임베딩 (전체 재처리 불필요) |
| 사전 작업 분리 | Indexing은 배포 전에 미리 해두고, 서비스 중엔 검색만 |

핵심은 **Indexing은 1회성 사전 작업**이라는 점이야.  
한 번 만들어두면 검색은 수십 ms로 빠르고, 문서가 바뀔 때만 해당 문서만 재임베딩하면 돼.  
ERP처럼 데이터가 방대한 경우 오히려 RAG가 파인튜닝보다 현실적인 선택이야.
