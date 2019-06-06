---
layout: post
title: "Quicksort 알고리즘 구현"
description: "Quicksort 알고리즘을 Python을 이용하여 구현하기"
tags: [algorithm, python]
---

# 목표 정의
랜덤하게 선택된 pivot을 사용한 [Quicksort 알고리즘](quicksort-algorithm)을 Python을 사용하여 구현한다. 중복값은 고려하지 않는다.

# Python code 작성

### Helper function
Quicksort 알고리즘에서는 배열의 원소를 swap하는 경우가 빈번하게 발생한다. 이에 따라 코드 구현 전에 Helper 함수를 작성한다.

{% highlight python %}
def swap(arr, i, j):
    tmp = arr[i]
    arr[i] = arr[j]
    arr[j] = tmp
{% endhighlight %}

전형적인 학부과정 swap 함수이다. Python에서는 `list`가 함수 parameter로 전달될 경우 reference가 전달된다. 따라서 function 내부에서 list의 변경이 일어난다면 reference가 가리키는 value가 직접 변경된다. 따라서 swap 함수에서 새 arr를 반환할 필요가 없다.

* 만일 부작용(Side effect)이 없는 함수를 작성하고 싶다면 arr를 `deepcopy`한 후 swap하고 deepcopy 된 배열을 리턴하면 된다.

### Quicksort
전형적인 분할정복 알고리즘을 구현할 때와 같이 함수를 구현한다.

다만 편의를 위하여 <b>pivot이 정해진 뒤 곧바로 배열의 첫번째 원소와 swap한다고 가정</b>한다. 이는 1회의 추가 연산이지만, 실행속도에 큰 영향을 주지 않는 범위에서 코드의 복잡도를 줄여준다.

{% highlight python %}
import random

def quicksort(arr):
    if len(arr) < 2:
        return arr

    pivot_idx = random.randint(0,len(arr)-1)
    swap(arr, 0, pivot_idx)
    pivot = arr[0]

    i = 1
    for j in range(1, len(arr)):
        if arr[j] < pivot:
            swap(arr, i, j)
            i+=1
    # Pivot swap
    swap(arr, 0, i-1)

    # Divide & Conquer
    left_arr = quicksort(arr[:i-1])
    right_arr = quicksort(arr[i:])

    total_arr = left_arr + [pivot] + right_arr

    return total_arr
{% endhighlight %}

- 가능한 arr의 인덱스 중 하나를 랜덤하게 pivot으로 선택한다.
- 편의를 위해 pivot을 현재 arr의 첫번째 원소와 swap 한다.
- arr를 선형탐색하면서 pivot보다 작은 원소는 swap을 통해 좌측 부분배열에 넣어준다.
- 선형탐색이 종료되었으면, 좌측 부분배열의 마지막 원소와 pivot을 swap한다. (Partition의 종결)
- 좌,우측 부분배열 각각 recursive하게 적용한다


# Test
새로운 Wrapper 함수를 생성하여 unittest를 수행한다. Python에 기본으로 내장된 `sort()`와 비교하면 될 것이다.

{% highlight python %}
import unittest
import random

class TestQuicksort(unittest.TestCase):
    def test_single(self):
        self.assertEqual(quicksort([]), [])
        self.assertEqual(quicksort([3]), [3])
        self.assertEqual(quicksort([193]), [193])
    def test_sort(self):
        for i in range(100):
            test_list = random.sample(range(1000), 10)
            ### WARNING! ###
            self.assertEqual(quicksort(test_list), sorted(test_list))


if __name__ == '__main__':
    # unittest.main(exit=False) # Python shell에서 실행할 경우 exit=False 옵션을 넣어준다.
    unittest.main()
{% endhighlight %}
- 원소가 1개이거나 없는 극한값을 테스트한다
- 랜덤하게 100개의 원소를 가지는 list를 생성하고 Python 기본 내장 함수와 비교한다.

결과는 다음과 같다.

{% highlight bash %}
....
----------------------------------------------------------------------
Ran 2 tests in 0.090s

OK
{% endhighlight %}

모두 잘 진행되었다.

...라고 생각하면 <b>안된다!</b>

### Side effect
위 테스트 과정에는 심각한 결함이 있다. 현재 `quicksort()`함수는 부작용이 존재하는 함수이다. 상술했듯이 parameter로 전달되는 list는 reference가 전달되는데, 이에 따라 함수 내부에서 arr의 원소를 변경하는 모든 과정은 전달된 reference가 가리키는 value를 직접 변형시킨다. 따라서 `WARNING!` 아래의 `quicksort(test_list)`는 `test_list`를 변형시킨다. 따라서 이후 `sorted(test_list)`에는 변형된 배열이 전달되게 된다.

이 경우 `quicksort()`와 `sorted()`에 동일한 array가 들어가지 않으므로 우리가 원하는 테스팅이 이루어지지 않는다. 물론 위 알고리즘이 정확하고, 잘 구현되었으므로 테스트 결과는 OK일 것이지만 엄밀하게는 잘못된 테스팅을 하고 있는 것이다.

이를 해결하기 위해서는 `quicksort()` 함수 자체를 실행할 때 input으로 들어온 array를 deepcopy하여 사용하거나, 애초에 테스터에 던져지는 변수를 deepcopy하여 던지면 된다. 본 절에서는 후자를 선택하지만 일반적으로 전자가 바람직하다.

{% highlight python %}
# ... 전략
    def test_sort(self):
        for i in range(100):
            test_list = random.sample(range(1000), 10)
            import copy
            copied_list = copy.deepcopy(test_list)
            self.assertEqual(quicksort(copied_list), sorted(test_list))
{% endhighlight %}

예쁘진 않지만 로직은 정확하다.

Quicksort 알고리즘 구현 및 검증이 완료되었다.

