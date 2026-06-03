먼저 `SimpleVectorStore` + 문자열 문서 몇 개로 RAG가 도는지 확인하고, 작동하면 실제 문서 파일을 `DocumentReader`/`TokenTextSplitter`로 적재하는 ETL 단계를 붙이고, 마지막에 운영용 벡터 DB(PGVector 등)로 교체하는 순서입니다.

# 백터 DB
먼저 핵심 질문. 위 FAQ를 그냥 일반 RDB에 넣고 검색하면 안 될까요? 문제는 **사용자가 문서의 단어 그대로 묻지 않는다**는 데 있습니다.

```
문서에 적힌 말: "membership tiers", "tier status is evaluated on January 1st"
사용자가 묻는 말: "내 등급 언제 바뀌어요?", "VIP 되려면 얼마 써야 해요?"
```

`LIKE '%등급%'` 같은 키워드 검색은 단어가 정확히 겹쳐야 찾습니다. "등급"과 "tier", "언제 바뀌어요"와 "evaluated on January 1st"는 글자가 안 겹치니 못 찾아요. 하지만 **의미는 같습니다.** 벡터 DB는 바로 이 "글자가 달라도 의미가 비슷한 것"을 찾기 위한 도구입니다.

## 임베딩 — 의미를 좌표로 바꾸기

핵심 원리는 **임베딩(embedding)** 입니다. 텍스트를 임베딩 모델에 넣으면, 그 의미를 나타내는 **숫자 배열(벡터)** 이 나옵니다.

```
"VIP 등급 혜택"        → [0.12, -0.34, 0.88, ... ]  (예: 1536개의 숫자)
"VIP 멤버십 이점"      → [0.13, -0.31, 0.85, ... ]  ← 의미 비슷 → 좌표도 가까움
"환불 처리 기간"        → [-0.77, 0.42, 0.05, ... ]  ← 의미 다름 → 좌표 멀리 떨어짐
```

의미가 비슷한 문장은 이 다차원 공간에서 **가까운 좌표**에 위치합니다. 그래서 "VIP 되려면 얼마?"라는 질문을 같은 방식으로 벡터화한 뒤, **그 좌표와 가장 가까운 문서 청크**를 찾으면 됩니다. 이게 벡터 DB가 하는 일이에요 — "좌표 간 거리(유사도) 기반 검색".

비유하면, 일반 DB 검색이 "사전에서 단어 철자로 찾기"라면, 벡터 검색은 "뜻이 비슷한 단어를 의미의 지도 위에서 가까운 곳에서 찾기"입니다.

## 그래서 벡터 DB에 저장되는 실제 형태

위 FAQ 문서는 이런 식으로 변환되어 저장됩니다. 각 행(레코드)이 하나의 청크예요.

```
[id]  [원본 텍스트(content)]                          [벡터(embedding)]              [메타데이터]
 1    "Tier status is evaluated on January 1st..."   [0.12, -0.34, ...]   {section: "membership tiers"}
 2    "Points are earned at 1%/3%/5%..."             [0.45, 0.20, ...]    {section: "loyalty points"}
 3    "Go to Settings > Privacy > Delete..."         [-0.30, 0.61, ...]   {section: "delete account"}
 ...
```

여기서 중요한 점 두 가지:

- **원본 텍스트도 같이 저장됩니다.** 벡터는 "검색용 색인"일 뿐이고, 실제로 모델 프롬프트에 붙는 건 원본 텍스트(content)입니다. 벡터만 저장하는 게 아니에요.
- **메타데이터를 붙일 수 있습니다.** 섹션명, 출처, 날짜 등을 함께 저장해 두면 나중에 "이 답의 출처는 멤버십 섹션" 같은 필터링·인용에 씁니다.

## 청킹 — 당신 문서에서 가장 중요한 결정

지금 문서를 보면 `###` 헤더로 질문 단위가 깔끔하게 나뉘어 있죠. 이게 **청킹(chunking)**, 즉 문서를 어떤 단위로 쪼개 저장할지의 문제로 직결됩니다. 그리고 RAG 품질은 여기서 갈립니다.

당신 문서의 경우 선택지를 보면:

**나쁜 방법 — 글자 수로 기계적 절단**

```
"...Birthday double-points | VIP | Spend ≥ 800,000" [여기서 잘림]
"won/year | Priority on all orders..." [다음 청크]
```

표 중간이나 문장 중간이 잘려서 맥락이 깨집니다. 검색돼도 모델이 제대로 못 읽어요.

**좋은 방법 — 의미 단위(`###` 헤더)로 분할**

