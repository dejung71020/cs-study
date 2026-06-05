# Embedding 모델과 유사도 검색

## 1. 정의

Embedding은 텍스트·이미지 등 비정형 데이터를 고차원 벡터(숫자 배열)로 변환하는 기술.  
의미적으로 유사한 데이터는 벡터 공간에서 가까운 거리에 위치하게 된다.

```
"강아지" → [0.12, -0.34, 0.87, ...]  (768차원)
"개"     → [0.11, -0.32, 0.85, ...]  (768차원)  ← 유사
"축구"   → [-0.54, 0.21, -0.03, ...] (768차원)  ← 거리 멀다
```

## 2. 등장 배경

| 연도 | 기술 | 의의 |
|---|---|---|
| 2013 | Word2Vec (Google) | 단어 간 의미 관계 학습 가능 |
| 2017 | Transformer (Attention Is All You Need) | 문맥 이해 가능 |
| 2019 | BERT | 문장 수준 의미 임베딩 가능 |
| 2020+ | Sentence-BERT, text-embedding-ada-002 | 실용적 임베딩 모델 대중화 |

전통적인 키워드 검색(BM25, TF-IDF)은 단어 완전 일치에만 반응했다.  
→ "강아지"와 "개"를 완전히 다른 개념으로 인식하는 한계.

## 3. 해결하는 문제

- **의미 기반 검색**: "notebook 구매" 쿼리로 "laptop 추천" 문서 찾기 가능
- **언어 장벽 극복**: 한국어 쿼리 → 영문 문서 검색 (다국어 임베딩)
- **RAG의 핵심**: LLM에 외부 지식 주입 시 관련 문서를 찾는 검색 단계
- **추천 시스템**: 유저 취향 벡터 ↔ 아이템 벡터 유사도 계산

## 4. 내부 동작 원리

### 임베딩 생성 과정

```
입력 텍스트 → Tokenizer → Transformer 인코더 → [CLS] 토큰 벡터 → L2 정규화 → 임베딩 벡터
```

### Self-Attention (Transformer 핵심)

```
Q = 쿼리 행렬 (내가 찾는 것)
K = 키 행렬   (각 토큰의 특성)
V = 값 행렬   (실제 정보)

Attention(Q, K, V) = softmax(QK^T / √d_k) × V
```

각 단어가 문장 내 다른 단어들과 얼마나 관련 있는지 가중치를 계산해 문맥 반영.

### 유사도 측정 방법

| 방법 | 공식 | 특징 |
|---|---|---|
| 코사인 유사도 | cos(θ) = A·B / (\|A\|\|B\|) | 방향만 비교, 크기 무관. 가장 일반적 |
| 유클리드 거리 | √Σ(aᵢ-bᵢ)² | 절대 거리. 좌표 공간 |
| 내적 (Dot Product) | A·B | 크기 + 방향. FAISS에서 빠름 |

```python
import numpy as np

def cosine_similarity(vec_a, vec_b):
    dot = np.dot(vec_a, vec_b)
    norm = np.linalg.norm(vec_a) * np.linalg.norm(vec_b)
    return dot / norm
```

## 5. 관련 기술

| 기술 | 역할 |
|---|---|
| Sentence-BERT (SBERT) | 문장 임베딩 특화 모델 |
| KR-SBERT | 한국어 특화 SBERT |
| text-embedding-ada-002 | OpenAI 임베딩 모델 (1536차원) |
| multilingual-e5 | 다국어 임베딩 |
| FAISS | Facebook AI 벡터 유사도 검색 라이브러리 |
| pgvector | PostgreSQL 벡터 확장 (Supabase 사용) |
| ChromaDB | 경량 로컬 벡터 DB |
| BM25 | 키워드 기반 검색 (Hybrid Search에서 병행) |

## 6. 실제 서비스 사례

- **ChatGPT**: 학습 데이터 인덱싱 및 검색에 임베딩 활용
- **Notion AI**: 워크스페이스 문서를 임베딩 → 관련 문서 자동 추천
- **Netflix / Spotify**: 콘텐츠·유저 임베딩으로 추천 시스템 구현
- **GitHub Copilot**: 코드 파일 임베딩 → 관련 코드 컨텍스트 주입
- **Google 검색**: 2019년 BERT 도입 이후 의미 기반 검색 강화

## 7. 구현 예시

### 기본 임베딩 생성 + 유사도 검색

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("snunlp/KR-SBERT-V40K-klueNLI-augSTS")

docs = [
    "Python은 인기 있는 프로그래밍 언어입니다.",
    "FastAPI는 빠른 Python 웹 프레임워크입니다.",
    "축구는 전 세계에서 인기 있는 스포츠입니다.",
]

