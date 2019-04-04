---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.5 (2)"
description: "Django template 완전 구현"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기 Part.5 (2)

1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV)
3. View develop하기
4. Django template 기본 구현
5. Django template 완전 구현
    1. 상용 서비스와 같은 디자인 골격 잡기
    2. 수정/삭제 Dropdown 및 댓글창 예쁘게 만들기 **(current)**
    3. 기타 짜잘한 디자인 완성시키기
    4. AJAX를 이용한 기본적인 댓글 시스템 구현
6. 좀더 멋있는 기능 추가하기
    1. Infinite scroll 구현하기
    2. 사진 Carousel 붙이기

<hr>

## 1. 내부를 멋있게 만들기
큰 틀의 디자인을 잡았으므로 남은 것은 짜친 것들 뿐이다. 아무 디자인도 하지 않았을때보단 낫지만 여전히 우리는 볼때마다 눈갱을 당하고 있다. 무턱대고 들어가기 전에 뭘 해야 하는지 리스트업을 해보자. 우린 View를 작성할 때 리스트업을 해둔 것들이 있었다. 가져와보자.

#### Frontend Checklist

- [ ] 1. `InstagramPhoto` 객체들을 뿌리는 형식은 carousel이어야 한다.
- [ ] 2. 댓글 입력 input 버튼은 따로 없고 한줄로 input이 표시되어야 한다.
- [ ] 3. `Instagram` 객체에는 편집/삭제 버튼이 위치해야 한다.
- [ ] 4. 편집/삭제버튼은 우측 상단 또는 우측 하단의 숨겨진 dropdown안에 있어야 한다.
- [ ] 5. 댓글 입력창에서 엔터를 치면 댓글이 자동 업데이트 되어야 한다.
- [ ] 6. 어느정도 스크롤을 내렸으면 다음 인스타그램 객체들을 로드해야 한다.(무한스크롤)


이 중에서 가장 시급해보이는 것은 2,3,4번인 듯 싶다. 작성자 옆에 시퍼런 색으로 수정하기/삭제하기가 대놓고 나와있고, 댓글쓰는 input창은 쓸데없이 크다. 먼저 이것들을 잡아보자.


## 2. Dropdown 구현하기