```
청크 1 = "What are the membership tiers..." 질문 + 그 답변(표 전체) 통째로
청크 2 = "How does the loyalty point system work?" 질문 + 답변 통째로
청크 3 = "How do I permanently delete my account?" 질문 + 답변 통째로
```

이렇게 하면 한 청크가 하나의 완결된 Q&A라, "계정 삭제하려면?"이라고 물었을 때 청크 3이 통째로 검색돼 완전한 답이 나옵니다.

당신 문서는 이미 FAQ 구조라 **헤더 기준 분할이 자연스럽게 맞습니다.** 이게 큰 이점이에요 — 잘 구조화된 문서는 청킹이 쉽습니다. Spring AI에는 마크다운 구조를 인식하는 리더와 토큰 단위 분할기가 있어서, `###` 단위를 살려 자를 수 있습니다.

한 가지 주의: 표(table)는 벡터화·청킹에서 까다로운 편입니다. 위 멤버십 표처럼 행마다 의미가 다른 경우, 표 전체를 한 청크로 두되 답변 텍스트에 표 맥락이 살아있도록 두는 게 무난합니다.

## 임베딩 모델은 한 번 정하면 고정

실무에서 자주 놓치는 점 하나. **저장할 때 쓴 임베딩 모델과 검색할 때 쓰는 임베딩 모델은 반드시 같아야 합니다.** 모델이 다르면 좌표계 자체가 달라서 거리 계산이 무의미해져요. 그리고 모델마다 벡터 차원 수(예: OpenAI `text-embedding-3-small`은 1536차원)가 정해져 있어서, 벡터 DB 스키마도 그 차원에 맞춰 만들어집니다. 나중에 임베딩 모델을 바꾸면 **전체 문서를 다시 임베딩해서 적재**해야 합니다.

## 어떤 벡터 DB를 고르나

당신 상황(FAQ 문서 기반 챗봇)에 맞춰 정리하면:

학습·프로토타입용으로는 `SimpleVectorStore`가 좋습니다. 별도 DB 설치 없이 메모리(또는 파일)에서 도는 Spring AI 내장 구현이라, 위 FAQ 정도 규모면 충분하고 바로 시작할 수 있습니다. 문서가 적고 자주 안 바뀌면 운영에서도 의외로 쓸 만해요.

운영·확장용으로는 보통 **PGVector**를 첫 후보로 둡니다. Postgres 확장이라 이미 Postgres를 쓰고 있다면 별도 인프라 없이 벡터 검색을 더할 수 있고, 메타데이터 필터링·트랜잭션 같은 RDB 장점을 그대로 씁니다. 그 외에 Redis(이미 캐시로 쓴다면), Qdrant·Milvus·Weaviate(벡터 검색 전문, 대규모) 등이 있는데, 처음부터 전문 벡터 DB로 갈 필요는 거의 없습니다.

당신 케이스라면 **`SimpleVectorStore`로 시작 → 잘 되면 PGVector로 이전**이 가장 무난한 경로입니다. FAQ 문서는 양이 많지 않고 변경도 잦지 않으니, 처음부터 무거운 벡터 DB를 세울 이유가 없어요.

## 정리

벡터 DB의 본질은 "텍스트를 의미 좌표(벡터)로 바꿔 저장하고, 질문과 의미적으로 가까운 청크를 거리 계산으로 찾아주는 DB"입니다. 당신처럼 잘 구조화된 FAQ가 있으면 절반은 끝난 셈이에요 — `###` 단위 청킹이 자연스럽고, 메타데이터로 섹션명을 붙이기도 쉽습니다.

다음으로 궁금한 게 임베딩·적재를 Spring AI에서 실제 코드로 어떻게 짜는지(ETL 파이프라인)인가요, 아니면 벡터 DB 선택을 더 깊게 비교해보고 싶으신가요? 어느 쪽이든 이 FAQ 문서를 그대로 예시로 이어서 잡아드릴게요.

---
# ETL 파이프라인은 이렇게 흐릅니다.

```
FAQ.md 파일
  → [Extract]   DocumentReader가 읽어 Document로
  → [Transform] TextSplitter가 청크로 분할 (+ 메타데이터)
  → [Load]      vectorStore.add() — 내부에서 임베딩 후 저장
```

## 0. 준비: 의존성과 임베딩 모델 설정