doc_embeddings = model.encode(docs)          # shape: (3, 768)
query_embedding = model.encode("백엔드 개발에 쓰이는 언어")

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

scores = [cosine_similarity(query_embedding, doc) for doc in doc_embeddings]
best_idx = np.argmax(scores)
print(f"가장 유사한 문서: {docs[best_idx]} (점수: {scores[best_idx]:.4f})")
# → "FastAPI는 빠른 Python 웹 프레임워크입니다."
```

### Supabase pgvector 연동

```python
from supabase import create_client
from sentence_transformers import SentenceTransformer

supabase = create_client(SUPABASE_URL, SUPABASE_KEY)
model = SentenceTransformer("snunlp/KR-SBERT-V40K-klueNLI-augSTS")

def insert_document(text: str):
    embedding = model.encode(text).tolist()
    supabase.table("documents").insert({
        "content": text,
        "embedding": embedding    # vector(768) 컬럼
    }).execute()

def search_similar(query: str, top_k: int = 3):
    query_embedding = model.encode(query).tolist()
    result = supabase.rpc("match_documents", {
        "query_embedding": query_embedding,
        "match_count": top_k
    }).execute()
    return result.data
```

## 8. 장단점

### 장점
- 키워드 불일치에도 의미 기반 검색 가능
- 다국어 쿼리 지원 (multilingual 모델)
- RAG, 추천, 분류 등 다양한 태스크에 범용 활용

### 단점
- 모델 크기에 따라 추론 비용 증가
- 도메인 특화 텍스트는 범용 모델 성능 저하 가능 (파인튜닝 필요)
- 차원이 클수록 저장 비용 증가 (1536차원 = float32 기준 약 6KB/벡터)
- 임베딩만으로 정확한 키워드 매칭 불가 → Hybrid Search 필요

## 9. 면접 질문

**Q. 코사인 유사도와 유클리드 거리의 차이는?**
> 코사인 유사도는 벡터의 방향(각도)만 비교해 크기에 무관합니다. 유클리드 거리는 절대 거리를 측정합니다. 문장 길이에 상관없이 의미 유사도를 측정해야 하는 임베딩 검색에서는 코사인 유사도가 더 적합합니다.

**Q. 임베딩 차원이 크면 무조건 좋은가?**
> 아닙니다. 차원이 클수록 표현력은 높지만 저장 비용과 검색 속도가 선형 증가합니다. 차원이 지나치게 높으면 "차원의 저주"로 유사도 구분력이 오히려 낮아질 수 있습니다.

**Q. RAG에서 임베딩의 역할은?**
> 사용자 쿼리와 문서 청크를 모두 벡터로 변환한 뒤, 코사인 유사도로 가장 관련 있는 문서를 검색합니다. 검색된 문서를 LLM 컨텍스트에 주입해 정확한 답변을 생성하는 핵심 단계입니다.

## 10. 나의 프로젝트 적용 아이디어

주말 프로젝트(AI 멀티툴 에이전트 플랫폼)에서:

- **문서 검색 파이프라인**: 사용자 업로드 문서 → 청크 분할 → 임베딩 → Supabase pgvector 저장 → 쿼리 임베딩 → 코사인 유사도 검색 → LLM 컨텍스트 주입
- **한국어 모델 선택**: `snunlp/KR-SBERT-V40K-klueNLI-augSTS` (무료, HuggingFace) 또는 Gemini Embedding API (무료 티어)
- **Supabase 통합**: Agent 상태 저장용 PostgreSQL에 pgvector 확장만 추가하면 벡터 검색까지 같은 DB에서 처리 가능 → 인프라 단순화
- **다음 단계**: 이번 주제(벡터 검색 원리) → RAG 아키텍처 → RAG 고도화(BM25 + 벡터 Hybrid Search) 순서로 확장

## 11. 나의 생각
- Tokenizer 가 무엇인가
- Transformer 가 무엇인가
- CLS 토큰 벡터가 무엇인가
- L2 정규화가 무엇인가
- FAISS 가 무엇인가
- 의미 유사도는 코사인 유사도가 적합하다는데 유클리드 거리는 언제 쓰이나

## 12. 추가 공부

### Tokenizer란?

텍스트를 모델이 처리할 수 있는 **토큰(token)** 단위로 쪼개는 도구.  
단어 전체가 아닌 **서브워드(subword)** 단위로 분리해 미등록 단어 문제를 해결한다.

```
"FastAPI는 빠르다" → ["Fast", "##API", "##는", "빠르", "##다"]
```

| 방식 | 사용 모델 | 특징 |
|---|---|---|
| BPE (Byte-Pair Encoding) | GPT 계열 | 빈도 높은 문자 쌍을 반복 병합 |
| WordPiece | BERT 계열 | ##로 이어지는 서브워드 표시 |
| SentencePiece | 다국어 모델 | 공백 포함 언어(한국어 등)에 적합 |

---

### Transformer란?

2017년 Google이 발표한 논문 "Attention Is All You Need"에서 등장한 신경망 구조.  
기존 RNN의 순차 처리 한계를 Self-Attention으로 극복해 병렬 처리 가능.

```
입력 토큰 → Positional Encoding → [Self-Attention → Feed Forward] × N층 → 출력 벡터
```

- **Encoder**: 입력 문장을 벡터로 압축 (BERT, 임베딩 모델에 사용)
- **Decoder**: 벡터를 받아 텍스트 생성 (GPT 계열에 사용)
- **Encoder-Decoder**: 번역, 요약 등 (T5, BART)

임베딩 모델은 Encoder만 사용해 입력 문장의 의미 벡터를 추출한다.

- 토큰에 순서를 부여하고, 셀프 어텐션을 통해 단어 가중치를 계산하고, 피드 포워드로 특징을 추출하여 벡터를 만든다.
---

### CLS 토큰 벡터란?

BERT 계열 모델은 모든 입력 앞에 **[CLS]** 라는 특수 토큰을 자동으로 붙인다.

```
입력: "FastAPI는 빠르다"
실제 처리: [CLS] Fast ##API ##는 빠르 ##다 [SEP]
```

Transformer가 Self-Attention으로 모든 토큰 관계를 학습하면,  
**[CLS] 위치의 벡터**에 문장 전체 의미가 압축된다.  
→ 이 벡터를 꺼내서 문장 임베딩으로 사용.

- CLS 토큰 벡터는 Self-Attention으로 문장 내 모든 단어를 참조한 의미를 압축하여 저장하는 역할을 한다. 문장 맨 앞에 붙는다.
---

### L2 정규화란?

벡터의 크기(L2 노름)를 1로 만드는 연산. 모든 벡터를 단위 구(unit sphere) 위에 올린다.

```
L2 노름: ||v|| = √(v₁² + v₂² + ... + vₙ²)

