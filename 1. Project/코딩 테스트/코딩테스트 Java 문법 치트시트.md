

## 0. 기본 뼈대

프로그래머스는 `solution` 메서드만 채우면 되므로 import 한 줄과 자료형 범위, 문자열 처리만 확실하면 됩니다.

```java
import java.util.*;
import java.util.stream.*;

class Solution {
    public int[] solution(int[] arr) { ... }
}
```

| 타입 | 범위 | 언제 |
| --- | --- | --- |
| int | 약 ±21억 (2.1×10^9) | 기본 |
| long | 약 ±9.2×10^18 | 곱셈·누적합·이분탐색 범위가 10^9 넘을 때 |
| Integer.MAX\_VALUE | 2147483647 | 최솟값 초기화, INF |

```java
// String 기본
String s = "hello";
s.length();              // 배열은 arr.length (괄호 X)
s.charAt(0);             // 'h'
s.substring(1, 3);       // "el" (끝 인덱스 미포함)
s.indexOf("l");          // 2, 없으면 -1
s.contains("ell");
s.startsWith("he");
s.split(" ");            // String[]
s.toCharArray();         // char[]
s.equals(t);             // == 쓰지 말 것
s.compareTo(t);          // 사전순 비교
s.replace("l", "L");
s.toUpperCase(); s.toLowerCase();
String.join(",", list);
String.valueOf(123);     // "123"
Integer.parseInt("123"); // 123

// char 연산
char c = 'a';
c - 'a';                 // 0 (알파벳 인덱스)
c - '0';                 // 숫자 문자 → int
(char)('a' + 1);         // 'b'
Character.isDigit(c); Character.isLetter(c); Character.isUpperCase(c);

// StringBuilder: 반복문에서 문자열 누적할 때 필수
StringBuilder sb = new StringBuilder();
sb.append("a").append(1);
sb.insert(0, "x");
sb.reverse();
sb.deleteCharAt(sb.length() - 1);
sb.setCharAt(0, 'z');
sb.toString();
```

## 1. Arrays / Collections / List

배열은 `Arrays`, 리스트는 `Collections`로 다룹니다. 반환 타입이 `int[]`인 문제가 많아 List ↔ 배열 변환이 핵심입니다.

```java
// 배열 생성·초기화
int[] arr = new int[n];
int[][] grid = new int[n][m];
Arrays.fill(arr, -1);
for (int[] row : grid) Arrays.fill(row, Integer.MAX_VALUE); // 2차원은 행마다

// Arrays
Arrays.sort(arr);                        // 오름차순 (int[]는 내림차순 불가)
Arrays.sort(arr, 1, 4);                  // [1, 4) 구간만
Arrays.toString(arr);                    // 디버깅 출력 "[1, 2, 3]"
Arrays.deepToString(grid);               // 2차원 출력
int[] copy = Arrays.copyOf(arr, arr.length);
int[] part = Arrays.copyOfRange(arr, 1, 3);
int[][] g2 = Arrays.stream(grid).map(int[]::clone).toArray(int[][]::new); // 2차원 깊은 복사
Arrays.equals(a, b);
Arrays.binarySearch(arr, key);           // 정렬된 배열에서만
List<Integer> list = Arrays.asList(1, 2, 3); // 고정 크기! add/remove 불가

// List
List<Integer> list = new ArrayList<>();
list.add(x); list.add(0, x);             // 맨 앞 삽입 O(n)
list.get(i); list.set(i, x);
list.remove(i);                          // 인덱스로 삭제 (int)
list.remove(Integer.valueOf(x));         // 값으로 삭제
list.contains(x); list.indexOf(x);
list.size(); list.isEmpty();
new ArrayList<>(other);                  // 복사

// Collections
Collections.sort(list);
Collections.sort(list, Collections.reverseOrder());
Collections.reverse(list);
Collections.max(list); Collections.min(list);
Collections.swap(list, i, j);
Collections.frequency(list, x);
```

## 2. 해시 (출제 빈도 높음)

