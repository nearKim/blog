---
layout: post
title: "Javascript의 function call"
description: "Javascript에서는 어떻게 parameter를 다루는가?"
tags: [javascript]

---

# 0. 문제의 시작
모든 것은 다음의 React 스니펫으로부터 시작되었다.

{% highlight javascript %}
// ...전략

componentDidMount() {
    // users를 받아와서 state에 저장한다
    fetch("https://jsonplaceholder.typicode.com/users/")
      .then(response => response.json())
      .then(users => this.setState({ users }));
  }

  deleteUserEmail(users) {
    return users.map(u => {
      delete u.email;
      return u;
    });
  }

  render() {
    let deletedEmailUsers = this.deleteUserEmail(this.state.users)
    return (
      <div>
        <UserWithoutEmail users={deletedEmailUsers}/>

        {this.state.users.map((user, i) => {
          return (
          <div key={i}>
          Name, Email: ({user.name}, {user.email})

          </div>);
        })}

      </div>
    );
  }
{% endhighlight %}

원하는 바는 명확하다.

- 컴포넌트가 마운팅되었을 때 api를 통해 User 리스트들을 받아온다.
- `UserWithoutEmail`이라는 컴포넌트에 email을 지운 User 리스트를 넘겨준다
- 그 아래에는 '정상적인' User 리스트로부터 name, email만 추출하여 tuple로 뿌려준다.

그러나 위 스니펫은 다음과 같은 결과를 보여준다.

{% highlight markdown %}
// UserWithoutEmail 컴포넌트는 생략

Name, Email: (Leanne Graham, )
Name, Email: (Ervin Howell, )
Name, Email: (Clementine Bauch, )
Name, Email: (Patricia Lebsack, )
Name, Email: (Chelsey Dietrich, )
{% endhighlight %}

나는 분명 정상적인 User 리스트를 넘겼다고 생각했는데 email이 지워져 있다. 왜 이런 현상이 발생한 것일까?

# 1. Pass by value
Javascript는 function parameter로 `Primitive type`인 값이 넘어왔을 때는 그 값을 복사하여 dummy variable에 저장한 채로 내부 스코프로 전달한다. 따라서 function 외부에서 선언된 `Primitive type` 변수는 값이 변하지 않는다.

{% highlight javascript %}
function callByValueTest(x, y) {
    x = 1
    y = 2
    console.log("x:", x, ", y:", y)
}

let a = 10
let b = 20

callByValueTest(10, 20) // x:1, y:2
console.log("a:", a, ", b:", b) // a:10, b:20
{%endhighlight %}

위 코드에서 자바스크립트 엔진은 callByValueTest 함수가 받은 a, b 값을 복사하여 x, y에 전달한다.
