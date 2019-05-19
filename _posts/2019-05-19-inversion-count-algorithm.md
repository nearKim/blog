---
layout: post
title: "Counting Inversions of an array"
description: "주어진 Array의 Inversion Pair의 갯수를 구하는 알고리즘 알아보기"
tags: [algorithm]

---

# 개요

임의의 길이 n의 정수배열 A가 주어졌을 때, A의 `inversion`이란 다음과 같이 정의된다.

<b>A의 index들의 쌍 $$(i, \ j)$$에 대해 $$ i < j, \ A[i]<A[j] $$이면 $$(i, \ j)$$는 A의 $$inversion$$이다.</b>

이 때 A의 모든 inversion들의 갯수를 출력한다.

가장 쉽게 생각할 수 있는 방법은 Brute force 알고리즘으로, 다음과 같다.

## Brute force 알고리즘
2개의 for loop을 돌며 outer loop의 value보다 큰 inner loop의 value가 존재하면 두 index들을 반환한다. 복잡도는 $$O(n^{2})$$이다.

## 목표 정의
$$O(n^{2})$$보다 효율적인 복잡도를 가지는 Counting Inversion 알고리즘을 고안한다.

## Intuition
$$n=6$$인 경우를 생각해보자. 예컨대 $$A=[1,3,5,2,4,6]$$이다. `정렬된 A`를 `기존 A`의 아래에 놓고 같은 숫자끼리 선으로 연결해본다.

{% include image.html path="documentation/inversion-algorithm-1.png" path-detail="documentation/inversion-algorithm-1.png" alt="image" %}

위 그림에서 교차점의 갯수가 곧 Inversion의 갯수임을 알 수 있다. 정확히는 모르겠지만 Sorting 과정이 본 알고리즘과 연관이 있음을 예상할 수 있다.


# Divide Conquer 알고리즘
Counting Inversion 문제를 Divide and Conquer 방식으로 풀어보자. 우리는 Sorting이 뭔가 해결과정에 도움을 줄 수 있음을 알고 있다. 따라서 대표적인 분할정복 알고리즘인 `Merge Sort`에서부터 출발해본다.

## 정의

`Merge Sort`는 중점을 기준으로 Array를 좌, 우로 분할하므로 Inversion 도 분할된 좌, 우의 Array에서 각각 계산된 후 합쳐져야 한다. 따라서 다음의 3가지를 새롭게 정의한다.

1. Left inversion: $$ \left \{ inversion (i, j) \ s.t \ i < j \ i,j \leq \frac{n}{2}  \right \} $$

2. Right inversion: $$ \left \{ inversion (i, j) \ s.t  \ i < j, \ i,j \ge \frac{n}{2}  \right \} $$

3. Split inversion: $$ \left \{ inversion (i, j) \ s.t  \ i < j, \ i \leq \frac{n}{2} \leq j \right \} $$

1, 2는 재귀적으로 쉽게 구할 수 있다. 그러나 3의 경우 추가적인 연산이 필요하다.

## Pseudocode
배열 A와 배열의 길이 n을 받는 Pseudocode를 생각해보자.

```
function CountInversion(A, n) {
    if n == 0 return 0
    x = CountInversion(A[:n/2], n/2)
    y = CountInversion(A[n/2:], n/2)
    z = CountSplitInversion(A, n)

    return x + y + z
}
```
- $$x, \ y, \ z$$는 각각 Left, Right, Split inversion의 갯수다.
- `CountSplitInversion()`이 $$O(n)$$이라면 merge sort와 마찬가지로 전체 알고리즘은 $$O(n logn)$$ 이 될 것이다.

아직 우리는 Sorting을 고려하지 않았다. 분할된 Array를 정렬하는 과정을 위 Pseudocode에 포함시킨다.

```
function SortAndCountInversion(A, n) {
    if n == 0 return 0
    B, x = CountInversion(A[:n/2], n/2)
    C, y = CountInversion(A[n/2:], n/2)
    D, z = MergeAndCountSplit(A, B, C, n)
    return x + y + z
}
```

- B, C는 각각 분할된 A를 정렬한 부분 배열을 뜻한다.
- D는 B, C를 병합한 (또는 병합된 배열을 저장할) 배열을 뜻한다.

이제 `MergeAndCountSplit()`은 기존의 무질서한 배열이 아닌, 정렬된 배열을 parameter로 받는다.

# 증명
다음의 Lemma를 생각하자.

## Lemma
> 만일 A에 Split inversion이 없으면 모든 B의 원소는 모든 C의 원소보다 작다.