카운팅은 `getOrDefault` 또는 `merge`, 존재 여부는 `HashSet`. 조회·삽입 평균 O(1).

```java
Map<String, Integer> map = new HashMap<>();
map.put(k, v);
map.get(k);                               // 없으면 null → int 언박싱 시 NPE 주의
map.getOrDefault(k, 0);
map.put(k, map.getOrDefault(k, 0) + 1);   // 카운팅 정석
map.merge(k, 1, Integer::sum);            // 카운팅 한 줄
map.containsKey(k); map.containsValue(v);
map.remove(k);
map.putIfAbsent(k, v);
map.computeIfAbsent(k, x -> new ArrayList<>()).add(v); // 그룹핑 (Map<K, List<V>>)

// 순회
for (Map.Entry<String, Integer> e : map.entrySet()) {
    e.getKey(); e.getValue();
}
for (String key : map.keySet()) {}
for (int val : map.values()) {}

// value 기준 정렬 (베스트앨범류)
List<String> keys = new ArrayList<>(map.keySet());
keys.sort((a, b) -> map.get(b) - map.get(a));   // value 내림차순

// Set
Set<Integer> set = new HashSet<>();
set.add(x);            // 중복이면 false 반환
set.contains(x);
set.remove(x);
set.size();
new HashSet<>(list);   // 중복 제거

// 순서가 필요할 때
new LinkedHashMap<>(); // 삽입 순서 유지
new TreeMap<>();       // 키 정렬 (firstKey, lastKey, floorKey, ceilingKey)
new TreeSet<>();       // 정렬된 Set (first, last, floor, ceiling)
```

대표 유형: 완주하지 못한 선수(카운팅), 전화번호 목록(접두어 → Set에 전부 넣고 prefix 검사), 의상(종류별 개수 → (개수+1) 곱 − 1), 베스트앨범(그룹핑 + 정렬).

## 3. 스택 / 큐

스택·큐·덱 모두 `ArrayDeque` 하나로 씁니다. `Stack` 클래스는 Vector 기반이라 느리고, 메서드 이름이 헷갈리면 스택은 push/pop/peek, 큐는 offer/poll/peek만 기억하면 됩니다.

```java
// 스택 (LIFO)
Deque<Integer> stack = new ArrayDeque<>();
stack.push(x);        // 맨 앞에 넣기
stack.pop();          // 꺼내기 (비어있으면 예외)
stack.peek();         // 보기 (비어있으면 null)
stack.isEmpty();

// 큐 (FIFO)
Queue<Integer> q = new ArrayDeque<>();
q.offer(x);           // 뒤에 넣기
q.poll();             // 앞에서 꺼내기 (비어있으면 null)
q.peek();
q.size();

// 덱 (양쪽)
Deque<Integer> dq = new ArrayDeque<>();
dq.offerFirst(x); dq.offerLast(x);
dq.pollFirst();  dq.pollLast();
dq.peekFirst();  dq.peekLast();

// 여러 값을 함께 넣을 때
Queue<int[]> q2 = new ArrayDeque<>();
q2.offer(new int[]{idx, priority});
int[] cur = q2.poll();
```

| 패턴 | 도구 | 예시 문제 |
| --- | --- | --- |
| 짝 맞추기 / 직전 값 비교 | 스택 | 올바른 괄호, 같은 숫자는 싫어 |
| 단조 스택 (다음 큰 수) | 스택에 인덱스 저장 | 주식가격 |
| 순서대로 처리·시뮬레이션 | 큐 | 기능개발, 다리를 지나는 트럭, 프로세스 |

`ArrayDeque`는 null을 넣을 수 없습니다. 원시 int 비교 시 `stack.peek() == x` 대신 `stack.peek().equals(x)` 또는 `int top = stack.peek();`로 언박싱 후 비교하세요 (Integer 캐시 범위 −128\~127 밖에서 `==`는 틀림).

## 4. 힙 (PriorityQueue)

`PriorityQueue`는 기본이 최소 힙이고, 삽입·삭제 O(log n), 최솟값 조회 O(1)입니다.

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

