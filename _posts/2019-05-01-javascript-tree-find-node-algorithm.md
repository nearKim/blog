---
layout: post
title: "Tree 구조에서 특정 Node 찾기"
description: "Javascript를 이용한 Tree 자료구조에서 특정 ID를 가지는 Node 탐색하기"
tags: [javascript, algorithm]

---

회사에서 `React`를 활용한 업무 중 기본적인 알고리즘 문제를 구현 할 기회가 있어서 적는 경험담.

# 1. 문제 정의
API는 전체 데이터를 Tree 구조로 Front에 넘겨준다. 한 Node는 javascript object이고, `id`, `name`,`parent_id`, `child`의 어트리뷰트를 갖고 있다. 이 경우 Binary Tree가 아님에 유의한다.

- `id` : Node의 id. 모든 Tree node에 대해 id는 고유하다.
- `name` : Node의 이름. React에서는 name을 유저들에게 표시해야 한다.
- `parent_id` : Node의 부모 Node의 id이다.
- `child` : 현재 Node를 부모로 가지는 자식 Node들로 이루어진 Array. Leaf node라면 빈 Array이다.


{% include image.html path="documentation/tree-path-algorithm-1.png" path-detail="documentation/tree-path-algorithm-1.png" alt="image" %}

- 현재 알고리즘에서는 `parent_id`는 필요 없으므로 표시하지 않는다

API는 root Node를 제외하고 Level 1인 Node들로 이루어진 Array를 넘겨준다. 이를 `allTrees`라는 변수에 저장하고 있다고 가정한다.

이 때, 임의의 `id`가 주어졌을 때, 해당 id를 가지는 Node를 반환한다. 물론, 주어진 id는 root Node의 id는 아니며 Tree의 node들 중 반드시 존재한다고 가정한다.


# 2. Pseudocode 작성
```
function getTreeNode(curLevelNodes, targetId)

  nextLevelNodes = []
  for (node in curLevelNodes) {
    if (node.id == targetId) return node
    if (len(node.child) != 0) {
      nextLevelNodes.merge(node.child)
    }
  }
return getTreeNode(nextLevelNodes, targetId)
```

1. 현재 Level의 node 중에 찾는 node가 있으면 바로 반환한다.
2. 다음 Level의 node들(현재 Level의 node들 각각의 child의 합집합)을 싹다 모은다. 이 때 빈값은 제거해야 한다.
3. 재귀적으로 id를 찾는다

# 3. Javascript code 작성
위 유사코드는 2가지 중심적인 '명령'을 사용한다.

- 주어진 Array 중 특정한 id를 가진 원소를 `찾는다`
- 주어진 Array의 각 원소에 존재하는 Array들을 한 곳으로 `모은다`

위는 각각 `.find()` , `.flatMap()` 함수를 사용하면 쉽게 구현 가능하다.

{% highlight javascript %}
function getTreeNode(curLevelNodes, targetId) {
  // 현재 Array level에 찾는 node가 존재하면 바로 반환한다.
  let node = curLevelNodes.find(n => n.id === targetId)
  if (node) return node

  // 다음 Level의 node들을 모은다. 이 때 undefined는 삭제한다.
  nextLevelNodes = curLevelNodes
      .flatMap(n => n.child)
      .filter(c => c)

  // 재귀적으로 id를 찾는다
  return this.getTreeNode(nextLevelNodes, targetId)
}
{% endhighlight %}

`this.getTreeNode(allTrees, 12)`와 같은 방식으로 사용하면 된다.






