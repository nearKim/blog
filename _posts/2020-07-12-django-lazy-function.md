---
layout: post
title: "Django lazy 함수 파헤치기"
description: "Django가 제공하는 lazy 함수의 구조 및 작동방식 분석하기"
tags: [python, django]
---


Django의 `reverse()` 함수는 viewname과 args 및 kwargs를 인자로 받아 url string을 반환한다. 

예컨대 **news** app에 다음과 같은 url이 등록되어 있다고 하자. 

{% highlight python %}
from news import views

path('archive', views.archive, name='news-archive')

{% endhighlight %}

이 때 어떤 View 함수에서 위 url로 리다이렉트하고 싶다면 다음과 같이 사용한다.

{% highlight python %}
from django.urls import reverse

def myview(request):
    return HttpResponseRedirect(reverse('news:news-archive'))
{% endhighlight %}

`reverse()`함수를 사용하면, url이 변경되더라도 View의 코드를 수정할 필요가 없다. 일반적으로 url이 수정될 확률이 높은 점을 고려하면 합리적인 설계라 할 것이다.

# reverse_lazy()

reverse_lazy는 reverse와 동일한 동작을 함수 호출시 곧바로 처리하지 않고, 나중에 해당 변수가 직접 접근되거나 메서드가 호출되었을 때 evaluate한다. 

`reverse`는 내부에서 urlconf를 참조하기 때문에 제대로 동작하기 위해서는 프로젝트의 **urlconf**가 모두 로드되어야 한다. 그러나 때에 따라 urlconf가 로드되기 전에 해당 값을 참조해야 할 수도 있다.

예컨대 아래의 경우 reverse로 success_url을 정의하면 동작하지 않는다.


{% highlight python %}

class JobCreateView(CreateView):
    template_name = 'company/job.html'
    form_class = JobForm
    success_url = reverse_lazy('job')
    # FIXME
    # success_url = reverse('job')
{% endhighlight %}

이는 Python은 모듈이 import될 때 class들을 evaluate하기 때문이다.
<details>
<summary>예시</summary>
<p>

{% highlight python %}

# test.py
def a():
	print('test1')

class B:
	print('test2')

{% endhighlight %}

{% highlight bash %}
>>> import test
>>> test2

{% endhighlight %}

</p>
</details>

<br/>
따라서 이 경우 `success_url`을 `reverse_lazy`를 사용하여 정의하고, 나중에 필요할 때 reverse가 일어나도록 해야한다.

그렇다면 어떻게 이런 동작이 가능할 것일까? 

Django의 소스코드를 보면 정의는 간단하다.

{% highlight python %}
reverse_lazy = lazy(reverse, str)
{% endhighlight %}

# lazy

소스코드의 lazy 함수의 정의를 살펴보면 아래와 같다.

{% highlight python %}

def lazy(func, *resultclasses):
	class __proxy__(Promise):
		# pass

	@wraps(func)
	def __wrapper__(*args, **kw):
		return __proxy__(args, kw)
	return __wrapper__

{% endhighlight %}

편의상 `__proxy__`의 코드를 일단 생략하자. 위 함수의 동작은 명확하다. 

`lazy`함수는 `wraps(func)`로 데코레이팅 된 `__wrapper__`함수, 즉 `wraps(func)(__wrapper__)`를 반환한다.


## @wraps

wraps 데코레이터는 Python functools 모듈에 기본 내장된 함수로, 데코레이터 적용 후에도 감싸진 함수의 기본적인 성질을 잃지 않도록 해준다.

Decorator를 적용하면, decorator 내부의 closure를 반환하게 된다. 이는 함수의 이름을 비롯한 어트리뷰트가 바뀌는 side effect를 발생시킨다.


<details>
<summary>예시</summary>
<p>

{% highlight python %}

def counter(fn):
	count = 0
	def inner(*args, **kwargs):
		nonlocal count
		count += 1
		print(count)
		return fn(*args, **kwargs)
	inner = wraps(fn)(inner)
	return inner

@counter
def mult(a, b, c=1):
	pass

mult.__name__  # inner
help(mult)
{% endhighlight %}