pq.offer(x);
pq.poll();     // 최솟값(최댓값) 꺼내기, 비면 null
pq.peek();
pq.size(); pq.isEmpty();
pq.remove(x);  // 특정 값 삭제 O(n) - 이중우선순위큐에서 사용

// 배열 원소 기준
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);      // [1] 오름차순
PriorityQueue<int[]> pq2 = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));
PriorityQueue<int[]> pq3 = new PriorityQueue<>((a, b) ->
        a[0] != b[0] ? a[0] - b[0] : b[1] - a[1]);  // [0] 오름, 같으면 [1] 내림

// long 값이면 뺄셈 대신 compare (오버플로 방지)
PriorityQueue<long[]> pql = new PriorityQueue<>((a, b) -> Long.compare(a[0], b[0]));

// 한 번에 넣기
PriorityQueue<Integer> pq4 = new PriorityQueue<>(list);
```

**주의:** `for (int x : pq)` 순회나 `toString()`은 정렬 순서가 아닙니다. 정렬 순서로 보려면 poll로 꺼내야 합니다.

대표 유형: 더 맵게(최소 2개 꺼내 섞기), 디스크 컨트롤러(요청 시간순 정렬 + 소요 시간 최소 힙), 이중우선순위큐(최소·최대 힙 2개 또는 `TreeMap`).

## 5. 정렬 (출제 빈도 높음)

원시 배열 `int[]`는 오름차순만 가능하고, 내림차순이나 커스텀 기준은 `Integer[]`, `int[][]`, `List`에서 Comparator로 합니다.

```java
// int[] 내림차순: 오름차순 후 뒤집거나 박싱
Integer[] boxed = Arrays.stream(arr).boxed().toArray(Integer[]::new);
Arrays.sort(boxed, Collections.reverseOrder());

// 2차원 배열
Arrays.sort(arr2d, (a, b) -> a[0] - b[0]);                       // [0] 오름차순
Arrays.sort(arr2d, (a, b) -> a[0] == b[0] ? a[1] - b[1] : a[0] - b[0]);
Arrays.sort(arr2d, Comparator.comparingInt((int[] a) -> a[0])
                             .thenComparingInt(a -> a[1]));      // 체이닝

// Comparator 조합
Comparator.comparing(Person::getName);
Comparator.comparing(Person::getAge).reversed();
Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder());
Comparator.reverseOrder();

// 문자열
String[] strs = ...;
Arrays.sort(strs);                                    // 사전순
Arrays.sort(strs, (a, b) -> b.compareTo(a));          // 사전 역순
Arrays.sort(strs, (a, b) -> a.length() - b.length()); // 길이순

// 문자열 자체를 정렬 (애너그램 등)
char[] cs = s.toCharArray();
Arrays.sort(cs);
String sorted = new String(cs);

// 가장 큰 수: 이어붙였을 때 큰 순
Arrays.sort(strs, (a, b) -> (b + a).compareTo(a + b));
```

| 반환값 | 의미 |
| --- | --- |
| 음수 | a가 앞 |
| 0 | 같음 (Arrays.sort 객체 정렬은 안정 정렬이라 기존 순서 유지) |
| 양수 | b가 앞 |

`a - b`는 값이 ±10^9 근처면 오버플로가 나므로 그땐 `Integer.compare(a, b)`를 씁니다. H-Index는 정렬 후 `citations[n - 1 - i] >= i + 1` 조건으로 셉니다.

## 6. 완전탐색 (출제 빈도 높음, 평균 점수 낮음)

Java엔 Python의 `itertools`가 없으니 순열·조합·부분집합 백트래킹 템플릿 3개는 외워 두세요. n ≤ 10이면 순열(10! ≈ 360만), n ≤ 20이면 부분집합(2^20 ≈ 100만)이 가능합니다.

```java
// 순열: n개 중 r개를 순서 있게 (소수 찾기, 피로도)
boolean[] visited = new boolean[n];
void perm(int[] arr, int[] out, int depth, int r) {
    if (depth == r) { /* out 사용 */ return; }
    for (int i = 0; i < arr.length; i++) {
        if (visited[i]) continue;
        visited[i] = true;
        out[depth] = arr[i];
        perm(arr, out, depth + 1, r);
        visited[i] = false;
    }
}

