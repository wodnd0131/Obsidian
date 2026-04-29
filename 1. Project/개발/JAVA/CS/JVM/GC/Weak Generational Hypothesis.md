#CS/JVM/Memory #CS/JVM/GC #Status/완료  

David Ungar와 Paul R. Wilson의 연구에서 나왔다.

- **David Ungar (1984)** — Generation Scavenging 논문에서 세대형 GC를 처음 제안하면서 이 관찰을 근거로 제시
- **Paul R. Wilson (1992)** — _Uniprocessor Garbage Collection Techniques_ 에서 다양한 프로그램을 실측해 이 패턴이 반복됨을 확인

즉, "대부분의 객체는 일찍 죽는다"는 말은 여러 프로그램의 객체 생존율을 실제로 측정한 결과에서 나온 관찰이다.

---

## 왜 "가설"이라는 단어를 쓰는가

측정 결과가 있어도 **"항상 성립한다"고 증명할 수 없기 때문**이다.

```java
// 이 가설이 성립하는 전형적인 코드
public List<String> processOrders(List<Order> orders) {
    return orders.stream()
        .map(o -> o.getItemName().toUpperCase()) // 중간 String 객체들 → 바로 죽음
        .collect(toList());
}

// 이 가설이 성립하지 않는 코드
private static final Map<Long, User> SESSION_CACHE = new ConcurrentHashMap<>();
// 한번 들어온 User 객체는 세션이 끊길 때까지 수십 분 ~ 수 시간 생존
```

워크로드 성격에 따라 가설이 깨진다. 그래서 "법칙"이 아니라 "가설"이다.

---

## JVM 설계에서의 위상

이 가설은 **설계 전제(design assumption)** 로 채택됐다.

가설이 성립한다고 가정하고 설계하면:

- Young Gen을 작게 유지하면서 자주 GC해도 대부분의 쓰레기가 여기서 처리됨
- Copying GC가 효율적 — 살아남는 객체가 적으니 복사 비용이 낮음
- Old Gen GC는 드물게만 발생

가설이 깨지면 (= 오래 사는 객체가 많으면):

- Young Gen에서 살아남는 객체가 많아짐 → Copying 비용 증가
- Old Gen 승격이 빨라짐 → Full GC 빈도 증가
- 세대형 GC의 이점이 사라짐

---

## 한 줄 정리

> 실측 기반의 경험적 가설이고, JVM GC 설계의 전제 조건이다.  
> 성립하면 세대형 GC가 효율적이고, 깨지면 G1GC나 ZGC 같은 다른 접근이 필요하다.