---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.5 (4)"
description: "AJAX를 이용한 기본적인 댓글 시스템 구현"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기 Part.5 (4)

1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV)
3. View develop하기
4. Django template 기본 구현
5. Django template 완전 구현
    1. 상용 서비스와 같은 디자인 골격 잡기
    2. 수정/삭제 Dropdown 및 댓글창 예쁘게 만들기
    3. 기타 짜잘한 디자인 완성시키기
    4. AJAX를 이용한 기본적인 댓글 시스템 구현 **(current)**
6. 좀더 멋있는 기능 추가하기
    1. Infinite scroll 구현하기
    2. 사진 Carousel 붙이기

<hr>


## 0. AJAX 없이 댓글 구현하기
현재 댓글기능은 80%정도 구현되어 있다. Template의 댓글 입력 form의 action을 `comments:comment-create` url과 연결하고 javascript로 엔터키 입력 여부에 따라 data를 넘겨주면 그만이다. 상용 서비스에는 댓글의 수정/삭제 기능이 없어서 개인적으로 매우 불편했다. 댓글 수정/삭제 버튼도 넣어주자.

예컨대 다음과 같은 형식이 될 것이다.

```django
{% raw %}
... 전략
    {# 7. 7번째 row: 댓글 입력 form이 들어간다#}
    <div class="comment-creator-div row" >
        <div class="comments-form col-12">
            <!-- ADDED! -->
            <form action="{% url 'comments:comment-create' feed_pk=instagram.pk %}" method="post">{% csrf_token %}
                {{ comment_form | crispy }}
                <input type="hidden" name="next" value="{{ request.path }}">
                <input class="input-comment-create" style="display: none;" type="submit" class="btn btn-secondary"/>
            </form>
        </div>
    </div>

...후략

{% endraw %}
```
- `action`에는 CommentCreate URL이 들어간다.
- 댓글이 달릴 인스타그램 객체의 pk를 parameter로 같이 넘겨준다.
- 댓글이 달린 후 이전페이지로 리다이렉트 해야 하므로 `next`라는 이름으로 현재 path를 넣어준다.

이제 javascript는 다음과 같을 것이다.

```javascript
...전략
$('textarea').on('keypress', function (event) {
        if (event.which === 13) {
            event.preventDefault()

            if ($(this).val() === '') return
            $(this).closest('form').submit()
        }
})
...후략
```

- '엔터'키의 `keyevent`는 13이다. 이걸로 엔터키가 눌렸음을 알 수 있다.
- 댓글로 아무것도 들어오지 않으면 아무것도 하지 않는다.
- 현재 textarea로부터 가장 가까운 상위 form 을 찾은 후 submit해준다.



{% include image.html path="documentation/insta-clone-pt5-25.gif" path-detail="documentation/insta-clone-pt5-25.gif" alt="image" %}

이런 방식은 매우 간편하지만 몇가지 문제점이 있다.

1. form submit 후에는 새로고침이 되므로 페이지의 맨 처음으로 이동한다. 이를 보정하기 위해선 현재 position을 저장하는 새로운 로직이 필요하다.
2. 페이지가 새로고침되므로 중간에 빈창을 보게되어 사용자 경험면에서 좋지 않다.

위 두가지 고민을 한번에 해결하기 위해선 동적으로 페이지를 로딩하는 방법을 적용해야 한다.


## 1. AJAX로 댓글 달기 기초
AJAX를 이용하면 댓글달기 로직이 어떻게 변화할까?

1. 댓글창에서 댓글을 입력한 후 엔터를 누른다.
2. 댓글을 `POST`로 `CommentCreateView`에게 넘긴다.
3. 댓글창을 비워준다.
4. `CommentCreateView`에서 댓글 생성에 성공했으면 **댓글 리스트를 html로 렌더링하여 리턴**한다.
5. 기존의 댓글 리스트 html을 반환된 댓글 리스트로 바꿔준다.


이를 위해서는 기존의 `CommentCreateView`를 수정해야 한다. 댓글 리스트 html을 아예 바꿔치기할 것이기 때문에 단순히 `next`인자로 넘어온 url로 리다이렉트해서는 안된다.