// 조합: 순서 없이 r개
void comb(int[] arr, int[] out, int start, int depth, int r) {
    if (depth == r) { return; }
    for (int i = start; i < arr.length; i++) {
        out[depth] = arr[i];
        comb(arr, out, i + 1, depth + 1, r);
    }
}

// 부분집합: 넣는다/안 넣는다
void subset(int idx, int sum) {
    if (idx == n) { return; }
    subset(idx + 1, sum + arr[idx]);
    subset(idx + 1, sum);
}

// 비트마스크 부분집합
for (int mask = 0; mask < (1 << n); mask++) {
    for (int i = 0; i < n; i++) {
        if ((mask & (1 << i)) != 0) { /* i 포함 */ }
    }
}

// 소수 판별 O(√n)
boolean isPrime(int x) {
    if (x < 2) return false;
    for (int i = 2; (long) i * i <= x; i++) if (x % i == 0) return false;
    return true;
}

// 약수 쌍 (카펫: 가로 >= 세로)
for (int h = 1; h * h <= total; h++) {
    if (total % h == 0) { int w = total / h; }
}
```

**문자열로 순열 만들 때** (소수 찾기): `StringBuilder` 이어붙인 뒤 `Integer.parseInt`, 중복 숫자는 `Set<Integer>`로 제거합니다. 모음사전처럼 사전순 생성은 DFS에서 `list.add(cur)` 후 다음 글자를 붙이면 순서대로 쌓입니다.

## 7. 탐욕법 (Greedy)

새 문법은 거의 없고 "정렬 + 투 포인터 + 카운팅" 조합이 대부분입니다.

```java
// 투 포인터 (구명보트: 가장 가벼운 + 가장 무거운)
Arrays.sort(people);
int l = 0, r = people.length - 1, cnt = 0;
while (l <= r) {
    if (people[l] + people[r] <= limit) l++;
    r--; cnt++;
}

// 구간 정렬 (단속카메라: 끝점 기준)
Arrays.sort(routes, (a, b) -> a[1] - b[1]);
int cam = Integer.MIN_VALUE, count = 0;
for (int[] r : routes) {
    if (r[0] > cam) { cam = r[1]; count++; }
}

// 큰 수 만들기: 스택으로 앞자리 최대화
StringBuilder sb = new StringBuilder();
for (char c : number.toCharArray()) {
    while (k > 0 && sb.length() > 0 && sb.charAt(sb.length() - 1) < c) {
        sb.deleteCharAt(sb.length() - 1); k--;
    }
    sb.append(c);
}
sb.setLength(sb.length() - k);  // 남은 k만큼 뒤에서 제거

// 조이스틱: 알파벳 최소 이동
Math.min(c - 'A', 'Z' - c + 1);
```

섬 연결하기는 그리디이지만 실제로는 크루스칼(그래프 섹션의 유니온 파인드)로 풉니다.

## 8. 동적계획법 (DP)

배열 선언·초기화와 모듈러 연산이 문법의 전부입니다. 점화식을 먼저 주석으로 쓰고 코드를 채우세요.

```java
// Bottom-up
int[] dp = new int[n + 1];
dp[0] = 0; dp[1] = 1;
for (int i = 2; i <= n; i++) dp[i] = (dp[i - 1] + dp[i - 2]) % 1_000_000_007;

// 2차원 (정수 삼각형: 위에서 내려오며 최대)
int[][] dp2 = new int[h][];
for (int i = 0; i < h; i++) dp2[i] = new int[i + 1];   // 가변 길이 배열

// 격자 경로 (등굣길: 물웅덩이 = -1 표시 후 skip)
dp[i][j] = (dp[i - 1][j] + dp[i][j - 1]) % MOD;

