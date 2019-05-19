---
layout: post
title: "Karatsuba 알고리즘 개요"
description: "빠른 곱셈연산을 위한 Karatsuba 알고리즘 알아보기"
tags: [algorithm, ]

---

# 개요
일반적인 n자리수 자연수끼리의 곱셈연산은 $$O(n^2)$$ 시간복잡도를 가진다. Karatsuba 알고리즘은 이를 $$O(n^{\log_{2}3})$$ 시간복잡도로 줄여준다.
이 시간복잡도는 `Master Theorem`에 의해 $$\theta (n^{\log_{2}3})$$ 임이 증명된다.
Karatsuba 알고리즘은 전형적인 분할정복(Divide and conquer) 알고리즘이며, Recursion을 이용하여 구현한다.

본 포스팅에서 Karatsuba 알고리즘을 적용한 자연수곱을 수행하는 함수의 이름을 `karatsuba(x, y)`라 가정한다.


# 알고리즘
곱셈연산을 수행할 2개의 input parameter를 각각 x, y라 하자. Karatsuba 알고리즘은 x, y 각각을 절반으로 나누어 4개로 분할(divide)하는 과정을 포함한다. 따라서 이들이 짝수개의 자리를 가지는가 여부에 따라 시간복잡도 및 사고의 복잡도가 달라진다.

## x, y 모두 $$ 2n $$개의 자리수를 가질 경우

논의의 단순함을 위해 x, y가 모두 <b> 동일한 짝수 자리수($$2n$$)</b>의 자연수가 들어온다고 가정한다. 나중에 보겠지만 이는 알고리즘의 일반성을 해치게 된다.

### Initialize
만일 x, y가 2자리수라면 $$(n=1)$$ 단순곱을 통해 $$x \cdot y$$를 반환한다.

### Divide
x, y 각각을 n번째 자리를 기준으로 분할한다. x, y를 분할한 결과물들을 각각 a, b, c, d 라 하자. 예컨대 $$ x = 1234, \ y = 5678 $$ 인 경우, $$ a = 12, \ b = 34, \ c = 56, \  d = 78 $$ 이다.

그러면 x, y는 a, b, c, d를 이용하여 다음과 같이 표현가능하다.

$$
x = a\cdot 10^n + b
\\
y = c\cdot 10^n + d
$$

이제 우리의 목표는 $$karatsuba(x, \ y) \ = x \cdot y = ac \cdot 10^2n +(ad +bc) \cdot 10^n + bd $$ 를 계산하는 것이다.

$$ 2n \times 2n $$ 연산이었던 $$ x \cdot y $$ 가 $$ n \times n $$ 연산 $$4$$개로 분할되었다

### Conquer
다음을 계산한다.

$$ ac, \ bd, \  ad + bc $$

$$ac, \ bd $$는 두 자연수의 곱이므로 Karatsuba 알고리즘을 재귀적으로 적용한다.

$$
ac = karatsuba(a, \ c) \\
bd = karatsuba(b, \ d) \\
$$

여기서 $$ad, \ bc $$ 도 재귀적으로 함수를 적용하여 구할 수 있다. 그러나 이 경우 복잡도는 기존의 $$ O(n^2) $$가 되어 구현의 의미가 없어진다.

따라서 재귀횟수를 줄이기 위한 트릭이 필요하다.

#### 가우스의 방법
$$ ad+bc $$를 구하기 위해 다음의 트릭을 적용한다.

$$
ad+bc = (a+b)(c+d)-ac-bd
$$

이미 위에서 $$ac$$ 및 $$bd$$는 구해놓았다. 따라서 우리는 $$ad, \ bc $$ 를 재귀적으로 구하는 대신, $$ (a+b)(c+d) $$ 만 재귀적으로 구하면 된다.

$$
ad+bc = karatsuba(a+b, \ c+d)-karatsuba(a, \ c)-karatsuba(b, \ d)
$$

### Combine
Divide에서 생각한바와 같이, 다음을 리턴한다

$$
karatsuba(x, \ y) \ = x \cdot y = karatsuba(a, \ b)\cdot 10^{2n} +(karatsuba(a+b, \ c+d)-karatsuba(a, \ c)-karatsuba(b, \ d)) \cdot 10^{n} + karatsuba(b, \ d)
$$

이상을 Pseudocode로 표현하면 다음과 같다.

### Pseudocode
```
function karatsuba(x, y) {
    length = len(x)
    if (length == 2) {
        return x * y
    }

    n = length / 2
    a = split_half(x, position=n, firstHalf=true)
    b = split_half(x, position=n, firstHalf=false)
    c = split_half(y, position=n, firstHalf=true)
    d = split_half(y, position=n, firstHalf=false)

    ac = karatsuba(a, c)
    bd = karatsuba(b, d)
    ad_bc = karatsuba(a+b, c+d) - ac - bd

    return 10**(2*n) * ac + 10**n * ad_bc + bd
}
```


