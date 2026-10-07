
![[../../../Repository/7주차 - DP.png]]

백트레킹으로 풀 수 있다.
재귀함수를 어떻게 설계해야할까

이 날에 일을 할 거냐 말거냐. ox 분기로 진행
2^N -> 3만 2천가지

퇴사 2는 N이 1,500,000
여기선 최적화가 필요함.

백트레킹 기본구조.
기저조건 쌓고 가지치기하고, 분기처리하기

이것과 유사하지만 다른 느낌의 코드,

매개함수의 정의가 다르다.
현재 CUIR 이고, 앞으로 일을 최선을 다해서 고를때 최대 얼마를 받을 수 있을까?


6일과 7일은 날짜상 불가능
5일부터 시작을 했을때, 최대 15원
4일부터는 35원

recur 4를 호출하면 35를 호출하는 걸 만들고싶다. 가 의도.
이 날짜(4일)부터 얻을 수 있는 최대값은 무엇이냐?

1->4->recur(5) =15
1->2->3->4->recur(5) =15

이럼 5일차는 항상 같은 15.
다시 볼 필요없다.
이를 dp[5]에 저장. 똑같은 값에 대하서는 항상같은 값을 리턴한다.
인자가 일정할때, 리턴값이 일정하다
-> 메모리제이션 => 탑 다운 DP

일반적으로 DP를 배우면, 바텀업을 많이 배운다.
=> 공부하다보면, 벽이 느껴진다.

탑다운 DP의 특징
1. 백트래킹 중하수 정도의 진입장벽
2. 근데? 실버~플레 체감 난이도가 모두 같다

---
배낭

![[../../../Repository/7주차 - DP-1.png]]

이걸 짜면, 바텀업으로 바꿀수있다.

#
# def recur(cur, total):
#     global answer
#     if cur > n:
#         return
#     if cur == n:
#         answer = max(answer, total)
#         return
#     # 일을 하거나
#     recur(cur + arr[cur][0], total + arr[cur][1])
#     # 하지 않거나
#     recur(cur + 1, total)
#
# # 메모이제이션
# def recur(cur): # 현재 cur일이고, 앞으로 일을 최선을 다해서 고를때 최대 얼마를 벌 수 있는지 리턴하는 함수
#     # 잘못 왔으면. 오지 말아야 할 곳에 왔으면? 절대 답이 안되게
#     if cur > n:
#         return -1289312983891239
#     if cur == n:
#         return 0
#     if dp[cur] != -1:
#         return dp[cur]
#
#     a = recur(cur+ arr[cur][0]) + arr[cur][1]
#     b = recur(cur + 1)
#     dp[cur] = max(a, b)
#     return dp[cur]
#
#
#
# n = int(input())
# arr = [list(map(int, input().split())) for _ in range(n)]
# answer = -1
# dp =  [-1] * n
#
# print(recur(0))

"""
recur(5) => 15
recur(4) => 35

인자가 일정할때 리턴값이 일정해야한다.

탑다운 DP

바텀업 DP
=> 벽이 느껴져

dp란.

1. 겹치는 부분문제
2. 최적 부분 구조


바텀업 푸는방법

1. 가짜 문제 정의
2. 가짜 문제를 통해 진짜 문제를 풀 수 있는지?
3. 초기값 설정
4. 점화식 도출
5. 진짜 문제 정답 출력

dp == 수학적 귀납법

---

탑다운 DP 특징 

1. 진입장벽이 높다 -> 백트래킹 중하수 정도. 기본적으로 짤줄 알아야해
2. 근데? 실버나 골드나 플레나 다이아나 그냥 체감 난이도 차이가 없음 그냥
익숙해지면 ?? dp == 백트래킹


"""


"""
DP 짜는법

1. 백트래킹 짠다
2. 리턴하는 방식으로 바꾼다 (처음부터 이렇게 해도 됨)
3. 메모이제이션
"""

n = int(input())
arr = [list(map(int, input().split())) for _ in range(n)]
answer = 123891238921389

def recur(cur, prev, total):
    global answer
    if cur == n:
        answer = min(answer, total)
        return

    for i in range(3):
        if i == prev:
            continue
        recur(cur + 1, i, total + arr[cur][i])

def recur(cur, prev): # 현재 cur번째 집을 칠해야하고, 직전 집을 prev색으로 칠했을때, 앞으로 최선을 다해서 집을 색칠했을때 드는 최소비용 을 리턴하는함수
    if cur == n:
        return 0

    if dp[cur][prev] != -1:
        return dp[cur][prev]

    ret = 123891238921389

    for i in range(3):
        if i == prev:
            continue
        ret = min(ret, recur(cur + 1, i) + arr[cur][i])

    dp[cur][prev] = ret
    return dp[cur][prev]


dp = [[-1] * 3 for _ in range(n)]
print(recur(0, -1))





"""

이거 dp로 풀어봐


이거 백트래킹 해봐


신입 공채 기준 코테

4문제

1번이 구현
2번이 백트래킹
3번이 대충 알고리즘 하나
4번 dp


0001 -> 1번집을 방문한상태 -> 1
1010 -> 2,4번 집을 방문한 상태 -> 10


"""

"""

일단 최대 시간 정하는건 필요하다. -> 사람마다 다르다

1. 완탐이 안보인다. -> 바로 답 봄
2. 완탐은 보이는데? 최적화가 안보인다 -> 10분정도 고민하다가? 태그만 본다.
3. 완탐도 보이고? 최적화도 보이는데? 구현이 안된다 -> 그냥 내 능력문제 BABO

솔직히 마음에 안든다 AI답변이

너무 과하게 최적화를 한다거나, 군더더기 기법들을 바른다거나

gpt가 짱이다.

"""