// Top-down 메모이제이션
int[] memo = new int[n + 1];
Arrays.fill(memo, -1);
int f(int x) {
    if (x <= 1) return x;
    if (memo[x] != -1) return memo[x];
    return memo[x] = f(x - 1) + f(x - 2);
}

// 원형 배열 (도둑질): 첫 집 포함 / 미포함 두 번 DP
// 집합 DP (N으로 표현): List<Set<Integer>> dp, dp[i] = dp[j] ⊕ dp[i-j]
List<Set<Integer>> sets = new ArrayList<>();
for (int i = 0; i <= 8; i++) sets.add(new HashSet<>());
```

**모듈러:** 덧셈마다 `% MOD`, 곱셈은 `(long) a * b % MOD`. 재귀 깊이가 1만 이상이면 StackOverflow 위험이 있으니 Bottom-up으로 바꿉니다.

## 9. DFS / BFS (출제 빈도 높음, 평균 점수 낮음)

최단 거리(가중치 없음)는 BFS, 모든 경우·연결 요소는 DFS. 방문 체크는 **큐에 넣을 때** 해야 중복 삽입이 없습니다.

```java
// 방향 벡터 (상하좌우)
int[] dx = {-1, 1, 0, 0};
int[] dy = {0, 0, -1, 1};

// 격자 BFS 최단거리 (게임 맵 최단거리)
int bfs(int[][] map) {
    int n = map.length, m = map[0].length;
    int[][] dist = new int[n][m];
    Queue<int[]> q = new ArrayDeque<>();
    q.offer(new int[]{0, 0});
    dist[0][0] = 1;
    while (!q.isEmpty()) {
        int[] cur = q.poll();
        for (int d = 0; d < 4; d++) {
            int nx = cur[0] + dx[d], ny = cur[1] + dy[d];
            if (nx < 0 || ny < 0 || nx >= n || ny >= m) continue;
            if (map[nx][ny] == 0 || dist[nx][ny] != 0) continue;
            dist[nx][ny] = dist[cur[0]][cur[1]] + 1;
            q.offer(new int[]{nx, ny});
        }
    }
    return dist[n - 1][m - 1] == 0 ? -1 : dist[n - 1][m - 1];
}

// 인접 행렬 DFS (네트워크: 연결 요소 개수)
void dfs(int[][] computers, boolean[] visited, int v) {
    visited[v] = true;
    for (int i = 0; i < computers.length; i++)
        if (computers[v][i] == 1 && !visited[i]) dfs(computers, visited, i);
}

// 경우의 수 DFS (타겟 넘버)
int dfs(int[] nums, int idx, int sum, int target) {
    if (idx == nums.length) return sum == target ? 1 : 0;
    return dfs(nums, idx + 1, sum + nums[idx], target)
         + dfs(nums, idx + 1, sum - nums[idx], target);
}

// 문자열 상태 BFS (단어 변환): 방문은 Set<String> 또는 boolean[]
// 한 글자 차이 판별
int diff = 0;
for (int i = 0; i < a.length(); i++) if (a.charAt(i) != b.charAt(i)) diff++;

// 경로 백트래킹 (여행경로): 티켓 사전순 정렬 후 DFS, 첫 완성 경로가 답
Arrays.sort(tickets, (a, b) -> a[0].equals(b[0]) ? a[1].compareTo(b[1]) : a[0].compareTo(b[0]));
```

**아이템 줍기**처럼 좌표 테두리를 따라가는 문제는 좌표를 2배로 늘려 붙은 선이 끊기지 않게 한 뒤 BFS, 결과를 2로 나눕니다.

## 10. 이분탐색

프로그래머스 이분탐색은 대부분 "답(시간·거리)을 정해 놓고 가능한지 검사"하는 파라메트릭 서치이고, 범위가 10^18까지 가므로 **전부 long**으로 씁니다.

```java
// 파라메트릭 서치: 조건을 만족하는 최솟값 (입국심사)
long lo = 1, hi = (long) maxTime * n, ans = hi;
while (lo <= hi) {
    long mid = lo + (hi - lo) / 2;      // 오버플로 방지
    long cnt = 0;
    for (int t : times) cnt += mid / t;
    if (cnt >= n) { ans = mid; hi = mid - 1; }  // 가능 → 더 작게
    else lo = mid + 1;
}

