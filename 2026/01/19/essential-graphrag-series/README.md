# Essential GraphRAG 시리즈: 벡터 검색에서 에이전틱 RAG까지

[##_Image|kage@bhIm1f/dJMcadAJbIv/AAAAAAAAAAAAAAAAAAAAAGukAmAmcKYeGFm8IIjL1ZhEhyQx3yve1yVWBUTZ18RU/img.jpg|alignCenter|width="100%"|_##]

> **시리즈 개요**: 이 시리즈는 LLM의 한계를 극복하고 신뢰할 수 있는 AI 시스템을 구축하기 위한 RAG(Retrieval-Augmented Generation) 기술의 진화 과정을 다룬다. 단순 벡터 검색부터 지식 그래프 기반 GraphRAG, 자율적 판단이 가능한 Agentic RAG까지 단계별로 학습한다.


## 시리즈 구성

| Part | 제목 | 핵심 주제 |
|------|-----|----------|
| 1 | [LLM 정확도 향상](https://blog.imprun.dev/148) | LLM의 구조적 한계와 RAG의 필요성 |
| 2 | [벡터 검색과 하이브리드 검색](https://blog.imprun.dev/149) | 임베딩, 코사인 유사도, 키워드 검색 결합 |
| 3 | [고급 벡터 검색 전략](https://blog.imprun.dev/150) | Step-back Prompting, Parent Document Retriever |
| 4 | [Text2Cypher: 자연어를 그래프 쿼리로](https://blog.imprun.dev/151) | Cypher 쿼리 생성, 스키마 활용 |
| 5 | [Agentic RAG](https://blog.imprun.dev/152) | 자율적 도구 선택, 자가 교정 루프 |
| 6 | [LLM으로 지식 그래프 구축](https://blog.imprun.dev/153) | 엔티티/관계 추출, 자동 그래프 생성 |
| 7 | [Microsoft GraphRAG 구현](https://blog.imprun.dev/154) | 커뮤니티 탐지, Global/Local Search |
| 8 | [RAG 평가 체계](https://blog.imprun.dev/155) | Faithfulness, Ground Truth, RAGAS |

## 학습 경로

```
[기초: LLM의 한계 이해]
Part 1 → Part 2 → Part 3

[그래프 통합: 관계 기반 검색]
Part 4 → Part 5 → Part 6

[고급 구현과 평가]
Part 7 → Part 8
```

## 기술 스택

- **LLM**: OpenAI GPT-4, Claude
- **벡터 DB**: ChromaDB, Neo4j Vector
- **그래프 DB**: Neo4j
- **프레임워크**: LangChain, LlamaIndex
- **평가**: RAGAS

## 참고 자료

- [Essential GraphRAG (원서)](https://go.neo4j.com/rs/710-RRC-335/images/Essential-GraphRAG.pdf)
- [Neo4j RAG Tutorial](https://neo4j.com/blog/developer/rag-tutorial/)
- [Microsoft GraphRAG](https://github.com/microsoft/graphrag)