명시적으로 `render()`메소드를 통해 html을 렌더링하여 넘겨야 한다. 그럼 무슨 html을 넘겨야 할까? 이를 위해서 우린 초창기에 `comments/comments_container.html`을 적어놓았다.

### 1.1. Comment 리스트를 담는 컨테이너 html 작성
**댓글 리스트를 html로 렌더링**할 것이기에 template에서 해당 부분을 따로 추출하는게 좋을 것이다. `comments/templates/comments/comments_container.html`을 만든다.

```django
{% raw %}
{% load static %}

{% for comment in comments.all %}
    <div class="comment {% if request.user == comment.author %} col-9 {% else %} col-12 {% endif %} text-left">
        <p>
            <b>{{ comment.author.username }}</b> &nbsp {{ comment.content }}
        </p>
    </div>
    {% if request.user == comment.author %}
        <div class="comment-button-container col-3 text-right">
            <form action="{% url 'comments:comment-delete' pk=comment.pk %}" method="post">
                {% csrf_token %}
                <input type="hidden" name="next" value="{{ request.path }}">
                <button class="btn comment-delete" onclick="return confirm('정말로 삭제합니까?');">삭제</button>
            </form>
            <form action="{% url 'comments:comment-update' pk=comment.pk %}" method="post">
                {% csrf_token %}
                <button class="btn comment-edit">수정</button>
            </form>
        </div>
    {% endif %}
{% endfor %}
{% endraw %}
```
- `feed_list.html`의 중간에 삽입될 html조각이다. 따라서 `index.html`을 상속받지 않는다.
- `comments`를 돌면서 정보를 뿌려주고, 현재 user가 comment의 author라면 수정,삭제 버튼을 보여준다.
- 만일 현재 작성자가 댓글의 author라면 댓글 내용과 수정/삭제 버튼은 9:3의 비율로 나눠질 것이다. (col-9 col-3)
- 만일 현재 작성자가 댓글의 author가 아니라면 버튼이 보이지 않고 댓글창은 하나의 row를 차지할 것이다. (col-12)

그리고 `feed_list.html`에서 원래 comment container가 될 부분에 위 template을 삽입한다.

```django
{% raw %}
...전략

    {# 5. 5번째 row: 현재 인스타그램에 달린 댓글들이 포함된다다. #}
    <div class="comment-container-div row">
        {% include 'comments/comments_container.html' with comments=instagram.comments %}
    </div>

...후략
{% endraw %}
```
- 우리는 `comments_container.html`에서 댓글들을 `comments`라는 변수로 선언했다.
- 따라서 해당 template을 include할 때 instagram에 포함된 comments들을 그 이름으로 넣어준다.

일단 여기까지 하고 잘 나오는지 새로고침하여 확인해보자.


{% include image.html path="documentation/insta-clone-pt5-26.png" path-detail="documentation/insta-clone-pt5-26.png" alt="image" %}


잘 나온다. 물론 스타일이 안이쁘지만 나중에 고치면 되겠지.

## 2. AJAX에서 사용할 View 작성하기
`comments/views.py`를 다음과 같이 수정한다.

```python
class CommentCreateAjaxView(LoginRequiredMixin, CreateView):
    # 댓글 내용과 분기점을 받아서 적절한 FK를 찾아 연결하고 댓글 목록을 AJAX로 업데이트한다.
    model = Comment
    fields = ['content']
    template_name = 'comments/comments_container.html'

    def form_valid(self, form):
        # 잠깐 db 저장을 멈춘다
        comment = form.save(commit=False)
        # 현재 request를 요청한 user를 댓글의 작성자로 넣어준다
        comment.author = self.request.user
        # 현재 댓글이 달릴 instagram 객체의 pk는 routing rule의 <int:feed_pk>로 넘어온다
        instagram = get_object_or_404(Instagram, pk=self.kwargs.get('feed_pk'))
        comment.instagram = instagram
        comment.save()

        context = {
            'comments': Comment.objects
                .select_related('author')
                .filter(instagram=instagram)
                .order_by('created')
        }

        return render(self.request, 'comments/comments_container.html', context)
```
- Ajax를 사용하는 View임을 클래스 이름을 통해 명시한다.
- 넘어온 `feed_pk`로 찾은 `Instagram`객체에 작성된 모든 댓글들을 context에 넘겨준다. 이 때 순서는 생성일 순서대로 정렬한다. (그래야 나중에 생성된 댓글이 아래로 간다)
- `comments_container.html`에서 댓글들은 `comments`라는 변수에 담겨있기를 기대한다.