// 최댓값 찾기 (징검다리: 최소 거리의 최댓값)
if (possible(mid)) { ans = mid; lo = mid + 1; } else hi = mid - 1;

// lower_bound: target 이상 첫 위치
int lowerBound(int[] a, int target) {
    int lo = 0, hi = a.length;
    while (lo < hi) {
        int mid = (lo + hi) >>> 1;
        if (a[mid] < target) lo = mid + 1; else hi = mid;
    }
    return lo;
}
// upper_bound: a[mid] <= target 로 바꾸면 target 초과 첫 위치
```

`Arrays.binarySearch`는 중복 값이 있을 때 어떤 인덱스를 줄지 보장하지 않으므로, 개수·경계가 필요하면 위 lower/upper bound를 직접 씁니다.

## 11. 그래프

노드 번호가 1부터면 크기를 `n + 1`로 잡는 게 가장 흔한 실수 방지책입니다. 간선 리스트로 주어지면 인접 리스트로 바꾸고 시작하세요.

```java
// 인접 리스트
List<List<Integer>> graph = new ArrayList<>();
for (int i = 0; i <= n; i++) graph.add(new ArrayList<>());
for (int[] e : edges) {
    graph.get(e[0]).add(e[1]);
    graph.get(e[1]).add(e[0]);   // 무방향
}

// 가장 먼 노드: BFS 거리 배열 → 최댓값 개수
int[] dist = new int[n + 1];
Arrays.fill(dist, -1);

// 다익스트라 (가중치 있음)
List<List<int[]>> g = new ArrayList<>();   // {to, cost}
int[] d = new int[n + 1];
Arrays.fill(d, Integer.MAX_VALUE);
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
d[start] = 0;
pq.offer(new int[]{start, 0});
while (!pq.isEmpty()) {
    int[] cur = pq.poll();
    if (cur[1] > d[cur[0]]) continue;       // 이미 더 짧은 경로 확정
    for (int[] nx : g.get(cur[0])) {
        int nd = cur[1] + nx[1];
        if (nd < d[nx[0]]) { d[nx[0]] = nd; pq.offer(new int[]{nx[0], nd}); }
    }
}

// 유니온 파인드 (크루스칼: 섬 연결하기)
int[] parent = new int[n];
for (int i = 0; i < n; i++) parent[i] = i;
int find(int x) { return parent[x] == x ? x : (parent[x] = find(parent[x])); }
boolean union(int a, int b) {
    a = find(a); b = find(b);
    if (a == b) return false;
    parent[b] = a; return true;
}
Arrays.sort(costs, (a, b) -> a[2] - b[2]);
for (int[] c : costs) if (union(c[0], c[1])) total += c[2];

// 플로이드-워셜 (순위: 승패 전이, n ≤ 100)
for (int k = 1; k <= n; k++)
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= n; j++)
            if (win[i][k] && win[k][j]) win[i][j] = true;
```

`static`이 아닌 인스턴스 필드(`int[] parent;`)로 선언하면 `solution` 안에서 초기화해 다른 메서드에서 그대로 쓸 수 있습니다.

## 12. Stream 자주 쓰는 변환

Stream은 변환·집계를 한 줄로 끝낼 때만 쓰고, 반복 루프 안에서는 for문이 더 빠릅니다 (시간 초과 경계에서 Stream이 원인인 경우가 있음).

```java
// ── 배열 ↔ List ──
List<Integer> list = Arrays.stream(arr).boxed().collect(Collectors.toList());
int[] arr = list.stream().mapToInt(Integer::intValue).toArray();
Integer[] boxed = Arrays.stream(arr).boxed().toArray(Integer[]::new);
String[] sa = list.toArray(new String[0]);
List<String> sl = new ArrayList<>(Arrays.asList(sa));   // 수정 가능한 List

