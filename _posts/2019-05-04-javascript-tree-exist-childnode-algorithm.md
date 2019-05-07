---
layout: post
title: "Tree 구조에서 특정 Node의 후손 중 특정 id를 가진 Node의 존재 여부 알아내기"
description: "Javascript를 이용하여 Tree의 어떤 Node의 모든 하위 Node 중에 특정한 id를 가진 Node가 존재하는지 검사하기"
tags: [javascript, algorithm]

---

회사에서 `React`를 활용한 업무 중 기본적인 알고리즘 문제를 구현 할 기회가 있어서 적는 경험담.

# 1. 문제 정의
제반 상황은 [이전 포스팅](javascript-tree-find-node-algorithm) 과 동일하다.


이 때, 임의의 `id`와 `Node`객체가 주어졌을 때, 해당 Node의 모든 Child 중 해당 id를 가진 node가 존재하는지 검사한다.


# 2. Pseudocode 작성

```
function containsTargetId(node, targetId)
  // 1.
  if (node.id == targetId) return true

  // 2.
  if (len(node.child) == 0) return false

  // 결과 변수 초기화
  result = false

  // 3.
  for (child in node.child) {
    // 4.
    result |= this.containsTargetId(child, targetId)
  }

  return result
```

1. node.id 가 targetId이면 true이다
2. node.child의 길이가 0 이면 false 이다
3. node.child 중에서 id를 찾는다.
4. child 각각의 원소에 대해 existsChildId를 적용하고 결과를 모두 or로 묶는다

# 3. Javascript code 작성

굳이 for문을 돌 필요 없이 간편하게 `.forEach()`를 이용하면 된다.


{% highlight javascript %}
function containsTargetId(node, targetId) {
  // node.id 가 targetId이면  true이다
  if (node.id === targetId) return true

  // node.child의 길이가 0 이면 false 이다
  if (node.child.length === 0) return false

  // 결과 변수 초기화
  result = false
  // child 각각의 원소에 대해 existsChildId를 적용하고 결과를 모두 or로 묶는다
  node.child.forEach(c => {
    result |= this.containsTargetId(c, targetId)
  })

  return result
}
{% endhighlight %}