pf) C의 어떤 원소 $$c_{1}$$보다 큰 B의 원소 $$b_{1}$$가 존재한다고 가정하자. 정렬하기 전 $$b_{1}, \ c_{1}$$의 index를 각각 $$i$$, $$j$$라 하자. 그러면 정의에 의해 $$(i, \ j)$$는 inversion이고 따라서 모순. <br>QED

만일 Lemma와 같은 상황에서는 Merge sort의 Combine 과정은 D를 생성할 때, 단순히 B를 먼저 복사하고 그 옆에 C를 이어 붙일 것이다. 즉, B가 가진 pointer가 끝까지 움직이기 전까지 C가 가진 포인터는 첫번째 원소를 가리킨 채 움직이지 않는다. 돌려말하면, <b>A에 inversion이 존재해야 B의 pointer가 끝까지 가지 전에 C의 포인터가 움직인다.</b>

Lemma를 이용하여 다음의 일반적인 Claim을 생성할 수 있다.

## General Claim
> C의 원소 $$y$$를 포함하는 inversion의 수는 $$y$$가 $$D$$에 복사될 때 남아있는 $$B$$의 원소의 개수이다.

pf) $$x$$가 $$B$$에 남은 원소 중 하나라 가정하자.<br>
Lemma에 의해 만일 $$x$$가 $$y$$보다 먼저 $$D$$에 복사되었다면, $$ x < y $$이다. 따라서 inversion이 아니다. <br>
$$y$$가 $$x$$보다 먼저 $$D$$에 복사되었다면, $$ x > y $$이다. 따라서 inversion이다. 그리고 $$B$$는 정렬되어 있으므로 $$x$$이후에 있는 $$B$$의 원소들은 모두 $$y$$보다 크다. 따라서 이들도 모두 inversion이다. <br> QED

### 예시

이제 `Intuition`이 성립하는 이유를 알았다. $$C$$에서 $$D$$로 원소가 복사될 때마다 교차점이 발생하고 교차점의 갯수는 교차점 발생시점에서 남은 $$B$$의 원소 갯수와 일치한다.

{% include image.html path="documentation/inversion-algorithm-2.png" path-detail="documentation/inversion-algorithm-2.png" alt="image" %}

## MergeAndCountSplit
위 Pseudocode를 완결한다.

```
function MergeAndCountSplit(A, B, C, n) {
    i, j = 0
    invTotal = 0
    for k=0 to n-1 {
        if (B[i] < C(j)) {
            D[k] = B[i]
            i++
        }
        if (B[i] >= C(j)) {
            D[k] = C[j]
            j++
            invTotal += (n/2 - i)
        }
    }
    return D, invTotal
}
```

- 총 inversion count를 저장하다가 Merge 과정에서 C의 포인터가 움직일 때마다 B의 잔여 원소갯수를 더한다.

# 복잡도
앞서 논의한대로 `MergeAndCountSplit`의 복잡도가 n이라는 것이 증명된다면 알고리즘의 총 복잡도는 Merge Sort와 같은 $$O(nlogn)$$이 될 것이다. Pseudocode에서 보듯이 이 로직은 sort를 하고 <b>C의 원소를 iteration할 때마다 </b>`invTotal`변수에 자연수를 더해주는 과정이 추가된 것 뿐이다. C의 원소의 갯수만큼 복잡도가 추가되었으므로 `MergeAndCountSplit`의 총 복잡도는 $$O(n)=O(n) + O(n)$$ 이다.

따라서 전체 알고리즘의 복잡도는 $$O(nlogn)$$ 이다.

# 활용
Inversion count는 완전히 역순으로 정렬된 배열일 경우 가장 크다. 이를 이용하여 recommender system에서 취향의 <b>similarity measure</b>로 사용할 수 있다.

예컨대 A라는 사람이 영화 $$c_{1}, \ c_{2}, \ c_{3},\ c_{4}$$에 대해 좋아하는 순서대로 $$[c_{1}, \ c_{2}, \ c_{3},\ c_{4}]$$ 로 평가했다고 하자. 그리고 B, C는 동일한 영화들을 각각 $$[c_{1}, \ c_{2}, \ c_{4},\ c_{3}]$$, $$[c_{4}, \ c_{2}, \ c_{1},\ c_{3}]$$ 으로 평가했다고 가정하자.

B, C중 누가 A와 비슷한 `취향`을 가지고 있다고 할 수 있을까?

이 때 A와 B, 그리고 A와 C 벡터들 사이의 inversion count를 구하면 각각 1, 4이다. A-C 사이의 갯수가 더 크므로 C보다 B가 A와 비슷한 취향을 갖고 있다고 판단할 수 있다.