// ── 집계 ──
int sum = Arrays.stream(arr).sum();
int max = Arrays.stream(arr).max().getAsInt();
int min = Arrays.stream(arr).min().orElse(0);
double avg = Arrays.stream(arr).average().orElse(0);
long cnt = Arrays.stream(arr).filter(x -> x > 0).count();
long lsum = Arrays.stream(arr).asLongStream().sum();   // 합이 int 넘을 때

// ── 변환·필터·정렬 ──
int[] doubled = Arrays.stream(arr).map(x -> x * 2).toArray();
int[] evens   = Arrays.stream(arr).filter(x -> x % 2 == 0).toArray();
int[] uniq    = Arrays.stream(arr).distinct().toArray();
int[] sorted  = Arrays.stream(arr).sorted().toArray();
int[] desc    = Arrays.stream(arr).boxed().sorted(Comparator.reverseOrder())
                      .mapToInt(Integer::intValue).toArray();

// ── 범위 ──
int[] range = IntStream.range(0, n).toArray();         // 0..n-1
int[] rangeC = IntStream.rangeClosed(1, n).toArray();  // 1..n
IntStream.range(0, n).filter(i -> arr[i] == target).toArray(); // 인덱스 찾기

// ── 문자열 ──
String joined = list.stream().map(String::valueOf).collect(Collectors.joining(""));
String joined2 = Arrays.stream(arr).mapToObj(String::valueOf).collect(Collectors.joining(","));
int[] digits = String.valueOf(num).chars().map(c -> c - '0').toArray();
int[] fromStr = Arrays.stream("1 2 3".split(" ")).mapToInt(Integer::parseInt).toArray();
String rev = new StringBuilder(s).reverse().toString();

// ── 그룹핑·카운팅 ──
Map<String, Long> freq = Arrays.stream(words)
        .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
Map<String, List<String>> byType = Arrays.stream(clothes)
        .collect(Collectors.groupingBy(c -> c[1], Collectors.mapping(c -> c[0], Collectors.toList())));
Set<Integer> set = Arrays.stream(arr).boxed().collect(Collectors.toSet());

// ── Map 정렬해서 키 뽑기 ──
List<String> keys = map.entrySet().stream()
        .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());

// ── 2차원 ──
int total = Arrays.stream(grid).flatMapToInt(Arrays::stream).sum();
int[][] copy = Arrays.stream(grid).map(int[]::clone).toArray(int[][]::new);
```

| 원하는 것 | 핵심 연결 |
| --- | --- |
| int\[\] → List | `.boxed().collect(...)` |
| List\<Integer> → int\[\] | `.mapToInt(Integer::intValue).toArray()` |
| int\[\] → 문자열 | `.mapToObj(String::valueOf).collect(joining())` |
| 숫자 → 자릿수 배열 | `String.valueOf(n).chars().map(c -> c - '0')` |
| 카운팅 Map | `groupingBy(x -> x, counting())` (값이 Long) |

## 13. 제출 전 체크리스트

- [ ] 곱셈·누적합·이분탐색 범위가 int(약 21억)를 넘지 않는가 → `long`, `(long) a * b`
- [ ] 문자열·Integer 비교에 `==` 대신 `equals` 썼는가
- [ ] `Arrays.asList` 결과에 add/remove 하지 않았는가
- [ ] `list.remove(x)`가 인덱스 삭제로 동작하지 않는가 → `Integer.valueOf(x)`
- [ ] `map.get(k)`가 null일 때 언박싱 NPE가 나지 않는가 → `getOrDefault`
- [ ] Comparator에서 `a - b` 오버플로 가능성 → `Integer.compare`
- [ ] BFS 방문 체크를 큐에 넣을 때 했는가
- [ ] 노드 번호 1부터면 배열 크기 `n + 1`
- [ ] 반복문 안 문자열 누적은 `StringBuilder`
- [ ] 빈 입력, n = 1, 모두 같은 값 같은 경계 케이스 확인
- [ ] 디버깅 `System.out.println` 제거 (출력 많으면 시간 초과)