- decorator가 적용되면 반환된 함수는 counter 함수 내부의 inner이기 때문에 함수 이름이 inner로 변경된다.

</p>
</details>

따라서 이러한 문제를 방지하기 위해 `wraps` 함수는 자신이 래핑하고 있는 함수 (func)의 어트리뷰트를 뽑아내어 자신이 반환할 함수에 넣어준다.


{% highlight python %}

def wraps(wrapped,
          assigned = WRAPPER_ASSIGNMENTS,
          updated = WRAPPER_UPDATES):
    return partial(update_wrapper, wrapped=wrapped,
                   assigned=assigned, updated=updated)
{% endhighlight %}

- 이 데코레이터는 update_wrapper의 3가지 인자에 기본값을 넣은 새로운 함수를 `partial`을 통해 생성하여 반환한다.

- 데코레이터 첫번째 인자인 **wrapped**는 `func`이고 나머지는 따로 값을 부여하지 않았으므로 기본값이 들어간다


### update_wrapper


{% highlight python %}
WRAPPER_ASSIGNMENTS = ('__module__', '__name__', '__qualname__', '__doc__',
                       '__annotations__')
WRAPPER_UPDATES = ('__dict__',)

def update_wrapper(wrapper,
                   wrapped,
                   assigned = WRAPPER_ASSIGNMENTS,
                   updated = WRAPPER_UPDATES):
    for attr in assigned:
        try:
            value = getattr(wrapped, attr)
        except AttributeError:
            pass
        else:
            setattr(wrapper, attr, value)
    for attr in updated:
        getattr(wrapper, attr).update(getattr(wrapped, attr, {}))
    wrapper.__wrapped__ = wrapped
    # Return the wrapper so this can be used as a decorator via partial()
    return wrapper

{% endhighlight %}

- `wrapped`로부터 WRAPPER_ASSIGNMENTS로 정의된 어트리뷰트들을 뽑아낸 후,
- `wrapper`에 그 값을 그대로 넣어준다.
- `wrapped`로부터 WRAPPER_UPDATES로 정의된 어트리뷰트들을 뽑아낸 후,
- `wrapper`에 그 값을 추가하여 업데이트한다.
- `wrapper`에 무엇을 래핑했는지를 `__wrapped__` 로 넣어준 후 반환한다.


### 중간정리

{% highlight python %}
@wraps(func)
def __wrapper__(*args, **kw):
	# Creates the proxy object, instead of the actual value.
	return __proxy__(args, kw)
return __wrapper__

{% endhighlight %}

위 코드는 결국 아래와 같다.

{% highlight python %}
__wrapper__ = wraps(func)(__wrapper__)
__wrapper__ = partial(update_wrapper, wrapped=func, assigned=WRAPPER_ASSIGNMENTS, updated=WRAPPER_UPDATES)(__wrapper__)

__wrapper__ = update_wrapper(__wrapper__, wrapped=func, assigned=WRAPPER_ASSIGNMENTS, updated=WRAPPER_UPDATES)


{% endhighlight %}

`@wraps(func)`는 데코레이터가 부여된 함수\__wrapper__에 `func`의 어트리뷰트를 인젝션한다.

`func`가 `str`을 반환하는 것과 다르게, \__wrapper__는 \__proxy__ 객체를 반환한다. 프록시 객체가 무엇인지 알아보자. 

## \__proxy__

{% highlight python %}
@total_ordering
class __proxy__(Promise):
    __prepared = False

    def __init__(self, args, kw):
        self.__args = args
        self.__kw = kw
        if not self.__prepared:
            self.__prepare_class__()
        self.__prepared = True

	@classmethod
    def __prepare_class__(cls):
        for resultclass in resultclasses:
            for type_ in resultclass.mro():
                for method_name in type_.__dict__.keys():
                    if hasattr(cls, method_name):
                        continue
                    meth = cls.__promise__(method_name)
                    setattr(cls, method_name, meth)

	@classmethod
    def __promise__(cls, method_name):
        # Builds a wrapper around some magic method
        def __wrapper__(self, *args, **kw):
            # Automatically triggers the evaluation of a lazy value and
            # applies the given magic method of the result type.
            res = func(*self.__args, **self.__kw)
            return getattr(res, method_name)(*args, **kw)
        return __wrapper__
{% endhighlight %}

