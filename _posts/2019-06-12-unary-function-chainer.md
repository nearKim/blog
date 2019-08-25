---
layout: post
title: "Higher order function Python implementation"
description: "Chaining 된 Unary Function들로 이루어진 Higher order function을 Python으로 구현하기 "
tags: [python]
---

알고리즘 문제를 풀던 중 [매우 아름다운 문제](https://www.codewars.com/kata/unary-function-chainer/python)를 발견해서 쓰는 글.

# 목표 정의
Unary function들의 List가 주어졌을 때, 이들의 합성함수를 나타내는 `Higher order function`을 생성한다.

함수의 이름은 편의상 `chaining`이라 하자. 즉 다음 함수를 완성해야 한다.

{% highlight python %}
def chaining(functions):
    # TODO
    return # A function
{% endhighlight %}

# 문제 해결 전략
직관적으로 `reduce`를 사용해야 할 것 같은 느낌이 든다. **여러 원소를 한개의 원소로 축소하여 반환하는** 문제이기 때문이다. 하지만 조금 더 여유를 가지고 생각해봤을 때, `Closure`를 사용하면 보다 깔끔하게 해결할 수 있을 것 같았고, 실제로도 그러했다.

본 포스팅에서는 2가지 방법을 모두 다뤄본다.

# Reduce를 통한 구현

<br>

### Step 1
먼저, 정의한대로, `chaining`은 단일 함수를 리턴해야 한다.
- 이 단일 함수는 function 내지는 lambda일 것이고, Unary function들의 합성함수는 Unary이므로 input은 1개일 것이다. 이 함수를 `func(x)`라 하자.

{% highlight python %}
def chaining(functions):
    def func(x):
        return # TODO
    return func
{% endhighlight %}

### Step 2
`func`는  **함수의 리스트**를 **단일 함수**로 변환해야 한다.
- func는 reduce를 통해 구현될 것이고 아래 reduce의 명세에서, iterable은 `functions`일 것이며 initializer는 `x`일 것이다.

> reduce (function, iterable [, initializer])

{% highlight python %}
from functools import reduce
def chaining(functions):
    def func(x):
        return reduce(
            None, # TODO
            functions,
            x
        )
    return func
{% endhighlight %}


### Step 3
reduce의 명세에서 첫번째 parameter인 function은 accumulator(`acc`)와 funcions의 원소들을(`f`) 각각 받아서 합성하여 리턴해야 한다.
- reduce의 첫 인자로 `lambda acc, f: f(acc)` 가 들어가야 한다.

{% highlight python %}
from functools import reduce
def chaining(functions):
    def func(x):
        return reduce(
            lambda acc, f: f(acc),
            functions,
            x
        )
    return func
{% endhighlight %}

#### 부가설명
- 최초로 x가 acc로 삽입될 것이다. 이는 functions의 첫번째 함수($$f_1$$)과 합성되어 반환된다:  $$f_{1}(x)$$
- $$f_{1}(x)$$는 reduce의 정의에 따라 acc로 삽입되고 두번째 함수($$f_2$$)와 합성되어 반환된다: $$f_{2}(f_{1}(x))$$
- 같은 방법으로 계속하면 결국 $$f_{n}(...f_{2}(f_{1}(x))...)$$가 반환됨을 알 수 있다.

### Step 4
함수가 완성되었으므로 간결하게 리팩터해보자. 위 함수는 의도치 않게 closure의 형태를 갖게 되었는데 (여기서 Closure를 이용한 풀이의 실마리를 얻는다) 그럴 필요 없이 lambda를 사용하면 한줄에 표시할 수 있다.

{% highlight python %}
from functools import reduce
def chaining(functions):
    return lambda x: reduce(lambda acc, f: f(acc), functions, x)
{% endhighlight %}

이로써 풀이가 끝났다.

# Closure를 통한 구현
위에서 이미 Closure에 대한 아이디어를 얻었다. reduce를 사용하지 않고 구현해본다.

### 풀이 과정
inner function에서 해야할 일은 input parameter `x`를 받아서 합성함수를 리턴하는 것 뿐이다.
- x에 functions의 첫번째 함수를 적용하고 다시 x에 대입한다.
- x에 functions의 두번째 함수를 적용하고 다시 x에 대입한다.
- 이 과정을 반복한 후 x를 리턴하면 결국 모든 functions의 함수들을 chaining 한 것과 동일하다.

{% highlight python %}
from functools import reduce
def chaining(functions):
    def func(x):
        for f in functions:
            x = f(x)
        return x
    return func
{% endhighlight %}

reduce를 이용한 방법보다 위 방법이 보다 깔끔하고 이해하기 쉽다.