### 한계
위 알고리즘은 input parameter 모두가 <b>동일한 2n자리수</b>의 자연수가 들어온다고 가정하였다. 따라서 재귀적으로 알고리즘을 적용할 때도 반드시 이 조건이 지켜져야 한다.

그러나 많은 경우 모든 재귀호출에 대해 이 조건은 지켜지기 힘들다. 예컨대 다음의 경우를 보자.

$$
karatsuba(1090, 5000)
$$

이경우 $$a=10, \ b=90 $$이 되는데, $$ad\_bc=karatsuba(100, \ 50)$$이 되어 조건을 만족하지 못한다.

즉, input parameter들은 <b>서로 다른 홀짝에 상관없는 자리수</b>가 들어올 가능성이 높다. 다음 절에서는 이러한 일반적 경우를 다뤄본다.

## 일반적인 경우

이전의 경우와 다르게 자리수가 다른 자연수가 들어올 수 있기에  <b>'중간에서 끊는다'</b>는 개념이 애매해진다. 이 알고리즘에서 가장 중요한 것은 가우스의 방법을 사용하여 재귀의 갯수를 하나라도 줄일 수 있게 하는 것인데, 그러기 위해서는 $$ad+bc$$가 하나의 항으로 묶여야 한다. 그러려면 x, y 모두 반드시 같은 위치에서 잘려야 한다.

예컨대 x를 100의 자리에서 잘랐다면, y도 무조건 100의자리에서 잘려야 한다.

### Initialize
만일 x, y가 1자리수라면 단순곱을 통해 $$x \cdot y$$를 반환한다.

### Divide
<b>일반성을 잃지 않고 x, y 중 작은 자리수를 가진 수를 y라 가정하자. </b> 이 때, y의 길이 를 2로 나눈 몫이 의미하는 자리수를 기준으로 분할한다.

즉, $$ n= int( \frac{len(y)}{2} )$$ 번째 자리를 기준으로 분할한다. x, y를 분할한 결과물들을 각각 a, b, c, d 라 하자.

예컨대 $$ x = 1234567, \ y = 12345 $$ 인 경우, $$n= int( \frac{len(7)}{2} ) = 3 $$이고 따라서 $$ a = 1234, \ b = 567, \ c = 12, \  d = 345 $$ 이다.

그러면 x, y는 a, b, c, d를 이용하여 다음과 같이 표현가능하다.

$$
x = a \cdot 10^n + b
\\
y = c \cdot 10^n + d
$$

위 식이 이전과 동일하므로, 목표 역시 동일하다. $$karatsuba(x, \ y) \ = x \cdot y = ac \cdot 10^2n +(ad +bc) \cdot 10^n + bd $$ 를 계산한다.

### Conquer
이전과 마찬가지로 $$ ac, \ bd, \  ad + bc $$ 는 다음과 같다.

$$
ac = karatsuba(a, \ c) \\
bd = karatsuba(b, \ d) \\
ad+bc = karatsuba(a+b, \ c+d)-karatsuba(a, \ c)-karatsuba(b, \ d)
$$


### Combine
이전과 동일하다.

$$
karatsuba(x, \ y) \ = x \cdot y = karatsuba(a, \ b)\cdot 10^{2n} +(karatsuba(a+b, \ c+d)-karatsuba(a, \ c)-karatsuba(b, \ d)) \cdot 10^{n} + karatsuba(b, \ d)
$$

이를 Pseudocode로 나타내면 다음과 같다.

### Psudocode
```
function karatsuba(x, y) {
    lenx, leny = len(x), len(y)
    if (lenx == 1 or leny == 1) {
        return x * y
    }

    n = int(min(lenx, leny) / 2)
    a = split_half(x, position=n, firstHalf=true)
    b = split_half(x, position=n, firstHalf=false)
    c = split_half(y, position=n, firstHalf=true)
    d = split_half(y, position=n, firstHalf=false)

    ac = karatsuba(a, c)
    bd = karatsuba(b, d)
    ad_bc = karatsuba(a+b, c+d) - ac - bd

    return 10**(n*2) * ac + 10**n * ad_bc + bd
}
```

# 정리
Karatsuba 알고리즘은 기존의 복잡도 $$O(n^{2})$$의 단순곱을 $$O(n^{\log_{2}3})$$ 복잡도로 줄여준다. 대표적인 분할정복 알고리즘이며 재귀함수를 통해 이를 구현한다.

다음 포스팅에서는 Karatsuba 알고리즘을 Python으로 구현하는 방법을 알아본다.