- \__proxy__ 는 생성자에서  \__prepare_class__ 를 호출한다.

### \__prepare_class__

- \__prepare_class__ 는 resultclasses의 mro를 순회하며 각 mro들이 갖고 있는 모든 메소드들의 이름을 추출한다.
- 추출된 메소드 이름을 method_name이라고 할 때, 이를 \__proxy__ 객체가 가지고 있다면 패스한다.
- 그렇지 않다면, \__proxy__객체의 method_name 어트리뷰트로 `cls.__promise__(method_name)`을 넣어준다.

즉, \__prepare_class__는 \__proxy__ 객체가 resultclasses의 모든 어트리뷰트를 `cls.__promise__(method_name)`의 형태로 promisify 하여 들고있게한다.

### \__promise__(cls, method_name)

- 이 함수는 inner function인 \__wrapper__를 반환한다.
- \__wrapper__는 **호출되었을 때, `func`를 evaluate**한다. 그리고 evaluate한 결과값으로부터 method_name을 가진 메서드를 추출한 후, **추출된 메서드를 호출**한다. 

즉, \__proxy__의 함수 "method_name"이 호출되면, 그 때 `func`가 evaluate되고, 그 결과값의 "method_name"이 대신 호출된다. 

따라서 `__proxy__(args, kw).{method_name}`은 곧 `{func(*args, **kw)의 결과값}.{method_name}`이다.


### 중간정리


`__proxy__`객체의 모든 어트리뷰트 메서드들은 resultclasses들의 모든 method_name들을 free variable로 가지고 그에 바인딩된 내부함수 `__wrapper__`로 이루어진 closure이다. 

그리고 `__proxy__`의 method_name이 호출되었을 때, `func`가 호출된다. (lazy evaluation)

한마디로 `__proxy__`객체는 resultclasses들의 탈을 쓴 프록시 객체이다. 


# 결론

다음 예시를 통해 위 내용을 정리해본다. 타이핑은 편의를 위해 임의로 사용하였다.

{% highlight python %}
def func(text):
	return text.title()

lazy_func: '__wrapper__' = lazy(func, str)
res: '__proxy__' = lazy_func('test')

res.find('T')
{% endhighlight %}

1. `lazy_func`는 호출되었을 경우 *즉시 func를 evaluate하지 않고* `__proxy__`객체를 반환한다. 이에 따라 아직 func는 호출되지 않는다.
2. 이 `__proxy__`객체는 제공된 `str` resultclasses를 이용하여 str클래스의 mro가 가지고 있는 모든 attribute를 들고 있다.
3. 그 attribute들은 lazy evaluation이 일어나도록 `__proxy__.__promise__` 메서드를 이용하여 `__wrapper__`로 패치된 후 `__proxy__`에 인젝션된다. (promisify)
4. `res.find('T')`가 호출되면 `__proxy__('test').find('T')`가 호출되고, 이 때 `__wrapper__`가 실행되면서 내부의`func`가 evaluate 된다. (lazy evaluation)

따라서 사용자가 `res.find('T')`를 사용할 때`func`가 실행된다.

이를 위의 Django CBV에 적용해보자.

{% highlight python %}
class JobCreateView(CreateView):
    template_name = 'company/job.html'
    form_class = JobForm
    success_url = reverse_lazy('job')
{% endhighlight %}

class가 evalaute 되더라도 `success_url`은 곧바로 evaluate 되지 않는다. `success_url`은 \__proxy__객체이기에 에러가 발생하지 않는다.

이 변수에  \__proxy__ 객체이지 `str` 객체가 아니다. 그럼에도 문제가 일어나지 않는다. \__proxy__는  `str`의 모든 어트리뷰트를 `__wrapper__`로 감싼 후 가지고 있기 때문이다.

이후에 이 변수의 메서드가 evaluate될 때 비로소 reverse 함수가 실행되어 본래의 값을 반환하게 된다.


