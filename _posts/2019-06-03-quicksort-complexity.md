---
layout: post
title: "Quicksort 알고리즘 복잡도"
description: "Quicksort 알고리즘의 평균 시간복잡도 수학적으로 증명하기"
tags: [algorithm]
---

# 목표 정의
[이전 포스팅](quicksort-algorithm)에서 논의한 바와 같이 Quicksort의 시간 복잡도는 최악의 경우 $$O(n^2)$$ 이며, pivot을 랜덤하게 선택하면 <b>평균적으로</b> $$O(n \log n)$$의 복잡도를 가진다. 본 포스팅에서는 알고리즘의 평균 복잡도가 $$O(n \log n)$$임을 수학적으로 증명한다.

# Worst case
Pivot을 선택할 때 항상 배열의 <b>최소값</b>을 선택한다고 하자. 이 경우 `Partition`과정 종료 후 좌측 부분배열은 공집합일 것이다. 따라서 한번의 iteration이 종료된 후 pivot을 제외한 $$n-1$$개의 원소들에 대해 알고리즘이 적용되게 된다.

즉, Pivot이 최소값일 경우에는 분할정복 알고리즘의 이점을 전혀 살리지 못한다. 각 iteration은 단순히 배열을 선형탐색한 후 오직 1개의 원소만을 제거할 뿐이다.

$$i+1$$번째 iteration에서는 $$n-i$$개의 원소를 탐색하게 되고 $$0 \leq i \leq n$$이므로 복잡도는 다음과 같다.

$$n+(n-1) + (n-2) + ... + 1 = O(n^2)$$

# Best case
위 worst case는 분할정복의 이점을 살리지 못한다고 하였다. 그렇다면 항상 주어진 배열의 중간값(median)을 pivot으로 지정할 수 있다면 어떻게 될까?

* 각 iteration마다 모든 부분배열의 원소들을 1번씩 선형탐색한다.
* 한번에 정확히 2개씩 부분배열이 생성되므로 iteration은 약 $$\log n$$ 번 진행된다.

이 경우 Mergesort와 정확히 동일하다는 것을 알 수 있다. 따라서 복잡도는 $$O(n \log n)$$임을 알 수 있다. 하지만 중간값을 구하려면 위 과정을 진행하기 전에 한번 더 선형탐색을 해야 하는 번거로움이 있다.

# Avg case
가장 구현이 편하고 직관적으로 생각하기 쉬운 pivot은 random pivot이다. Random pivot을 선택해도 평균적으로 Best case에 가까운 $$O(n \log n)$$의 복잡도가 도출됨을 증명한다.

### 정의
* $$\Omega$$: Quicksort의 표본공간. 즉, Quicksort에서 도출 가능한 모든 outcome들의 집합.
* $$C(\sigma )$$: 주어진 Quicksort 시행 $$\sigma$$에서 발생한 원소(element) 사이의 비교 회수.

### Lemma
$$RT( \sigma )$$가 Quicksort $$ \sigma $$의 전체 Running time이라고 하자. 이 때 다음이 성립한다.

$$\exists c >0 \$$  $$s.t \$$  $$\forall  \sigma \in \Omega \$$  $$RT( \sigma ) \leq c C( \sigma)$$

직관적으로 이는 자명하다. 알고리즘에서 원소간 비교를 제외한 연산은(예컨대 swap) 상수번 발생하기 때문이다.

Lemma에 따라 $$RT$$의 상한이 $$C$$이므로 $$E(C)=O(n \log n)$$임을 증명하면 된다.

### Indicator variable
주어진 배열의 두 원소가 비교되는지 여부를 Indicator variable로 선언하여 평균을 구한다.

$$z_i , z_j$$가 각각 배열의 $$i, j$$번째로 <b>작은</b> 원소일 때,

$$X_{ij}( \sigma)$$: Quicksort $$\sigma$$ 진행과정에서 $$z_i$$와 $$z_j$$ 간의 비교 회수

라 정의하자.