이렇게 리팩터한 View에 맞게 url도 바꿔준다. `comments/urls.py`를 다음과 같이 수정한다.

```python
urlpatterns = [
    # Fixed!
    path('create/<int:feed_pk>/', CommentCreateAjaxView.as_view(), name='comment-create'),
    ...후략
]
```

이제 해당 url로 POST를 통해 접근하면 댓글 리스트를 담은 `comments_container.html`이 렌더되어 html로 리턴될 것이다.

## 3. AJAX 함수 완성하기
마지막 구현이다.

### 3.1. 준비하기

일단 이 절을 시작하기 전에 반드시 **jQuery cdn이 slim 빌드가 아닌 것**을 확인한다. slim빌드에는 ajax가 빠져있다.

```html
    <!-- WRONG! -->
    <script src="https://code.jquery.com/jquery-3.1.1.slim.min.js"></script>
    <!-- CORRECT! -->
    <script src="https://code.jquery.com/jquery-3.1.1.min.js"></script>
```

한가지 더 추가해줘야 할 것이 있다. Django에서 form submit을 할 때는 `{%raw%}{% csrf_token %} {%endraw%}`를 삽입하였다. 토큰 없이 바로 AJAX요청을 서버로 보내게 되면 `403 Forbidden`에러가 발생한다.

다음을 `feed_list_js.js`최상단에 추가한다. 굳이 document ready 이후에 넣을 필요는 없다.

```javascript
// Django에서 AJAX로 form post를 할 때 csrf token을 발급해주는 script
// using jQuery
function getCookie(name) {
    var cookieValue = null;
    if (document.cookie && document.cookie != '') {
        var cookies = document.cookie.split(';');
        for (var i = 0; i < cookies.length; i++) {
            var cookie = jQuery.trim(cookies[i]);
            // Does this cookie string begin with the name we want?
            if (cookie.substring(0, name.length + 1) == (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

function csrfSafeMethod(method) {
    // these HTTP methods do not require CSRF protection
    return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
}

```

그리고 document 로딩이 끝났으면 ajax를 초기 setup해준다.

```javascript
...전략

$(document).ready(function () {
    var csrftoken = getCookie('csrftoken');
        $.ajaxSetup({
            beforeSend: function (xhr, settings) {
                if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
                    xhr.setRequestHeader("X-CSRFToken", csrftoken);
                }
            }
        });
        ...후략
})
```

Ajax로 요청을 보낼 때 request header에 CSRF 토큰을 위에서 정의한 메소드를 이용하여 삽입하는 코드이다.

### 3.2. 구현하기

상기 로직의 나머지 부분을 구현하자.

```javascript
...전략


 ...전략
    // 댓글 엔터키 입력
    // 기존의 댓글 엔터키 입력 부분을 아래와 같이 바꿔준다.
    $('textarea').on('keypress', function (event) {
        if (event.which === 13) {
            event.preventDefault()
            // 현재 textarea에 있는 댓글 내용을 가져온다
            let content = $(this).val()
            if (content === '') return

            let url = $(this).closest('form').attr('action')

            // textarea를 비워준다
            $(this).val('')

            $.ajax({
                url: url,
                method: 'POST',
                data: {
                    'content': content
                },
                success: (data) => {
                    // 현재 textarea로부터 가장 가까운 omment-container-div 를 찾고 리턴된 데이터로 교체한다
                    $(this).parents('.comment-creator-div')
                        .siblings('.comment-container-div')
                        .html(data)
                },
                fail: (data) => {
                    alert('댓글 등록에 실패하였습니다. 다시 시도해 주세요.')
                    console.log(data)
                }
            })
        }
    }

```
- 엔터키가 눌렸다면 먼저 해당 `textarea`에 존재하는 내용을 가져온다.
- 제출해야 하는 url도 action 어트리뷰트로부터 가져온다.
- 내용이 존재하고 valid하다면 일단 textarea를 비워준다.
- ajax로 해당 내용을 url로 쏴준다. View에서 `content`를 받기를 기대하므로 key를 'content'로 한다.
- 댓글 등록에 성공할 경우 현재 textarea의 부모 중 `.comment-creator-div`인 div와 동일 위치에 있는 div 중 `.comment-container-div`를 찾는다.
- `.comment-container-div`의 내용물을 바꿔준다.
- 댓글 등록에 실패할 경우 실패했다고 alert을 띄우고 디버깅이 가능하도록 console창에 에러로그를 띄워준다. (원래 production 상황에서는 이러면 좋지않다)