정규화: v_norm = v / ||v||
```

```python
import numpy as np
v = np.array([3.0, 4.0])
v_norm = v / np.linalg.norm(v)   # [0.6, 0.8]
print(np.linalg.norm(v_norm))     # 1.0
```

**왜 하나?** L2 정규화 후에는 코사인 유사도 = 내적(Dot Product)이 되어  
계산이 단순해지고 FAISS 같은 라이브러리에서 고속 처리 가능.

- L2 정규화를 하면 크기가 1이 되어 온전한 방향만으로 비교가 가능하다.
- L2 정규화를 하면 연산의 분모가 1이된다. 그래서 연산이 빨라진다.
---

### FAISS란?

Facebook AI가 만든 **대규모 벡터 유사도 검색 라이브러리**.  
수백만~수십억 개 벡터를 빠르게 검색할 수 있도록 최적화되어 있다.

| 인덱스 | 방식 | 특징 |
|---|---|---|
| IndexFlatL2 | 정확한 유클리드 거리 탐색 | 느리지만 정확, 소규모용 |
| IndexFlatIP | 정확한 내적(Inner Product) 탐색 | L2 정규화 후 코사인 유사도와 동일 |
| IndexIVFFlat | 클러스터 기반 근사 탐색 | 빠름, 대규모용 |
| HNSW | 그래프 기반 근사 탐색 | 매우 빠름, 정확도 높음 |

```python
import faiss
import numpy as np

d = 768  # 벡터 차원
index = faiss.IndexFlatIP(d)         # 내적 기반 (L2 정규화된 벡터에 사용)
index.add(doc_embeddings)            # 문서 벡터 추가
D, I = index.search(query_vec, k=3)  # 상위 3개 검색
# D: 유사도 점수, I: 문서 인덱스
```

---

### 유클리드 거리는 언제 쓰나?

코사인 유사도가 **방향(각도)**만 보는 반면, 유클리드 거리는 **절대 거리**를 본다.

| 상황 | 적합한 방법 | 이유 |
|---|---|---|
| 의미 유사도 검색 (RAG) | 코사인 유사도 | 문장 길이(벡터 크기)에 무관 |
| K-Means 클러스터링 | 유클리드 거리 | 중심점(centroid) 계산에 적합 |
| L2 정규화된 벡터 검색 | 둘 다 동일 | 정규화 후엔 코사인 ∝ 유클리드 |
| 이미지 픽셀 거리 | 유클리드 거리 | 절대 픽셀 차이가 의미 있음 |
| FAISS IndexFlatL2 | 유클리드 거리 | 정규화 없이 빠른 근사 탐색 |

> **결론**: 텍스트 임베딩 검색에는 코사인, 클러스터링·정규화된 벡터·이미지엔 유클리드.