Quicksort는 모든 iteration에서 <b>pivot을 기준으로</b> 나머지 원소들을 비교한다. 두개의 element 중 1개가 pivot으로 선택된다면 선형탐색시 무조건 1번 비교되고 이후의 recursive call에서는 제외된다. 알고리즘 종결시까지 둘다 pivot으로 선택되지 못했다면, 애초에 두 원소는 서로 비교되지 않는다.

따라서 $$z_i$$와 $$z_j$$는 한번의 Quicksort 과정에서 무조건 <b>0회</b> 혹은 <b>1회</b> 비교된다.

$$X_{ij}( \sigma) = \left\{\begin{matrix}
 0 & i \ or \ j \ is \ a \ pivot \\
 1 & else
\end{matrix}\right.$$

### Proof
정의에 의해 임의의 $$\sigma$$에 대하여

$$C(\sigma)=\sum_{i=1}^{n-1}\sum_{j=i+1}^{n}X_{ij}(\sigma)$$ 이고 기대값의 선형성에 의하여

$$E[C(\sigma)]=\sum_{i=1}^{n-1}\sum_{j=i+1}^{n}E[X_{ij}(\sigma)]$$ 이다.

$$P_0$$를 $$\sigma$$에서 $$i,j$$가 비교되지 않을 확률, $$P_1$$은 비교될 확률이라 하면, 기대값의 정의에 의해

$$E[X_{ij}(\sigma)] = P_{0}\cdot 0+P_{1}\cdot 1 = P_1$$ 이다.



#### $$[Claim] \ $$ $$\ \forall i<j$$ $$ \ P_{1}=\frac{2}{j-i+1}$$
$$pf)$$  $$ i < j $$인 $$i, j$$에 대해 $$A=\left \{ z_{i}, ... , z_{j} \right \}$$를 생각하자. $$(z_{i} < ... < z_{j})$$

$$A$$의 어떤 원소도 pivot으로 선택되지 않는다면 둘은 같은 recursive call에 속한다. 따라서 이 경우를 제외하고, $$A$$ 중 한 원소가 Pivot으로 선택될 경우를 생각하자.

1. $$z_i$$ 혹은 $$z_j$$가 pivot으로 최초 선택되면 둘은 비교된다. $$(P_{1})$$
2. 그렇지 않다면 둘은 절대로 비교되지 않는다. $$(P_{0})$$

따라서 확률은 $$i$$부터 $$j$$까지 $$j-i+1$$개의 원소들 중 2개가 선택될 확률과 동일하다.

$$\therefore P_{1}= \frac{2}{j-i+1} \ \blacksquare $$

$$Claim$$에 의해 다음을 보이면 충분하다.

$$WTS: \$$ $$ \ E[C(\sigma)]=\sum_{i=1}^{n-1}\sum_{j=i+1}^{n} \frac{2}{j-i+1}$$

$$ pf) \$$ $$E[C(\sigma)]= 2 \sum_{i=1}^{n-1}\sum_{j=i+1}^{n} \frac{1}{j-i+1}$$

이 때, $$\sum_{k=2}^{n} \frac{1}{k} < \sum_{k=2}^{\infty} \frac{1}{k} = \int_{2}^{n}\frac{1}{x}dx \leq \ln n $$ 이므로,

$$\therefore 2\sum \sum \frac{1}{j-i+1} \leq 2 \sum_{1}^{n-1} \ln n = 2(n-1) \ln n$$

따라서 시간복잡도의 정의에 의해 $$E[C(\sigma)]=O(n \log n)$$ 이고 증명이 끝났다. $$\blacksquare$$

# 정리
Random pivot을 활용한 Quicksort 알고리즘의 평균 복잡도는 Best case와 동일한 $$O(n \log n)$$ 이다. 배열의 원소들은 서로 최대 1회까지 비교된다는 사실을 활용하여 <b>Indicator variable</b>을 활용하여 평균을 계산할 수 있었다. 이와 같이 큰 문제를 작은 문제 여러개로 변환하여 푸는 방법론을 선형계획에서 <b>Decomposition principle</b>라 한다.