먼저 임베딩 모델이 있어야 합니다. `vectorStore.add()`가 내부에서 이 모델을 호출하니까요. OpenAI 임베딩을 쓴다고 가정하면, 스타터 의존성과 설정이 필요합니다.

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small   # 1536차원, 저렴하고 무난
```

이러면 `EmbeddingModel` 빈이 자동 구성됩니다. 그리고 벡터 스토어 빈을 만듭니다(앞서 본 그대로).

```java
@Bean
VectorStore vectorStore(EmbeddingModel embeddingModel) {
    return SimpleVectorStore.builder(embeddingModel).build();
}
```

문서 파일은 `src/main/resources/docs/faq.md`에 두었다고 하겠습니다.

## 1. Extract — 파일을 Document로 읽기

원본 FAQ를 읽어들이는 단계입니다. 당신 문서는 마크다운이니 두 가지 선택지가 있어요.

**선택지 A: 마크다운 구조를 인식하는 리더 (`MarkdownDocumentReader`)**

이게 당신 문서에 잘 맞습니다. 마크다운의 헤더·구조를 이해해서 읽어줍니다.

```java
Resource faqResource = new ClassPathResource("docs/faq.md");

MarkdownDocumentReaderConfig config = MarkdownDocumentReaderConfig.builder()
        .withHorizontalRuleCreateDocument(true)   // --- 구분선마다 문서 분리
        .withIncludeCodeBlock(false)
        .withIncludeBlockquote(false)
        .build();

MarkdownDocumentReader reader = new MarkdownDocumentReader(faqResource, config);
List<Document> documents = reader.get();
```

**선택지 B: 단순 텍스트 리더 (`TextReader`)**

구조 무시하고 통째로 읽습니다. 분할은 다음 단계(Transform)에 전적으로 맡길 때 씁니다.

```java
TextReader reader = new TextReader(new ClassPathResource("docs/faq.md"));
List<Document> documents = reader.get();
```

당신 FAQ는 `###` 질문 단위가 명확하니 마크다운 리더가 자연스럽지만, 입문 단계에선 `TextReader` + 다음 단계의 splitter 조합이 더 직관적이고 제어하기 쉽습니다. 아래는 그 조합으로 이어가겠습니다.

## 2. Transform — 청크로 분할

읽어온 `Document`를 검색에 적합한 크기의 청크로 쪼갭니다. Spring AI 기본 분할기는 `TokenTextSplitter`로, 글자 수가 아니라 **토큰 수** 기준으로 자릅니다(임베딩 모델이 토큰 단위로 동작하니 합리적이에요).

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
        .withChunkSize(400)          // 청크당 목표 토큰 수
        .withMinChunkSizeChars(350)  // 이 글자 수 넘으면 문장 경계에서 자름
        .withMaxNumChunks(10000)
        .build();

List<Document> chunks = splitter.split(documents);
```

여기서 당신 문서 특성상 주의할 점이 있습니다. 앞서 말했듯 **FAQ는 `###` 단위로 자르는 게 이상적**인데, `TokenTextSplitter`는 의미가 아니라 토큰 수로 자르므로 질문 중간이나 표 중간이 잘릴 수 있어요. 멤버십 표(`| Tier | ... |`)가 끊기면 검색 품질이 떨어집니다.

그래서 잘 구조화된 FAQ라면, 토큰 분할기에 의존하기보다 **`###` 단위로 직접 쪼개는** 편이 품질이 좋습니다. 이 방식이면 메타데이터(섹션명, 출처)도 같이 붙이기 좋고요.

```java
// faq.md 전체 텍스트를 ### 기준으로 직접 분할
String fullText = faqResource.getContentAsString(StandardCharsets.UTF_8);

String[] sections = fullText.split("(?=^### )");  // ### 앞에서 분리 (헤더 보존)

List<Document> chunks = Arrays.stream(sections)
        .map(String::trim)
        .filter(s -> s.startsWith("### "))         // 헤더 없는 앞부분(제목 등) 제외
        .map(section -> {
            // 첫 줄(### 질문)을 메타데이터로 추출
            String question = section.lines().findFirst().orElse("").replace("### ", "");
            return new Document(section, Map.of(
                    "source", "cholog-faq",
                    "section", question
            ));
        })
        .toList();
```

이러면 각 청크가 "질문 + 답변(표 포함)"을 통째로 담은 완결 단위가 되고, `section` 메타데이터로 어느 질문인지도 알 수 있습니다. 표가 중간에 잘릴 일도 없어요.

> 둘 중 무엇을 쓰든 결과물은 `List<Document>`로 동일하니, 다음 Load 단계는 그대로입니다. 입문이면 `TokenTextSplitter`로 빠르게 돌려본 뒤, 표가 깨지는 게 보이면 위 수동 분할로 바꾸는 순서를 권합니다.

