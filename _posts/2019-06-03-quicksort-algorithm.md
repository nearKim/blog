---
layout: post
title: "Quicksort 알고리즘 개요"
description: "Random pivot을 기준으로 정렬하는 quicksort 알고리즘 알아보기"
tags: [algorithm]
---

# 개요
Quicksort는 mergesort와 같이 대표적인 분할정복 알고리즘이다.

길이 n인 정수배열 A가 주어졌을 때, A의 원소 중 하나를 랜덤하게 지정하여 pivot으로 선택한다. 이 때 pivot을 기준으로 좌측에는 pivot 미만의 원소, 우측에는 pivot 초과의 원소가 존재하도록 A를 재배열한다. 그 후 좌, 우측 각각의 배열에 대해 recursive하게 적용하여 정렬한다.

Quicksort의 시간 복잡도는 Pivot을 어떻게 설정하느냐에 따라 달라진다. 최악의 경우 $$O(n^2)$$ 이며, pivot을 랜덤하게 선택하면 `평균적으로` $$O(n \log n)$$의 복잡도를 가진다. 따라서 pivot의 선택 전략과 pivot기준 좌/우측 분할 전략이 중요하다.

## 목표 정의
Quicksort 알고리즘과 알고리즘의 성질을 알아보고 랜덤 Pivot 선택에 따른 시간복잡도가  $$O(n \log n)$$ 임을 수학적으로 증명한다.


# 알고리즘
일반적인 Quicksort 알고리즘은 다음과 같다. 논의의 편의성을 위해 중복값은 없다고 가정하자.

1. 초기화
        * 만일 $$A$$의 원소가 1개라면 종료한다.
        * $$A$$의 원소 중 하나를(pivot) `선택`하여 $$p$$라 하자.
        * 현재 배열의 가장 좌측의 원소를 가리키는 포인터를 $$i=0$$라 하자.
2. $$A$$를 선형탐색하여 다음을 적용한다.
        1. 만일 탐색된 원소가 $$p$$보다 큰 경우 아무것도 하지 않는다.
        2. 만일 탐색된 원소가 $$p$$보다 작은 경우
                * $$i$$번째 원소와 탐색된 원소를 swap한다.
                * $$i++$$
3. 선형탐색 종료 후 pivot과 i번째 원소를 swap한다.
4. pivot 기준 좌측 배열 및 우측 배열에 본 알고리즘을 recursive하게 적용한다.

물론 pivot보다 작으면 아무것도 하지 않고 큰 원소를 우측부터 쌓아도 상관없다.

위 과정을 거치면 아래 그림과 같이 Pivot 기준 좌,우로 Array가 분할된다.

{% include image.html path="documentation/quicksort-algorithm-1.png" path-detail="documentation/quicksort-algorithm-1.png" alt="image" %}

이 과정의 Pseudocode는 다음과 같다.

# Pseudocode
Sorting할 배열 $$A$$, 현재 iteration에 해당하는 배열의 시작점을 가리키는 변수 $$l$$, 종점을 가리키는 변수 $$r$$이 input으로 들어온다고 하자.
또한 주어진 Array `X`에서 Pivot을 선택하는 함수를 `SelectPivot(X)`로 표현한다고 하자. 예컨대 항상 주어진 Array의 첫번째 원소를 pivot으로 선택한다면 `SelectPivot`은 `X[0]`을 반환할 것이다.


먼저 pivot을 기준으로 좌,우측 sub array로 분할하는 코드를 생각한다.

```
function Partition(A, l, r) {

    pivot = SelectPivot(A[l:r+1])

    for i=l to r {
        if (A[i] < pivot) {
            swap(A[i], A[l])
            l += 1
        }
    }

    swap(A[l], pivot)
    return l, pivot,  r
}

```
- 적당한 규칙으로 Pivot을 선택한다. 이 과정에 따라 복잡도가 달라진다.
- 현재 들어온 Array의 시작부터 끝까지 iteration을 돌면서 pivot보다 작은 원소를 가장 좌측으로 옮긴다.
- 마지막으로 pivot을 옮긴다.
- 좌측 부분배열, 우측 부분배열, pivot을 알 수 있도록 리턴한다.

위 함수를 사용하여 Quicksort를 구현한다.

```
function Quicksort(A, l=0, r=len(A)) {
    if (r-l < 2) return A[l:r+1]

    l, pivot,  r = Partition(A, l, r)

    left_arr = Quicksort(A, 0, l)
    right_arr = Quicksort(A, l+1, r)

    total_arr = left_arr + [pivot] + right_arr
    return total_arr
}
```
- 주어진 배열의 길이가 1이하라면 그대로 반환한다.
- Partition하여 recursive하게 적용할 Array의 경계를 구한다.
- 좌/우측 Array를 recursive하게 정렬한다.
- 마지막에 결과값들을 1개로 묶어서 리턴한다.

# 정확성 증명
위 Code가 정확하게 주어진 배열을 정렬할 수 있음을 수학적 귀납법으로 증명한다.

$$P(n)$$이 Quicksort가 $$n$$개 원소 배열을 정확하게 정렬하는 사건이라고 하자.

### $$[Claim]$$ $$P(n)$$이 $$pivot$$에 상관없이 모든 $$n \geq 1 $$에 대해 성립한다.

$$pf) $$ $$n=1$$ 인 경우 자명하다.

$$\forall k < n $$에 대해 $$P(k)$$가 성립한다고 가정하자. $$ETS:  P(n)$$

Quicksort는 $$A$$를 $$pivot$$ 기준으로 partition한다. partition 결과 좌측 부분 배열을 $$B$$, 우측 부분배열을 $$C$$라 하고 길이를 각각 $$k_1$$, $$k_2$$라 하자.

자명히 $$k_1 , k_2 < n$$이므로 귀납가설에 의해 $$P(k_1), P(k_2)$$가 성립한다. 따라서 $$B, C$$는 정렬 가능하다.

$$Q.E.D$$

# 정리
Quicksort는 대표적인 분할정복 알고리즘으로 임의로 정한 `pivot`을 기준으로 크기비교를 통해 좌,우측 부분 배열로 분할한다. 그 후 분할된 부분배열에 각각 recursive하게 알고리즘을 적용하여 정렬하게 된다. 알고리즘의 정확성은 수학적 귀납법에 의하여 쉽게 도출된다.

상술했듯이 Quicksort는 최악의 경우 $$O(n^2)$$, 평균적으로 $$O(n \log n)$$의 복잡도를 가진다. 다음 포스팅에서는 이에 대한 증명을 알아본다.

