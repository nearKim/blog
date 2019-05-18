---
layout: post
title: "Karatsuba 알고리즘 구현하기"
description: "빠른 곱셈연산을 위한 Karatsuba 알고리즘을 Python으로 구현하기"
tags: [algorithm, python]

---

# 목표 정의
일반적인 [Karatsuba 알고리즘](karatsuba-basic)를 Python을 사용하여 구현하고 검증한다.

Input으로는 integer 타입의 양수 x, y가 들어온다고 가정한다.


# Pseudocode
이전 포스팅에서 사용한 pseudocode를 참조한다.

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
<br>

# Python code 작성

{% highlight python %}
def karatsuba(x, y):
    # 각각의 length를 구한다
    strx, stry = str(x), str(y)
    lenx, leny = len(strx), len(stry)

    # Initialize
    if lenx == 1 or leny ==1:
        return x * y

    # Divide
    m = min(lenx, leny) // 2
    a, b = int(strx[:-m]), int(strx[-m:])
    c, d = int(stry[:-m]), int(stry[-m:])

    # Conquer
    ac = karatsuba(a, c)
    bd = karatsuba(b, d)
    ad_bc = karatsuba(a+b, c+d) - ac - bd

    # Combine
    return 10**(2*m) * ac + 10**m * ad_bc + bd
{% endhighlight %}

- Input parameter들의 길이와 같이 알고리즘에서 사용할 변수들을 먼저 구해놓는다
- 둘 중 하나라도 길이가 1이면 단순곱을 반환한다
- 두 parameter 중 짧은 길이를 가진 parameter의 length 절반만큼 잘라서 분할한다.
- 재귀적으로 변수를 구한다. 이 때 가우스의 방법을 이용하여 재귀 횟수를 줄인다.
- 결합하여 리턴한다.

# Test
랜덤하게 양의 정수 두개를 생성하여 단순곱을 한 후, 우리가 구현한 알고리즘과 비교하면 테스팅이 가능하다. 이미 karatsuba 함수가 존재한다고 가정한다.

{% highlight python %}
import unittest
import random

class TestKaratsuba(unittest.TestCase):

    def test_initialize(self):
        # length 가 1인 parameter가 들어왔을 때를 검사한다.
        self.assertEqual(karatsuba(2, 5), 10)
        self.assertEqual(karatsuba(3, 180), 540)
        self.assertEqual(karatsuba(201, 5), 1005)

    def test_karatsuba_full(self):
        # 일반적인 경우, 랜덤하게 자연수를 생성하여 결과를 100회 비교한다
        for i in range(100):
            x, y = random.randint(10, 100000000), random.randint(10, 100000000)
            self.assertEqual(karatsuba(x, y), x*y)

if __name__ == '__main__':
    # unittest.main(exit=False) # Python shell에서 실행할 경우 exit=False 옵션을 넣어준다.
    unittest.main()
{% endhighlight %}

결과는 다음과 같다.

{% highlight bash %}
...
3494974 94688452
7554174 15627692
6509325 13425690
9168297 4388704
2725998 80174482
2798603 44238231
9275283 39541047
5289959 31248220
.
----------------------------------------------------------------------
Ran 2 tests in 0.091s

OK
{% endhighlight %}

이로써 검증이 완료되었다.

> 참고로 Python3는 int 타입은 unbounded이다. 예컨대 PHP나 JAVA에 존재하는 `PHP_INT_MAX`나 `Integer.MAX_VALUE`와 같은 정수 최대치를 고려할 필요가 없다.
