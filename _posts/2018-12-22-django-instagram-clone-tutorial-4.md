---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.4"
description: "Django template 기본 구현"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기 Part.4


1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV)
3. View develop하기
4. Django template 기본 구현 **(current)**
5. Django template 완전 구현
    1. 상용 서비스와 같은 디자인 골격 잡기
    2. 수정/삭제 Dropdown 및 댓글창 예쁘게 만들기
    3. 기타 짜잘한 디자인 완성시키기
    4. AJAX를 이용한 기본적인 댓글 시스템 구현
6. 좀더 멋있는 기능 추가하기
    1. Infinite scroll 구현하기
    2. 사진 Carousel 붙이기

<hr>



## 1. 기본 골격 정의

Django template 엔진은 강력한 상속관계를 지원한다. 많은 경우 `header`나 `footer` 등을 가장 상위 template에 정의하고 `<body>`태그부분만 바꿔서 웹페이지를 구성한다. 우리도 이 기본 패턴을 따를 것이다.

### 1.1. index.html 추가


#### Template 기본경로 변경

아무것도 건드리지 않았다면 `settings.py`의 TEMPLATES는 다음과 같이 설정되어 있을 것이다.

```python
# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')]
        ,
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

- 기본경로는 `DIRS`로 `Instagram_clone/templates`이다. `Instagram_clone/Instagram_clone/templates`가 아님에 유의한다.
- `APP_DIRS`가 True이므로 각 앱별로 `templates`폴더를 검색하여 템플릿을 찾을 것이다.

위와 같은 구성에서는 다음과 같은 디렉토리 구조를 가져야 한다.

```bash
# templates폴더가 같은 계층에 위치한다.
.
├── Instagram_clone
├── comments
├── instagrams
├── myenv
└── templates
```

전체 프로젝트에서 공통적으로 사용되는 모든 소스코드는 `Instagram_clone`안에 추가적으로 만들어서 관리하고 있다.

따라서 공통 템플릿들도 `Instagram_clone/templates`안에 위치시키는게 좋을 것 같다.

디렉토리 구조를 다음과 같이 변경한다.

```bash
# templates폴더가 Instagram_clone 내부로 들어갔다.
.
├── Instagram_clone
│   ├── static
│   └── templates
├── comments
│
├── instagrams
│
```

이에 맞춰서 `settings.py`의 설정을 바꿔준다.

```python
# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR,'Instagram_clone', 'templates')], # 변경됨
            # 이하 생략..
    },
]
```

이는 Django가 template file을 찾을 때 `<project_root>/Instagram_clone/template`이라는 폴더를 함께 찾도록 명시한다. 여기에 모든 하위 template에서 상속받을 기본 template을 정의할 것이다.

이 폴더에 `index.html`을 만들고 다음을 넣자.


```django
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Instagram Clone</title>
</head>
<body>
{%raw%}{% block content %}
    {# fill here #}
{% endblock %}{%endraw%}
</body>
</html>
```

- 이 파일을 상속받은 template들에서는 `{%raw%}{% block content %}{%endraw%}` 를 오버라이드할 것이다.
- `bootstrap`등은 나중에 추가한다.


### 1.2. Authentication template 추가

Part.3에서 Django가 제공하는 Auth view를 디폴트로 url pattern에 포함시켰다. 각 View들은 미리 지정된 이름의 html 파일을 참조하는데, 이는 개발자가 직접 만들어줘야 한다. 친절하게도 Django documentation은 Login template으로 바로 이용가능한 [예제 코드](https://docs.djangoproject.com/ko/2.1/topics/auth/default/#django.contrib.auth.views.LoginView)를 제공해주는데, 이를 약간만 변형시켜서 적용해보자.


#### login.html 작성

이제 `Instagram_clone/templates/registration/`경로에 `login.html`을 만들고 다음을 입력한다.


```django
{%raw%}{% extends "index.html" %}

{% block content %}

{% if form.errors %}
<p>아이디와 비밀번호가 일치하지 않습니다. 다시 시도해 주세요.</p>
{% endif %}

{% if next %}
    {% if user.is_authenticated %}
    <p>이 페이지를 볼 권한이 없습니다.</p>
    {% else %}
    <p>이 페이지를 보기 위해선 로그인 해주세요.</p>
    {% endif %}
{% endif %}

<form method="post" action="{% url 'login' %}">
{% csrf_token %}
<table>
<tr>
    <td>{{ form.username.label_tag }}</td>
    <td>{{ form.username }}</td>
</tr>
<tr>
    <td>{{ form.password.label_tag }}</td>
    <td>{{ form.password }}</td>
</tr>
</table>

<input type="submit" value="login">
<input type="hidden" name="next" value="{{ next }}">
</form>

{# url에 비밀번호 리셋 뷰를 포함하긴 했지만 구현하진 않는다. #}
{# <p><a href="{% url 'password_reset' %}">Lost password?</a></p> #}

{% endblock %}{%endraw%}
```

- `username`과 `password`를 받는 간단한 form을 정의한다.
- Django 제공 password reset 뷰를 포함하긴 했지만 구현하진 않는다.
- 소스코드를 보면 `LoginView`는 `template_name`을 `registration/login.html`로 명시하고 있다.
- 만일 디렉토리를 바꾸고 싶거나 템플릿명을 바꾸고 싶으면 path안에 따로 LoginView를 오버라이드 해주면서 명시해준다.
- ex) path('accounts/login', auth_views.LoginView.as_view(template_name='accounts/main.html'))


### 1.3. 각 app별 template file 추가

이제 각 app 별로 분리될 템플릿을 만들자. 먼저 생각하기 쉬운 `instagrams`앱부터 작성한다.
기본적으로 `settings`의 `TEMPLATES`에서 `APP_DIRS`가 True로 되어 있을 것이다. 아니라면 처리해준다.


```django
<!-- instagrams/templates/instagrams/feed_list -->
{%raw%}{% extends index.html %}

{% block content %}
    {# FIXME: for test only #}
    <h1>Feed List!</h1>
{% endblock %}{%endraw%}
```

- 우리는 이 html파일에서 `Instagram`들의 List를 볼 것을 원한다.
- `InstagramListView`가 이 template과 연결될 것이다.
- 해당 뷰에서 `template_name = 'instagram/feed_list.html'`라고 작성했음을 기억한다.


대충 기본골격이 끝난거 같다. 여기까지 하고 runserver를 해주자. superuser를 만들지 않았으면 자유롭게 생성한다.



## 2. 중간점검.

현재 상태에서 `http://localhost:8000/instagram`으로 접근해보자. 뷰가 제대로 작성이 되었다면 로그인을 하지 않았으므로 로그인 페이지로 보내야 한다.

{% include image.html path="documentation/insta-clone-pt4-1.png" path-detail="documentation/insta-clone-pt4-1.png" alt="image" %}

좋다. 이제 로그인을 해보자. superuser 계정의 아이디와 비번을 넣고 로그인을 해본다.

{% include image.html path="documentation/insta-clone-pt4-2.png" path-detail="documentation/insta-clone-pt4-2.png" alt="image" %}

뭔가 잘못됐다. 지금까지 완벽하게 한거 같은데 어디서 잘못된거지?

문제는 간단하다. 우리는 LoginView를 모두 구현했다고 생각했지만 **로그인한 이후에 어디로 가야하는지**는 지정해주지 않았다. 즉, `LoginView`의 `success_url`을 지정하지 않았다.

아... 이걸 지정하기 위해 굳이 `LoginView`를 url pattern에 추가하여 오버라이드를 해야할까? 매우 귀찮다.

그냥 `settings.py`에 `LOGIN_REDIRECT_URL`을 설정하고 그걸 `instagram`의 `feed-list`로 연결시켜주자.


> Django 디폴트 `LoginView`를 이용할 때 로그인 후 리다이렉트 경로를 지정하고 싶을 때는 [`LOGIN_REDIRECT_URL`](https://docs.djangoproject.com/ko/2.1/ref/settings/#std:setting-LOGIN_REDIRECT_URL)을 활용한다.


```python
# settings.py
LOGIN_REDIRECT_URL = '/instagram'
```

이렇게 세팅하고 다시 로그인을 시도해보자.

{% include image.html path="documentation/insta-clone-pt4-3.png" path-detail="documentation/insta-clone-pt4-3.png" alt="image" %}

`feed_list.html`이 잘 뜬 것을 알 수 있다. 만일 잘 안떴다면 코드상의 오타를 잘 살펴보자. 이제 절반정도(?) 왔다.



## 3. instagram_form.html 작성 & Django crispy forms

이제 Instagram을 업로드할 기본적인 template을 작성한다. 우리는 Django crispy forms를 이용할 것이다.

#### Django crispy forms
개인적으로 Django 개발을 할 때 3rd party 라이브러리는 최소한으로 유지한다. 내 힘으로 개발하는 것이 재밌기도 하고, 3rd party 라이브러리의 경우 다수의 사람들이 사용한 라이브러리가 아니면 상대적으로 안정성을 담보하기 힘들기 때문이다. 하지만 [Django crispy forms](https://django-crispy-forms.readthedocs.io/en/latest/)은 단언컨대 frontend form 작업을 위해서 반드시 필요한 라이브러리라고 할 수 있다.

적용하는 절차는 다음과 같다.

1. pip를 통해 install
2. `settings.py`에 `INSTALLED_APPS` 및 `CRISPY_TEMPLATE_PACK`지정
3. 사용할 template에 `Bootstrap` CDN 추가. (다운로드 후 추가도 가능)
    - 모든 form에서 쓸 예정이므로 상속받는 `index.html`에 넣는게 좋겠지.
4. 사용할 template 상단에 `{% raw %}{% load crispy_forms_tags %} {%endraw%}` 추가
5. `<form></form>`태그 안에서 `{%raw%}{{ form | crispy }}{%endraw%}`와 같은 형태로 사용


```python
# settings.py
INSTALLED_APPS = [
    # ...생략
    # Tutorial apps
    'instagrams',
    'comments',

    # 3rd party
    'crispy_forms', # 추가
]

CRISPY_TEMPLATE_PACK = 'bootstrap4'
```

- document에 나온대로 설치를 마치고 apps를 추가해준다.
- TEMPLATE_PACK은 `bootstrap4`로 한다. 현재 `bootstrap4`는 alpha만 지원한다.


이제 `Bootstrap4`를 렌더링하기 위해 스타일시트를 CDN을 이용하여 추가한다.


#### index.html
```django
{%raw%}
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Instagram Clone</title>

    <!-- Bootstrap4-alpha CDN -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.6/css/bootstrap.min.css" integrity="sha384-rwoIResjU2yc3z8GV/NPeZWAv56rSmLldC3R/AZzGRnGxQQKnKkoFVhFQhNUwEyJ" crossorigin="anonymous">
    <script src="https://code.jquery.com/jquery-3.1.1.slim.min.js" integrity="sha384-A7FZj7v+d/sdmMqp/nOQwliLvUsJfDHW+k9Omg/a/EheAdgtzNs3hpfag6Ed950n" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tether/1.4.0/js/tether.min.js" integrity="sha384-DztdAPBWPRXSA/3eYEEUWrWCy7G5KFbe8fFjk5JAIxUYHKkDx6Qin1DkWx51bBrb" crossorigin="anonymous"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.6/js/bootstrap.min.js" integrity="sha384-vBWWzlZJ8ea9aCX4pEW3rVHjgjt7zpkNpZk+02D9phzyeVkE+jo0ieGizqPLForn" crossorigin="anonymous"></script>

</head>
<body>
{% block content %}
    {# TODO: Fill here #}
{% endblock %}
</body>
</html>
{%endraw%}
```

- `<head>`에 bootstrap4-alpha 스타일시트를 추가해준다.


#### instagrams/instagram_form.html

```django
{%raw%}
{% extends "index.html" %}
{% load crispy_forms_tags %}

{% block content %}
    <form method="post" enctype="multipart/form-data">
        {% csrf_token %}
        {{ form | crispy }}
        <input type="submit" class="btn btn-primary" value="올리기"/>
    </form>
{% endblock %}
{%endraw%}
```

- image를 업로드 해야하므로 `enctype="multipart/form-data"`로 해야 함을 유의한다
- 넘나 간단한 것


이제 `http://localhost:8000/instagram/create`로 들어가보자.

{% include image.html path="documentation/insta-clone-pt4-4.png" path-detail="documentation/insta-clone-pt4-4.png" alt="image" %}


잘 나왔다!

#### Troubleshooting
만일 잘 나오지 않은 경우, 두가지를 확인한다.
1. `MEDIA_URL`과 `MEDIAFILES_DIRS`가 제대로 설정되어 있는가?
2. root `url pattern`에 다음 코드가 제대로 추가되어 있는가?

```python
if settings.DEBUG is True:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```


이제 해당 폼에 적당한 내용과 이미지를 선택하고 submit을 해보자.


{% include image.html path="documentation/insta-clone-pt4-3.png" path-detail="documentation/insta-clone-pt4-3.png" alt="image" %}


다시 feed-list로 돌아왔다. 이는 `InstagramCreateView`에서 `success_url = reverse_lazy('instagrams:feed-list')`로 설정했기에 그렇다.
데이터가 잘 들어갔는지 admin을 통해서 확인해보자.

`https://localhost:8000/admin`



{% include image.html path="documentation/insta-clone-pt4-5.png" path-detail="documentation/insta-clone-pt4-5.png" alt="image" %}

{% include image.html path="documentation/insta-clone-pt4-6.png" path-detail="documentation/insta-clone-pt4-6.png" alt="image" %}

`Instagram`과 `InstagramPhoto`모두 잘 만들어졌음을 알 수 있다.

#### Troubleshooting
만일 admin에 Instagrams와 Instagram Photos가 나오지 않는다면, 각 app의 `admin.py`에 각 model이 제대로 register 되어 있는지 확인한다.


이제 기본적인 생성 template이 완성되었으므로 본격적으로 list template을 만들어보도록 하자.



## 4. feed_list.html 수정

처음 `feed_list.html`의 기본적인 골격을 잡았었다. 현재는 'Feed List!'라는 문구만 나오는 상황이다. 여기에 instagram 객체가 나오도록 처리해보자.

그전에 실제 Instagram 서비스가 feed를 어떻게 관리하고 있는지 보자.


{% include image.html path="documentation/insta-clone-pt4-7.png" path-detail="documentation/insta-clone-pt4-7.png" alt="image" %}


[출처](https://www.instagram.com/p/BrpjP0AH5oD/)


위 사진은 실제 instagram의 사진을 캡쳐해온 것이다. 보기좋게 7개의 row로 나뉘어 있는 것을 알 수 있다. '좋아요'의 경우 현재 구현에서는 제외된다.

template에서 사용할 변수들의 경우,

1. `object_list` : ListView는 기본적으로 `object_list`라는 변수에 model의 list를 담아 template에 던져준다.
2. `comment_form` : `get_context_data()`를 오버라이드 하여 `comment_form` 이라는 변수에 댓글 입력 form을 담아 던져준다.


이 두가지를 이용한다.

```django
{% raw %}
{% extends "index.html" %}
{% load crispy_forms_tags %}

{% block content %}
    {# 최상위 컨테이너  #}
    <div class="instagram-container">
        {% for instagram in object_list %}

            {# 각 instagram을 실질적으로 나타낼 div #}
            <div class="card">
                {# Bootstrap내장 container를 사용한다#}
                <div class="container">
                    {# 1. 첫번째 row: 작성자 정보를 나타내고 수정/삭제 드롭다운을 넣어준다. #}
                    <div class="row">
                        <div class="col-9">
                            <span>{{ instagram.author.username }}</span>
                        </div>
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
                    </div>
                    {# 2. 2번째 row: 업로드한 사진들이 들어간다. #}
                    {# TODO: Add Carousel #}
                    <div class="photo-container row">
                        {% for instagram_photo in instagram.photos.all %}
                            <div class="photo-carousel">
                                <img src="{{ instagram_photo.photo.url }}"/>
                            </div>
                        {% endfor %}
                    </div>
                    {# 3. 3번째 row: 말풍선 모양 버튼들이 들어간다. 추후 좋아요 등을 구현하면 여기에 넣는다.  #}
                    <div class="button-container row">
                        {# TODO: Add balloon icon and connect with comment add form #}
                        <button class="add-comment-balloon">TEST BUTTON</button>
                    </div>
                    {# 4. 4번째 row: 업로더 정보가 한번 더 나타나고 내용이 들어간다. #}
                    <div class="description-container row">
                        <div class="uploader col-3">{{ instagram.author.username }}</div>
                        <div class="content col-9">{{ instagram.content }}</div>
                    </div>
                    {# 5. 5번째 row: 현재 인스타그램에 달린 댓글들이 리스트로 표현되어 들어간다. #}
                    <div id="comment-container-div" class="row">
                        <div class="comments-list col-12">
                            {% for comment in instagram.comments.all %}
                                <li class="comment">
                                    <span>{{ comment.author }}</span>
                                    <span>{{ comment.content }}</span>
                                </li>
                            {% endfor %}
                        </div>
                    </div>
                    {# 6. 6번째 row: instagram 작성 시간이 현재시간과의 '시간차'로 나타난다. #}
                    {# TODO: Change created time representation #}
                    <div class="created-time row">
                        <div class="col-12">{{ instagram.created }}</div>
                    </div>
                    {# 7. 7번째 row: 댓글 입력 form이 들어간다#}
                    <div id="comment-creator-div" class="row">
                        <div class="comments-form col-12">
                            {# TODO: Fill in action. #}
                            <form action="#" method="post">{% csrf_token %}
                                {{ comment_form | crispy }}
                                <input class="input-comment-create" type="submit" class="btn btn-secondary"/>
                            </form>
                        </div>
                    </div>
                </div>
            </div>

        {% endfor %}
    </div>
{% endblock %}
{%endraw%}
```

- 차후에 css styling을 고려하여 적절한 className들을 삽입한다.
- 1. 첫번째 row: 작성자 정보를 넣어주는데 username을 일종의 nickname으로 간주하고 표시할 것이다. 그리고 옵션을 넣어줄 드롭다운도 구현한다.
    - 만일 사용자의 `profile_pic`을 넣어주고 싶으면 custom user model을 만들어야 한다.
    - 이 경우 `accounts` app을 만들고 Django 기본 `User`모델에 대한 `proxy model`을 만들어 구현한다.
    - 수정버튼의 경우 그냥 수정창으로 리다이렉트 시켜주면 된다.
    - 삭제버튼의 경우 Django는 기본적으로 `confirm_delete`뷰로 이동하여 삭제 여부를 확인하는데, 우리는 그냥 `confirm`창만 띄워주고 대답에 따라 삭제여부를 결정하도록 구현한다.
    - 삭제 후에는 우리가 현재 보고 있는 창으로 돌아와야 한다. 이를 위해 `CommentDelete`에 사용할 예정이었던 **hidden input** pattern을 그대로 적용한다. [Part.2 참조](https://nearkim.github.io/django-instagram-clone-tutorial-2/)
- 2. 두번째 row: 사용자가 업로드한 사진들이 들어간다. 여러장이 가능하므로 `for`문을 돌아주면서 사진을 뿌린다.
    - frontend에서 carousel로 표현하기로 하였으므로 고려해야 한다.
- 3. 세번째 row: 각종 버튼들이 들어가는 container이다. 삭제해도 상관없지만 차후 like 버튼 등의 구현을 대비하여 따로 row로 분리한다.
    - fontawesome 이미지로 처리하는 것이 가장 좋을 것이다.
- 4. 네번째 row: 업로더 username을 한번 더 보여주고 content도 보여준다. 특별한 것은 없다.
- 5. 다섯번째 row: 댓글들이 리스트 형식으로 뿌려진다. `queryset`에 join해온 comment들을 `for`문을 돌면서 적절히 뿌려준다.
- 6. 여섯번째 row: instagram 생성시간이 **현재 시간과의 차이** 형태로 나타난다. 관련 처리를 해야 한다.
    - ex) 7시간 전
- 7. 일곱번째 row: 댓글들 생성 form이 나타난다. 드롭다운이 있어서 수정, 삭제등을 할 수 있다.
    - 본 튜토리얼에서는 수정, 삭제등을 첫번째 row에서 처리한다.



## 5. 중간 점검 2

여기까지 잘 나오고 있는지 확인해보자. `http://localhost:8000/instagram`으로 접속해보자.

{% include image.html path="documentation/insta-clone-pt4-8.png" path-detail="documentation/insta-clone-pt4-8.png" alt="image" %}

사진과 내용간의 대비를 위해 검은 배경을 집어넣어 보았다. 여태껏 우리가 설계한 모든것들이 (비록 못생겼지만) 잘 나온다!

이제 이 템플릿을 디자인만 제외하고 완성시켜보자.



## 6. 정리

- Global하게 쓰이는 모든 소스코드를 한곳에 모아놓기 위해 `settings.py`의 `TEMPLATES.DIRS`를 수정하였다.
- 기본적인 template 상속관계를 정의하였다.
- Login 등은 DJango가 기본제공하는 view 및 template을 사용하였다.
- Login 이후 리다이렉션 처리는 `LOGIN_REDIRECT_URL`을 따로 정의하여 귀찮음을 덜었다.
- Create할 때의 form은 `crispy_forms` 라이브러리를 이용하여 구현하였다.
- 이를 위해 Bootstrap4 CDN을 이용하여 스타일시트를 적용하였다.
- 실제 Instagram 서비스를 최대한 따라하기 위해 7개의 row로 UI를 구분하여 설계하였다.
- 수정/삭제의 경우 `CommentDeleteView` 처리할 때 언급했던 패턴을 그대로 적용하였다.


