---
layout: post
title: "Counting Inversion 알고리즘 구현"
description: "Counting Inversion 알고리즘을 Python으로 구현하기"
tags: [algorithm, python]
---

# 목표 정의
주어진 Array의 [Inversion쌍의 갯수를 구하는 알고리즘](inversion-count-algorithm)을 Python을 사용하여 구현하고 검증한다. 다만 편의를 위해 증복값은 존재하지 않는다고 가정한다.

# Pseudocode
이전 포스팅에서 본 바와 같이 우리는 `SortAndCountInversion`, `MergeAndCountInversion` 2개의 함수가 필요하다는 것을 알고 있다. 해당 Pseudocode를 수정하여 Python으로 구현해본다.

# Python code 작성
Pseudocode에서는 편의상 각 함수가 Array 의 length를 인자로 받았다. 그러나 함수 내부에서 Array를 이용하여 length를 구할 수 있으므로 구현에서는 제외한다.

{% highlight python %}
def sortAndCountInversion(arr):
    """
    arr를 중점을 기준으로 좌,우로 분할하여 각각의 inversion의 갯수를 센다.
    split inversion의 경우 merge sort를 응용하여 갯수를 센다.
    """
    length = len(arr)

    # Initialize
    if length == 1:
        return arr, 0

    # Divide & Conquer
    a, b = arr[:length//2], arr[length//2:]
    sorted_a, left_inversions = sortAndCountInversion(a)
    sorted_b, right_inversions = sortAndCountInversion(b)
    sorted_arr, split_inversions = mergeAndCountInversion(sorted_a, sorted_b)

    # Combine
    total_inversions = left_inversions + right_inversions + split_inversions

    return sorted_arr, total_inversions

{% endhighlight %}

- 길이가 1개인 배열이 들어오면 바로 반환하고 inversion은 당연히 0이다
- 중점을 기준으로 분할한다. 물론 홀수갯수인 경우 비대칭분할이 될 것이다.
- 각각을 재귀적으로 계산하여 left_inversions과 right_inversions을 구한다
- 정렬된 좌,우 배열을 이용하여 split_inversions의 갯수를 구하고 merge sort로 정렬한다.
- 완전히 정렬된 배열과 전체 inversion 갯수를 반환한다.

{% highlight python %}
def mergeAndCountInversion(a, b):
    """
    정렬된 두개의 배열 a, b 를 받아 완전히 정렬된 배열 c와 a, b사이에 존재하는 split inversion 갯수를 반환한다.
    """
    lena, lenb = len(a), len(b)
    sorted_arr = []
    total_split_inversion = 0

    # a, b의 원소를 가리킬 포인터 변수 초기화
    i, j =(0,)*2
    for k in range(lena + lenb):
        # i, j중 하나가 끝에 도달했다면 나머지 배열 중 남은 원소들을 그대로 이어붙인다
        if i == lena:
            sorted_arr.extend(b[j:])
            break
        if j == lenb:
            sorted_arr.extend(a[i:])
            break

        if a[i] < b[j]:
            sorted_arr.append(a[i])
            i+=1
        elif a[i] > b[j]:
            sorted_arr.append(b[j])
            j+=1
            total_inversions += (lena-i)
    return sorted_arr, total_inversions
{% endhighlight %}

- Caller인 `sortAndCountInversion`는 Callee에게 비대칭적인 분할 배열을 넘겨줄 수 있다. 따라서 input array 각각의 길이를 저장해야 한다.
- 나머지는 일반적인 merge sort 알고리즘과 동일하다.
- 다만 배열 `b`에서 결과 배열 `sorted_arr`로의 복사가 일어난 경우, 그 시점에서 남아있는 `a`의 원소 갯수만큼 `total_split_inversion`에 더해준다.
- 최종적으로 merge sort로 정렬된 배열과 `total_split_inversion`을 리턴한다.


# Test
`sortAndCountInversion` 함수가 주어진 array에 대해 inversion의 갯수를 잘 리턴하는지 테스팅한다. 테스팅할 때 `sortAndCountInversion(testArray)[1]` 처럼 해도 되겠지만 편의상 로직을 감싸는 새로운 함수를 생성하여 테스팅하도록 한다.

{% highlight python %}
import unittest
import random

def countInversionWrapper(arr):
    return sortAndCountInversion(arr)[1]


class TestCountInversion(unittest.TestCase):
    def test_single(self):
        self.assertEqual(countInversionWrapper([3]), 0)
        self.assertEqual(countInversionWrapper([1913]), 0)

    def test_ascending(self):
        self.assertEqual(countInversionWrapper([1,2,3,4,5]), 0)
        self.assertEqual(countInversionWrapper([1,3,5,7,9]), 0)
        self.assertEqual(countInversionWrapper([i for i in range(100, 9999)]), 0)

    def test_descending(self):
        self.assertEqual(countInversionWrapper([4,3,2,1]), 6)
        self.assertEqual(countInversionWrapper([10,9,8,7,6,5,4,3,2,1]), 45)
        self.assertEqual(countInversionWrapper([i for i in range(10000, 0, -1)]), 49995000)

    def test_random(self):
        self.assertEqual(countInversionWrapper([1,3,5,2,4,6]), 3)
        self.assertEqual(countInversionWrapper([1,6,3,2,4,5]), 5)
        self.assertEqual(countInversionWrapper([9, 12, 3, 1, 6, 8, 2, 5, 14, 13, 11, 7, 10, 4, 0]), 56)

if __name__ == '__main__':
    # unittest.main(exit=False) # Python shell에서 실행할 경우 exit=False 옵션을 넣어준다.
    unittest.main()
{% endhighlight %}

- 주어진 배열이 오름차순일 경우 inversion은 없어야 한다.
- 주어진 배열이 내림차순일 경우 배열의 길이가 n이라면, inversion의 갯수는 $$\frac{n(n-1)}{2}$$ 개이다.
- 기타 랜덤하게 주어진 배열들을 테스트해본다.

결과는 다음과 같다.

{% highlight bash %}
....
----------------------------------------------------------------------
Ran 4 tests in 0.090s

OK
{% endhighlight %}

이로써 검증이 완료되었다.