우리가 원하는 것은 버튼 하나를 누르면 드롭다운 형태로 편집/삭제 버튼을 나타나게 하는 것이다. 이는 이미 모바일웹이나 네이티브앱에서 굉장히 많이 쓰이는 UI디자인으로 사용자들도 쉽게 친숙해질 수 있다. 이를 위해서는 javascript를 피해갈 수 없다. 본 절은 [w3school 튜토리얼](https://www.w3schools.com/howto/howto_js_dropdown.asp)을 기반으로 작성되었다.


먼저 템플릿을 보자

```django
{% raw %}
<div class="col-3">
    {# 드롭다운 옵션들을 넣어줄 컨테이너 #}
    <div class="dropdown-options">
        {# 먼저 아이콘을 뿌려준다. #}
        <i class="fas fa-ellipsis-v"></i>
        <div class="dropdown-content text-center">
            {# 수정하기는 단순히 수정 view로 리다이렉트한다 #}
            {# TODO: Complete redirection #}
            <a href="#">수정하기</a>
            {# 삭제하기는 아예 form으로 넘겨서 삭제 확인창 없이 한번에 처리한다. #}
            <form action="#" method="post">{% csrf_token %}
                {# 어디서 많이 본 패턴이다. #}
                <input type="hidden" name="next" value="{{ request.path }}">
                <a href="#" class="delete-instagram">삭제하기</a>
            </form>
        </div>
    </div>
</div>
{% endraw %}
```

- `<div class="dropdown-options">` 내부에 `<div class="dropdown-content">`가 있고 이 안에 `수정하기`와 `삭제하기`가 존재한다.
-`<div class="dropdown-content">` 는 평소에는 보이지 않아야 한다.
- `<div class="dropdown-options">` 를 클릭하면 `<div class="dropdown-content">`가 보여야 한다.
- 드롭다운이 열린 후에는 바깥쪽이나 `<div class="dropdown-options">` 를 다시 클릭하면 드롭다운이 닫혀야 한다.
- 기왕이면 드롭다운 버튼은 세로방향으로 정렬된 '...' 모양이면 좋을거 같다.

### 2.1. 모양잡기
일단 저 fontawesome 아이콘을 우측으로 밀어주자. Bootstrap에 내장된 class를 이용한다.


```django
{% raw %}
<!-- text-right 클래스를 넣어준다  -->
<div class="col-3 text-right">
    {# 드롭다운 옵션들을 넣어줄 컨테이너 #}
    <div class="dropdown-options">
        {# 먼저 아이콘을 뿌려준다. #}
        <i class="fas fa-ellipsis-v"></i>
        <div class="dropdown-content text-center">
            {# 수정하기는 단순히 수정 view로 리다이렉트한다 #}
            {# TODO: Complete redirection #}
            <a href="#">수정하기</a>
            {# 삭제하기는 아예 form으로 넘겨서 삭제 확인창 없이 한번에 처리한다. #}
            <form action="#" method="post">{% csrf_token %}
                {# 어디서 많이 본 패턴이다. #}
                <input type="hidden" name="next" value="{{ request.path }}">
                <a href="#" class="delete-instagram">삭제하기</a>
            </form>
        </div>
    </div>
</div>
{%endraw %}
```

- 우리는 드롭다운 및 아이콘을 첫번째 row의 col-3안에 넣어주고 있다.
- col-3의 내용물들을 우측정렬하기 위해 `text-right` 클래스를 넣어준다.



다음을 `feed_list_style.css`에 추가한다.

```css
.dropdown-options {
    position: relative;
    display: inline-block;
}

.dropdown-content {
    display: none;
    position: absolute;
    background-color: #f1f1f1;
    min-width: 8em;
    overflow: auto;
    box-shadow: 0px 8px 16px 0px rgba(0, 0, 0, 0.2);
    z-index: 1;
}

.show {
    display: block;
}
```

- dropdown-options는 상대적 위치를 가지고 같은 위치의 엘리먼트에 대해서 inline정렬이 되어야 한다.
- dropdown-content는 평소에는 보이지 않아야 한다. (display: none)
- dropdown-content는 상위 엘리먼트인 dropdown-options에 대한 상대적 위치를 가져야 한다. (position: absolute)
- 나머지는 적절히 조절해준다.
- show클래스가 만일 들어간다면 display를 none에서 block으로 바꿔준다.


결과는 다음과 같다.


{% include image.html path="documentation/insta-clone-pt5-10.png" path-detail="documentation/insta-clone-pt5-10.png" alt="image" %}


기존의 수정하기/삭제하기가 없어졌음을 알 수 있다.

당연히 지금은 저 땡땡땡을 클릭해도 아무 반응이 일어나지 않는다. javascript를 넣어서 click listener를 달아주자.

### 2.2. Javascript 기본 세팅
Django에서 Javascript파일은 static 파일로 분류된다. 우리가 추가하려는 click listener는 `feed_list.html`에만 specific하므로 전역 js가 아닌 개별 js로 처리한다.
개별 템플릿에 적용할 파일을 정의하는 것이므로 `index.html`에 자식 템플릿에서 상속받을 javascript block을 정의해야 한다.

```django
<!-- index.html -->

{%raw%}
... 생략
<body>
{% block content %}
    {# TODO: Fill here #}
{% endblock %}

{# ADDED! #}
<!-- Custom javascript-->
{% block javascript %}
{% endblock %}

</body>
</html>
{%endraw%}
```

그리고 `feed_list.html`에서 해당 블록을 상속받아둔다.

```django
{%raw%}
{% extends "index.html" %}
{% load static %}
{% load crispy_forms_tags %}

{% block style-head %}
    <link rel="stylesheet" type="text/css" href="{% static 'instagrams/css/feed_list_style.css' %}">
{% endblock %}

{% block content %}
...생략
{% endblock %}


{# ADDED! #}
{% block javascript %}
{% endblock %}
{%endraw%}
```

이제 해당 블록에 추가할 javascript파일을 `instagrams/static/instagrams/js/`경로에 추가한다. 네이밍 컨벤션은 `<적용할 html 파일 이름>_js.js`로 하기로 하자.

이제 자바스크립트 작업을 위한 모든 준비과정이 끝났다.

### 2.3. javascript 작성

이제 자바스크립트를 작성하자. 편의상 jquery를 사용한다. jquery는 아주 기초적인 selector만 사용한다.

로직은 간단하다.

1. `i`를 클릭하면 `show`라는 클래스를 가장 가까운 `dropdown-content`에 동적으로 추가한다.
2.  Instagram 객체들은 한 페이지에 여러개가 있으므로 `dropdown-options`를 클릭한 경우 가장 **가까운** `dropdown-content`를 열어줘야 한다.
3. Dropdown을 열어준 후 아예 밖을 클릭하거나 다시 dropdown을 클릭하면 `show`클래스를 없애서 다시 안보이게 해준다.

이를 구현하면 다음과 같다.

```javascript
// instagrams/static/instagrams/js/feed_list_js.js

$(document).ready(function () {
    // icon을 클릭한 것이 아니라면 드롭다운을 닫아준다
    $(window).on('click', function (event) {
        let allDropdowns = document.getElementsByClassName("dropdown-content")
        let targetDropdown = $(event.target).next('.dropdown-content')[0]
        // map 함수를 이용하기 위해 Array로 치환한다
        let dropdownArr = [...allDropdowns]

        // 클릭한게 icon이 아닌 경우 window를 클릭한 것이다.
        if (!event.target.matches('i')) {
            // show가 있는 div를 닫아준다
            dropdownArr.map(d => {
                if (d.classList.contains('show')) d.classList.remove('show')
            })
        } else {
            //클릭한게 icon인 경우
            if (targetDropdown.classList.contains('show')) {
                // 만일 클릭한 icon으로부터 가장 가까운 dropdown-content가 열려있으면 단순히 닫아준다.
                targetDropdown.classList.remove('show')
            } else {
                // 만일 클릭한 icon으로부터 가장 가까운 dropdown-content가 닫혀있으면 일단 모든 dropdown을 닫아준다.
                dropdownArr.map(d => d.classList.remove('show'))
                // 그 후 클릭한 icon으로부터 가장 가까운 dropdown-content를 열어준다.
                targetDropdown.classList.toggle('show')
            }
        }
    })
})

```
- document 로딩이 모두 끝났으면 로직들을 실행한다.
- 클릭 이벤트가 발생하면 해당 이벤트를 검증한다.
- 일단 현재 페이지의 모든 dropdown들과 (만일 icon이 클릭되었다면 타겟이 될) dropdown을 저장하고 적절히 타입을 변환한다.
- 클릭한 element가 icon이 아닌 경우 window를 클릭한 것이므로 모든 dropdown을 닫아준다.
- 클릭한 element가 icon인 경우 분기를 나눈다.
- 만일 타겟 dropdown이 열려있는경우(show 클래스 존재) 단순히 닫아주기만 하면 된다.
- 만일 타겟 dropdown이 닫혀있는경우(show 클래스 없음) 다른 곳에 **열린 dropdown**이 존재한단 뜻이다.
- 모든 dropdown을 닫아주고, 타겟 dropdown을 열어준다.


여기까지하고 저장한 후 새로고침해보자.


{% include image.html path="documentation/insta-clone-pt5-11.gif" path-detail="documentation/insta-clone-pt5-11.gif" alt="image" %}


직관적으로 잘 동작하는 것을 볼 수 있다.

### 2.4. Instagram 수정/삭제 연결하기

수정/삭제 연결하는 것은 매우 쉽다. 이미 우리는 url 라우팅을 완료했기 때문이다. `feed_list.html`을 다음과 같이 수정한다.

```django
{% raw %}
...전략
 <div class="dropdown-options">
    {# 먼저 아이콘을 뿌려준다. #}
    <i class="fas fa-ellipsis-v"></i>
    <div class="dropdown-content text-center">
        {# 수정하기는 단순히 수정 view로 리다이렉트한다 #}
        <a href="{% url 'instagrams:feed-update' instagram.pk %}">수정하기</a>
        {# 삭제하기는 아예 form으로 넘겨서 삭제 확인창 없이 한번에 처리한다. #}
        <form action="#" method="post">{% csrf_token %}
            {# 어디서 많이 본 패턴이다. #}
            <input type="hidden" name="next" value="{{ request.path }}">
            <a href="{% url 'instagrams:feed-delete' instagram.pk %}" class="delete-instagram">삭제하기</a>
        </form>
    </div>
</div>
...후략
{% endraw %}
```

- Django가 제공하는 url reverse를 이용한다.
- 위 코드 snippet은 현재 context variable로 넘어온 `instagrams`를 for loop을 돌면서 각 원소를 `instagram`으로 지칭하고 있음을 유의한다.
- url 인자로는 각 instagram의 pk를 넘겨준다.


이제 새로고침을 하고 드롭다운을 열어서 **수정하기**를 눌러보자.

{% include image.html path="documentation/insta-clone-pt5-14.png" path-detail="documentation/insta-clone-pt5-14.png" alt="image" %}


저기서 아무 사진이나 업로드하고 내용을 바꿔서 올리기 버튼을 눌러보자.
올리기를 누르는 순간 현재 instagram객체에 연결된 `InstagramPhoto`는 삭제되고 새로운 사진 객체가 연결된다. (UpdateView 참조)

{% include image.html path="documentation/insta-clone-pt5-15.png" path-detail="documentation/insta-clone-pt5-15.png" alt="image" %}

잘 수정된 것을 알 수 있다.


그럼 삭제하기도 잘 되겠지... 하고 삭제 버튼을 눌러보면 다음과 같은 에러가 발생할 것이다.


{% include image.html path="documentation/insta-clone-pt5-16.png" path-detail="documentation/insta-clone-pt5-16.png" alt="image" %}

에러메세지 자체는 단순하다. DJango는 기본적으로 DeleteView를 사용할 때  suffix가 `confirm_delete.html`인 **삭제확인 페이지**를 거치도록 하고 있다.
우리는 alert창으로 삭제 여부를 확인한 후 바로 삭제하고 싶다. 어떻게 해야 할까?

`DeleteView`의 소스코드를 뒤져보면 다음과 같은 사실을 알 수 있다.

- `DeleteView`는 POST와 GET메소드를 받는다.
- GET으로 뷰를 호출한 경우 `confirm_delete.html`을 부른다.
- POST로 호출한 경우 단순히 객체를 지워준다.

template에서 우리는 `{%raw%}<a href="{% url 'instagrams:feed-delete' instagram.pk %}">{%endraw%}`로 `DeleteView`를 접근하였으므로 DJango는 `confirm_delete`를 띄우게 된 것이다. 따라서 해당 url을 form `action`에서 직접 접근함으로써 POST 리퀘스트를 보내야 한다.

> Django DeleteView는 GET으로 접근하면 suffix가 `confirm_delete.html`인 페이지를 띄워주고 POST로 접근하면 객체를 지워준다.

`feed_list.html`에서 다음과 같이 수정한다.

```django
{%raw %}
{# 삭제하기는 아예 form으로 넘겨서 삭제 확인창 없이 한번에 처리한다. #}
<form action="{% url 'instagrams:feed-delete' instagram.pk %}" method="post">{% csrf_token %}
    {# 어디서 많이 본 패턴이다. #}
    <input type="hidden" name="next" value="{{ request.path }}">
    <input type="submit" onclick="return confirm('정말 삭제하시겠습니까?')" class="delete-instagram btn-link" value="삭제하기"></input>
</form>
{% endraw %}
```

- 기존의 a 태그에 있던 delete url을 form action으로 옮긴다.
- form은 submit버튼이 있어야 한다. class는 `btn-link`를 추가하여 '수정하기'의 a 태그와 **스타일을 일치시킨다** (물론 반대로 해도 상관없다)
- input의 onclick 세팅으로 confirm을 체크하는 간단한 js를 삽입한다. 만일 confirm이 아니라면 해당 이벤트는 진행되지 않을 것이다.


저장한 후 새로고침 해보자. Instagram객체를 지워보자. confirm창이 잘 뜬 후 지워질 것이다. 지운 후에는 View에 정의한 대로 hidden input에 담겨온 url(feed-list페이지)로 리다이렉트할 것이다.


#### Frontend Checklist

- [ ] 2. 댓글 입력 input 버튼은 따로 없고 한줄로 input이 표시되어야 한다.
- [x] 3. `Instagram` 객체에는 편집/삭제 버튼이 위치해야 한다.
- [x] 4. 편집/삭제버튼은 우측 상단 또는 우측 하단의 숨겨진 dropdown안에 있어야 한다.


## 3. Comment input form 예쁘게 하기
이번 일은 매우 쉽다. 우리는 `comments`앱에서 렌더링할 form의 class를 `textfield-comment`로 지정한 바 있다. `comments/forms.py`를 참조하자.
따라서 css에 `.textfield-comment`셀렉터에 다음과 같이 입력한다. css 스타일은 이전절에서와 같이 상용 Instagram의 댓글 입력창을 분석하였다.

그리고 댓글을 달기 위한 input 버튼은 아예 숨겨버리자. 나중에 javascript로 엔터를 누르면 해당 버튼이 눌리도록 처리할 것이다.

```css
.textfield-comment {
    display: flex;
    margin-top:10px;
    padding: 0;

    border: 0;
    background: 0 0;
    color: #262626;
    height: 20px;
    max-height: 80px;
    font-size: inherit;
    outline: 0;
    resize: none;
}

.input-comment-create {
    display: none;
}

```
 - height이나 max-heigth의 경우는 원하는 만큼 조절한다.


이후 다시 새로고침을 눌러보자.


{% include image.html path="documentation/insta-clone-pt5-12.png" path-detail="documentation/insta-clone-pt5-12.png" alt="image" %}


이전에 비해 훨씬 깔끔해졌다.


#### Frontend Checklist
- [x] 2. 댓글 입력 input 버튼은 따로 없고 한줄로 input이 표시되어야 한다.
- [x] 3. `Instagram` 객체에는 편집/삭제 버튼이 위치해야 한다.
- [x] 4. 편집/삭제버튼은 우측 상단 또는 우측 하단의 숨겨진 dropdown안에 있어야 한다.


## 4. 정리
- jQuery와 javascript를 이용하여 DOM 및 css 스타일을 적절하게 변환할 수 있다.
- 댓글을 달기 위한 input 및 버튼도 상용 서비스처럼 예쁘게 만들 수 있다.

다음 절에서는 기타 짜친 디자인들을 완성시킨다.
