# 논문 분야 지식 그래프

**논문과 연구자, 연구 분야를 연결해 연구의 흐름을 탐색하는 지식 그래프**

OpenAlex의 연구 분야 체계와 KAIST 김재철AI대학원 논문 데이터를 Neo4j에 저장합니다.
분야별 하위 주제를 살펴보고, 논문이 어떤 주제·저자와 연결되는지 탐색할 수 있습니다.

## 주요 기능

- **연구 분야 수집** — Computer Science의 `Field → Subfield → Topic` 계층 저장
- **논문 수집** — KAIST 김재철AI대학원의 2024~2026년 논문을 OpenAlex 제목 검색으로 연결
- **관계 구성** — 논문의 주제 점수, 저자와 저자 순서 정보를 함께 저장
- **중복 방지** — OpenAlex ID와 `MERGE`를 사용해 기존 노드·관계를 갱신

## 데이터 구조

```text
Field (field) ──HAS_CHILD──▶ Field (subfield) ──HAS_CHILD──▶ Field (topic)
                                                              ▲
                                                          HAS_TOPIC
                                                           {score}
                                                              │
                                                            Paper
                                                              │
                                                        AUTHORED_BY
                                                      {author_position}
                                                              ▼
                                                            Author
```

분야·하위 분야·주제는 모두 `Field` 노드로 저장하며, `level` 속성으로 구분합니다.
`Paper`에는 제목, 발행일, DOI, 초록, 인용 수 등 논문 메타데이터를 저장합니다.

## 시작하기

**준비:** Python 3.10 이상, 실행 중인 Neo4j, 외부 데이터 수집을 위한 인터넷 연결

### 1. 의존성 설치 및 환경 설정

```bash
pip install -r requirements.txt
cp .env.example .env
```

Windows PowerShell에서는 `cp` 대신 `Copy-Item .env.example .env`를 사용할 수 있습니다.

`.env`에 Neo4j 접속 정보를 입력합니다.

```dotenv
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-neo4j-password
```

선택 사항으로 `OPENALEX_MAILTO`에 요청 출처를 식별할 이메일을 설정할 수 있습니다.

### 2. 데이터 수집

프로젝트 루트에서 분야 계층을 먼저 저장한 뒤 논문을 수집합니다.

```bash
python src/ingest_field_hierarchy.py
python src/ingest_kaist_publications.py
```

논문은 OpenAlex에서 제목이 정확히 일치하는 항목만 저장합니다.
일치하는 항목을 찾지 못하면 건너뛰고, 실행 마지막에 해당 제목을 출력합니다.

### 3. 그래프 탐색

Neo4j Browser에서 논문과 주제·저자 연결을 확인합니다.

```cypher
MATCH (p:Paper)-[r:HAS_TOPIC|AUTHORED_BY]->(n)
RETURN p, r, n
LIMIT 100;
```

## 코드 구성

| 파일 (`src/`) | 역할 |
| --- | --- |
| `openalex_client.py` | 연구 분야 수집 및 논문 제목 검색 |
| `kaist_publication_client.py` | KAIST 연도별 논문 목록 수집 |
| `field_model.py` · `paper_model.py` · `author_model.py` | 데이터 모델 및 OpenAlex 응답 변환 |
| `neo4j_repository.py` | 노드·관계 저장 및 고유성 제약 관리 |
| `config.py` | 프로젝트 루트의 `.env` 로드 |
| `ingest_field_hierarchy.py` | Computer Science 분야 계층 수집 실행 |
| `ingest_kaist_publications.py` | KAIST 논문 수집 실행 |

구현 세부 사항과 추가 쿼리는 [온톨로지 문서](ontology.md)를 참고하세요.