## 3. Load — 임베딩 + 저장

마지막 단계입니다. `vectorStore.add()` 한 줄이 **임베딩과 저장을 동시에** 합니다. 앞서 설명한 대로, 이 시점에 각 청크가 임베딩 모델로 보내져 벡터가 되고 메모리에 저장돼요.

```java
vectorStore.add(chunks);
// 내부 동작: 각 chunk 텍스트 → EmbeddingModel 호출 → 벡터 받음 → 메모리 저장
```

`SimpleVectorStore`는 휘발성이니, 임베딩 API를 매 기동마다 다시 호출하지 않으려면 파일로 저장해둡니다.

```java
vectorStore.save(new File("faq-vectors.json"));
```

## 전체를 하나로 — 앱 시작 시 실행

위 세 단계를 `ApplicationRunner`로 묶어 앱 기동 시 한 번 돌리는 형태입니다. 이미 저장된 벡터 파일이 있으면 임베딩을 건너뛰고 불러오도록 분기를 넣었습니다(임베딩 비용 절약).

```java
@Configuration
public class FaqIngestionConfig {

    private static final File VECTOR_FILE = new File("faq-vectors.json");

    @Bean
    ApplicationRunner ingestFaq(VectorStore vectorStore) {
        return args -> {
            // 이미 적재된 벡터 파일이 있으면 임베딩 재호출 없이 복원
            if (VECTOR_FILE.exists()) {
                ((SimpleVectorStore) vectorStore).load(VECTOR_FILE);
                return;
            }

            // --- Extract ---
            Resource faqResource = new ClassPathResource("docs/faq.md");
            String fullText = faqResource.getContentAsString(StandardCharsets.UTF_8);

            // --- Transform (### 단위 분할 + 메타데이터) ---
            List<Document> chunks = Arrays.stream(fullText.split("(?=^### )", -1))
                    .map(String::trim)
                    .filter(s -> s.startsWith("### "))
                    .map(section -> {
                        String question = section.lines().findFirst().orElse("").replace("### ", "");
                        return new Document(section, Map.of(
                                "source", "cholog-faq",
                                "section", question
                        ));
                    })
                    .toList();

            // --- Load (임베딩 + 저장) ---
            vectorStore.add(chunks);
            ((SimpleVectorStore) vectorStore).save(VECTOR_FILE);
        };
    }
}
```

## 이게 챗봇과 연결되는 지점

ETL은 여기까지입니다. 한 번 적재해두면, 앞서 컨트롤러에 걸었던 `QuestionAnswerAdvisor`가 이 `vectorStore`를 자동으로 검색합니다. **ETL 코드와 챗 호출 코드는 완전히 분리**돼요 — 적재는 시작 시 한 번, 검색은 매 질문마다, 서로 건드리지 않습니다.

```
[앱 시작] FaqIngestionConfig → vectorStore에 FAQ 적재 (한 번)
[매 요청] 사용자 질문 → QuestionAnswerAdvisor가 vectorStore 검색 → 관련 청크 첨부 → 답변
```

## 짚어둘 실무 포인트

몇 가지 미리 알아두면 좋은 것들:

분할 방식은 정답이 없고 실측으로 정합니다. 위 `###` 수동 분할을 권하긴 했지만, 실제 질문들을 던져보고 "원하는 청크가 검색되는지" 확인하며 조정하는 게 맞습니다. 청크가 너무 크면 관련 없는 내용이 섞이고, 너무 작으면 맥락이 끊겨요.

`(SimpleVectorStore)` 캐스팅이 거슬릴 수 있는데, `save/load`가 `VectorStore` 공통 인터페이스가 아니라 `SimpleVectorStore` 전용 메서드라 그렇습니다. PGVector로 이전하면 이 저장/복원 로직 자체가 불필요해지니(DB가 영속화하니까) 그때 들어냅니다. 이게 앞서 말한 "이전 시 바뀌는 부분"의 한 예예요.

임베딩 비용은 적재할 때 청크 수만큼 발생합니다. 당신 FAQ는 수십 청크 규모라 미미하지만, 문서가 커지면 무시 못 합니다. 그래서 `save/load`로 재호출을 막는 게 중요해요.

다음 단계로는, 적재된 결과로 실제 검색이 잘 되는지 확인하는 방법(검색 결과를 직접 찍어보거나 `QuestionAnswerAdvisor`의 `topK`·유사도 임계값 튜닝)을 보는 게 자연스럽습니다. 그쪽으로 이어갈까요?