여기까지 하고 새로고침해보자.


{% include image.html path="documentation/insta-clone-pt5-27.gif" path-detail="documentation/insta-clone-pt5-27.gif" alt="image" %}


아주 만족스럽게 잘 되고 있다. 이제 댓글 스타일링만 끝내자.


## 4. 댓글리스트 스타일링 하기

다른건 다 좋은데 삭제/수정 버튼이 한 줄씩을 차지하고 있고 댓글간 간격도 넓다. 상용 서비스에서 본문과 댓글은 글씨크기 차이가 없어보이지만 그래도 댓글 글씨크기는 약간 줄이는게 좋을거 같다.

```css
...전략

.comment-container-div {
    padding-top: 10px;
}

.comment-container-div p {
    height: fit-content;
    margin-bottom: auto;
    font-size: 15px;
}

.comment-button-container form {
    display: inline-block;
}

.comment-button-container form button {
    padding: 0;
    font-size: smaller;
}
```
- comment-container-div는 위의 본문과 약간 떨어트려준다. 이는 말풍선의 padding과 동일한 수치다.
- 댓글 본문은 `<p>`태그에 담겨있는데, 댓글이 길어지면 높이가 높아져야 한다. 그리고 기본값으로 아래쪽 마진이 있는데 auto로 세팅해주자.
- 폰트 크기는 15px정도면 좋은 것 같다.
- 버튼들이 줄이나눠지는 이유는 버튼을 담고 있는 form들의 display가 block이어서다. 이를 `inline-block`으로 바꾼다.
- 버튼의 글씨는 작을 수록 좋을 것이고 패딩을 없애서 깔끔하게 한다.


{% include image.html path="documentation/insta-clone-pt5-28.png" path-detail="documentation/insta-clone-pt5-28.png" alt="image" %}

보다시피 아주 깔끔하게 잘 됐고 긴 댓글도 무리없이 커버가능한 것을 볼 수 있다.


## 5. 댓글 수정 template 만들기
삭제는 무리없이 되는데 아마 수정이 안될 것이다. template을 만들지 않았기 때문이다.

`comments/templates/comments/comment_update_form.py`를 새로 생성하고 다음을 입력한다. `CommentUpdateView`에서 `template_name_suffix`를 '_update_form'으로 했음을 기억한다.


```django
{% raw %}
{% extends "index.html" %}
{% load crispy_forms_tags %}

{% block content %}
    <form method="post">
        {% csrf_token %}
        {{ form | crispy }}
        <input type="submit" class="btn btn-primary" value="수정"/>
    </form>
{% endblock %}

{% endraw %}
```

이제 수정도 잘 될 것이다.

## 6. 정리
- AJAX 없이 댓글시스템 구현이 가능하다. 이 경우 댓글 작성 후 화면위치 초기화, 새로고침으로 인한 비직관적인 사용자 UI의 문제가 있다.
- AJAX를 통해 통신하기 위해서는 csrf token에 대한 추가 처리를 반드시 해줘야 한다. 댓글 말고도 많은 곳에서 AJAX를 사용한다면 해당 메소드들을 따로 추출하여 전역 js에 포함시키는 것이 좋을 것이다.
- AJAX를 위해서는 slim Jquery를 사용하면 안된다.
- 통신이 성공했다면 반환된 html로 기존 html을 동적으로 교체한다.

이제 우리는 fully functional한 인스타그램을 얻었다.

이제 심화 기능을 추가하고 최적화를 시작하자.

