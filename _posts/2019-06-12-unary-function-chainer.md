---
layout: post
title: "Higher order function Python implementation"
description: "Chaining 된 Unary Function들로 이루어진 Higher order function을 Python으로 구현하기 "
tags: [python]
---

알고리즘 문제를 풀던 중 [매우 아름다운 문제](https://www.codewars.com/kata/unary-function-chainer/python)를 발견해서 쓰는 글.

# 목표 정의
Function들의 List가 주어졌을 때, List를 구성하는 모든 Unary Function들의 합성함수를 반환하는 1개의 `Higher order function`을 생성한다.

함수의 이름은 편의상 `chaining`이라 하자. 즉 다음 함수를 완성해야 한다.

{% highlight python %}
def chaining(functions):
    # TODO
    return None
{% endhighlight %}

# 문제 해결 전략
개인적으로 처음에는 `reduce`를 이용한 방법밖에 떠오르지 않았다. 현업에서 React를 통해 reduce를 많이 쓰고 있기도 하고, **여러개를 한개로 반환한다**는 컨셉이 곧 직관적으로 reduce를 사용해야 할 것 같다는 강한 확신을 줬기 때문이다.

하지만 조금 더 시간을 두고 생각해봤을 때, `Closure`를 사용하면 보다 깔끔하게 해결할 수 있을 것 같았고, 실제로도 그러했다. 본 포스팅에서는 2가지 방법을 모두 다뤄본다.

# Reduce를 통한 구현

### Step 1
먼저, 정의한대로, `chaining`은 단일 함수를 리턴해야 한다.
- 이 단일 함수는 function 내지는 lambda일 것이고, Unary function들의 합성함수를 계산할 것이므로 input으로는 1개만을 받을 것이다. 이를 `func(x)`라 하자.

{% highlight python %}
def chaining(functions):
    def func(x):
        return # TODO
    return func
{% endhighlight %}

### Step 2
func는  **List of functions**을 **Single function**으로 변환해야 한다.
- func는 reduce를 통해 구현될 것이고 reduce의 명세를 보았을 때, iterable은 `functions`일 것이다.

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
reduce의 첫번째 parameter인 function은 accumulator(acc라 하자)와 funcions의 원소들을 각각 받아서(`f`라 하자) `어떤 처리`를 하여 합성함수를 리턴해야 한다.
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

### Step 4
함수는 완성되었으나 간결하게 리팩터해보자. 의도치 않게 closure의 형태를 띄게 되었는데 (여기서 이후에 논의할 Closure를 이용한 풀이의 실마리를 얻는다) 위 코드는 그럴 필요가 하등 없다. lambda를 사용하면 한줄에 표시할 수 있다.

{% highlight python %}
from functools import reduce
def chaining(functions):
    return lambda x: reduce(lambda acc, f: f(acc), functions, x)
{% endhighlight %}

이로써 풀이가 끝났다.

# Closure를 통한 구현
위에서 이미 Closure에 대한 아이디어를 얻었다. 사실 저렇게 놔둬도 상관은 없으나, 굳이 reduce를 쓸 필요가 있을까 싶다.

### 풀이 과정
inner function에서 해야할 일은 input parameter x를 받아서 함성함수들을 리턴하는 것 뿐이다.
- x에 함수를 적용하고 다시 x에 대입하자
- 이 과정을 반복한 후 x를 리턴하면 그게 결국 함수를 chaining 한 것이다

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

# 정리
Javascript에서도 Closure가 존재하지만 지금까지 현업에서 Closure를 쓸일은 사실 많지 않았다. Higher order function도 FP 패러다임에서 활발히 사용된다는 것만 알았지 직접 마주칠일이 없었는데, 알고리즘 문제를 풀다가 영감을 얻어 직접 구현하게 되니 대단히 재미있었다.

다음 프로젝트에서 함수 Chaining을 사용할 일이 있으면 응용해봐야겠다.
