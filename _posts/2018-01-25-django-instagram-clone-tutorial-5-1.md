---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.5 (1)"
description: "Django template 완전 구현"
tags: [tutorial, django, python]
---

# Django를 이용한 Instagram 클론 만들기 Part.5 (1)


1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV)
3. View develop하기
4. Django template 기본 구현
5. Django template 완전 구현
    1. 상용 서비스와 같은 디자인 골격 잡기 **(current)**
    2. 수정/삭제 Dropdown 및 댓글창 예쁘게 만들기
    3. 기타 짜잘한 디자인 완성시키기
    4. AJAX를 이용한 기본적인 댓글 시스템 구현
6. 좀더 멋있는 기능 추가하기
    1. Infinite scroll 구현하기
    2. 사진 Carousel 붙이기


<hr>

## 1. 외부를 멋있게 만들기
댓글을 제외하고 기본적 구현은 얼추 끝났지만 디자인이 구리다. 이번에는 디자인을 예쁘게 만들어보자. 우리는 Instagram을 clone하고 있으므로 Instagram의 스타일을 많이 적용할 것이다.
여러 사진을 올려봤다면 지금 현재의 화면은 다음과 비슷할 것이다.

{% include image.html path="documentation/insta-clone-pt5-1.png" path-detail="documentation/insta-clone-pt5-1.png" alt="image" %}

Critical한 문제가 바로 눈에 보인다. ** 큰 사진의 경우 resizing** 이 필요하다. 스마트폰의 카메라나 DSLR로 찍은 사진을 Instagram에 업로드하게 되는데, 일반적으로 이런 사진들은 고해상도 사진이다. 초고화질 사진들을 웹상에서 그대로 내보내게 되면 다음과 같은 문제들이 발생한다.

- 브라우저상에서 표시할 때 적절히 처리하지 않으면 크기가 너무 커진다.
- JPEG로 압축한 사진들과 비교하여 사진 화질의 차이를 생각보다 체감하기 어렵다.
- 모바일 사용자의 경우 엄청난 데이터를 잡아먹고 로딩시간도 오래걸린다.
- 서비스가 활성화될 경우 Static파일 저장 공간을 지나치게 많이 잡아먹는다.(100명이 한장에 4mb정도되는 사진을 하루에 5장씩 매일 업로드한다고 생각해보라)

따라서 우리는 이런 문제를 해결하기 위해 View단에서 `InstagramPhoto`를 저장할 때 resizing을 해줄 것이다. 그 이후에 css나 js를 적용하여 추가 기능들을 구현할 것이다.

## 2. 이미지 리사이징

모든 `InstagramPhoto`객체에 대해 우리는 resizing을 적용하길 원한다. 따라서 `InstagramPhoto`모델에서 `save()`메소드를 오버라이드하여 처리한다.

다음 코드를 `instagrams/models.py`에 추가한다.


#### instagrams/models.py
```python
from PIL import Image, ExifTags
from io import BytesIO

def rotate_and_resize(image):
    # 저장시 가로 최고 길이는 614px이다.
    base = 614
    image = Image.open(image)
    img_format = image.format

    # exif 태그를 검색하여 사진의 회전여부를 판단한다.
    try:
        for orientation in ExifTags.TAGS.keys():
            if ExifTags.TAGS[orientation] == 'Orientation':
                break
        exif = dict(image._getexif().items())

        if exif[orientation] == 3:
            image = image.rotate(180, expand=True)
        elif exif[orientation] == 6:
            image = image.rotate(270, expand=True)
        elif exif[orientation] == 8:
            image = image.rotate(90, expand=True)
    except (AttributeError, KeyError, IndexError):
        # 사진에 exif 태그가 없으면 그대로 진행한다.
        pass
    # 회전 컨트롤이 완료된 이미지의 크기를 얻는다.
    (width, height) = image.size

    # 사진이 작든 크든 너비를 614px로 맞추기 위한 비율을 구한다.
    factor = base / width

    # 너비와 높이를 해당 factor를 곱하여 계산한다.
    image = image.resize((int(width * factor), int(height * factor)), Image.ANTIALIAS)
    img_io = BytesIO()
    image.save(img_io, img_format, quality=60)
    return img_io

```
- 정방향이 아닌 돌려서 찍은 사진의 경우 EXIF 태그에 해당 Orientation 정보가 저장된다.
- 브라우저는 이를 읽어서 보정한 후 사진을 뿌려주지 않으므로 웹 브라우저에서는 이미지가 회전된 상태로 보이게 된다.
- 이에 따라 Orientation정보를 통해 수동으로 정방향으로 회전시켜 줘야 한다.
- 너비를 614px으로 맞추고 높이는 비율에 맞춰서 리사이징 한다. (만일 둘다 614보다 작다면 확대된다)
- JPEG 퀄리티 60으로 압축하여 저장한다.


이제 이 코드를 `InstagramPhoto`의 `save()`메소드에 넣어 준다.

```python
class InstagramPhoto(TimeStampedMixin):
    photo = models.ImageField(upload_to=get_instagram_photo_path, verbose_name='사진')
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='photos',
                                  verbose_name='인스타그램')

    def save(self, *args, **kwargs):
        photo_io = rotate_and_resize(self.photo)
        self.photo.save(self.photo.name, ContentFile(photo_io.getvalue()), save=False)
        # 직접 인스턴스의 save를 불러서 파일을 저장한다.
        super(InstagramPhoto, self).save(*args, **kwargs)
```
- `PIL`의 save 파일을 로컬저장소에 저장하고 원격저장소에 저장하지 않는다.
- 따라서 직접 `super`를 불러서 직접 인스턴스를 저장한다.

#### 정리

admin사이트에서 기존의 데이터를 싹 밀어버린 후 다시 사진들을 저장해보자.


{% include image.html path="documentation/insta-clone-pt5-2.png" path-detail="documentation/insta-clone-pt5-2.png" alt="image" %}

리사이징이 잘 된 것을 알 수 있다. 이제 css 스타일링에 들어가자.

## 3. CSS 스타일링
현재 우리는 2개의 template을 가지고 있다. 각 template별로 따로 적용되는 css를 만들어야 한다. 이를 위해서 `index.html`에 새로운 `style-head` block을 만들자.

또 전역에서 사용될 css파일을 만들어서 넣어줘야 한다. 일단 template을 아래와 같이 수정한다.

```django
{%raw%}
<!-- index.html -->
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Instagram Clone</title>

    <!-- Bootstrap4-alpha -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.6/css/bootstrap.min.css" integrity="sha384-rwoIResjU2yc3z8GV/NPeZWAv56rSmLldC3R/AZzGRnGxQQKnKkoFVhFQhNUwEyJ" crossorigin="anonymous">
    <script src="https://code.jquery.com/jquery-3.1.1.slim.min.js" integrity="sha384-A7FZj7v+d/sdmMqp/nOQwliLvUsJfDHW+k9Omg/a/EheAdgtzNs3hpfag6Ed950n" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tether/1.4.0/js/tether.min.js" integrity="sha384-DztdAPBWPRXSA/3eYEEUWrWCy7G5KFbe8fFjk5JAIxUYHKkDx6Qin1DkWx51bBrb" crossorigin="anonymous"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.6/js/bootstrap.min.js" integrity="sha384-vBWWzlZJ8ea9aCX4pEW3rVHjgjt7zpkNpZk+02D9phzyeVkE+jo0ieGizqPLForn" crossorigin="anonymous"></script>

    {# ADDED! #}
    <!-- Custom css-->
    <link rel="stylesheet" type="text/css" href="{% static 'css/index_style.css' %}">
    {% block style-head %}
    {% endblock %}

</head>
...이하 동일
{%endraw}
```

이제 `feed_list.html`이나 `instagram_form.html` 등 `index.html`을 상속받는 모든 template에서 `style-head` block을 상속받고 그 안에 본인들이 사용할 개별 stylesheet link를 넣을 것이다.

이 때 많이 쓰이는 패턴은 전역으로 사용되는 base css(혹은 SCSS) 파일을 root레벨 static폴더에 (현재는 Instagram_clone/static/)에 만들고 개별 template에서 사용할 css파일은 각 app별로 따로 만드는 것이다.

> 일반적으로 전역 css파일은 root 레벨의 static폴더에 넣고 `index.html`에 포함한다. 개별 css파일은 app 레벨의 static폴더에 넣고 `index.html`에서 정의한 빈 block을 오버라이드 하여 겹치지 않게 사용한다.

css 파일 네이밍 컨벤션은 `<적용할 html 파일 이름>_style.css`로 하기로 하자.


### 3.1. CSS 파일 로딩

CSS파일은 static 파일이므로 이를 로딩하기 위해서는 기본적으로 다음 3가지 과정을 거쳐야 한다.

1. 먼저 전역 css파일인 `Instagram_clone/static/css/index_style.css`를 새로 생성한다.
2. 개별 css파일인  `instagrams/static/instagrams/feed_list_style.css`를 새로 생성한다.
3. `Instagram_clone/urls.py`에 `urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)`를 추가한다.
4. `Instagram_clone/settings.py`에 `STATIC_URL`과 `STATICFILES_DIRS`를 세팅한다.
5. `instagrams/templates/instagrams/feed_list.html`에 stylesheet를 추가한다.

```python
# settings.py

STATIC_URL = '/static/'
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'Instagram_clone', 'static'),
]

# urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('instagrams/', include('instagrams.urls', namespace='instagrams')),
    path('comments/', include('comments.urls', namespace='comments')),
    path('accounts/', include('django.contrib.auth.urls')),
]

if settings.DEBUG is True:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
    urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)

```

```django

{%raw%}
<!-- feed_list.html -->
{% extends "index.html" %}
{% load static %}
{% load crispy_forms_tags %}

{% block style-head %}
    <link rel="stylesheet" type="text/css" href="{% static 'instagrams/css/feed_list_style.css' %}">
{% endblock %}

{% block content %}
...이하 동일
{%endraw%}
```

크롬 개발자도구를 연 상태로 `localhost:8000/instagrams/`로 접근해보자. networks 탭에 `index_style.css`, `feed_list_style.css`가 200 OK로 잘 로딩되었다면 성공한 것이다.


{% include image.html path="documentation/insta-clone-pt5-3.png" path-detail="documentation/insta-clone-pt5-3.png" alt="image" %}

만일 제대로 로딩되지 않았다면 (404 Not found 등) stylesheet등이 제대로 포함되어있는지 살펴보자.


### 3.2. CSS 파일 적용
#### 3.2.1. feed_list_style.css
지금까지 작성한 템플릿의 개략적인 디자인은 다음 그림과 같다.


{% include image.html path="documentation/insta-clone-pt5-4.png" path-detail="documentation/insta-clone-pt5-4.png" alt="image" %}

현재는 디자인이 참 못생겼다. 이 상황을 타개해보자.


#### 상용 Instagram 디자인 분석
현재 상용 Instagram 서비스는 어떻게 css 스타일을 적용하고 있는지 연구해보자. 이 부분은 스킵해도 무방하다.

다음 그림은 인스타그램 Feed를 개발자도구를 사용하여 오픈한 모습이다. 개략적인 html구조는 다음과 같다.

```html
<section class="_9eogI E3X2T">
    <main class="SCxLW  o64aR">
        <section class="_1SP8R j9XKR ">
            <div class="cGcGK">
                <div>
                    <article>

...
```


{{site.baseurl}}
{% include image.html path="documentation/insta-clone-pt5-7.png" path-detail="documentation/insta-clone-pt5-7.png" alt="image" %}

첫번째 section과 main에 집중하여 분석해보자. 개발자도구에서 각각을 클릭하여 우측에 나타나는 css를 분석한다.

- 모든 컴포넌트들은 `section` 안에 담겨 있다.
- `main` 컴포넌트와 `nav` 컴포넌트가 그 안에 존재한다.
- `nav`는 상단 내비게이션바와 검색창을 담고 있다.
- `main`은 나머지 모든 컴포넌트들을 포함한다.
- `section`은 `min-heigth:100%;`인 것으로 보아 전체의 기준이 된다.
- `section`은 `flexbox`속성을 지정하여 responsive하게 표현가능하다. `flex-direction`은 당연히 column이 될 것아고 `align-items`는 stretch로 지정한다.
- `section`에서 테두리를 포함해서 너비 등을 계산하는 것이 편할 것 같다. `box-sizing`은 border-box로 설정한다.
- `main`은 `background-clor`가 `#fafafa`로 지정되어 있다.


이제 `main`을 클릭하여 열어보면 또다른 `section`과 `div`뭉치가 나온다.

{% include image.html path="documentation/insta-clone-pt5-8.png" path-detail="documentation/insta-clone-pt5-8.png" alt="image" %}

- 두번째 `section`은 `max-width`가 935px로 잡혀있다. `position`은 relative로 잡는다.
- 두번째 `section`은 여러개의 `div`를 포함한다. 각 `div`는 좌측 feed들의 컨테이너와 우측 story 및 suggestion 컨테이너이다.
- 본 튜토리얼에서는 우측 story 및 suggestion 컨테이너는 없으므로 굳이 section 안에 여러개의 div를 넣을 필요 없이 통합하면 된다.
- `class`가 **cGcGK**인 `div`가 feed 컨테이너이다.
- 해당 `div`는  `float:left`, `margin-right: 28px`, `max-width:614px`, `width:100%`로 잡혀있다.
- 이 중 float과 margin-right 속성은 우측 story 및 suggestion 컨테이너와의 관계때문에 설정된 스타일이므로 무시한다.
- 종합하면 Instagram의 설계자들은 모든 좌,우측 컨테이너를 담을 큰 컨테이너의 너비를 935px로 잡고 그 내부의 feed 컨테이너는 614px로 잡은 것으로 보인다.
- 나머지 남는 공간은 margin으로 처리될 것이다.


이제 `class`가 **cGcGK**인 `div`를 열어보면 다음과 같다.


{% include image.html path="documentation/insta-clone-pt5-5.png" path-detail="documentation/insta-clone-pt5-5.png" alt="image" %}

- 각 Instagram 포스팅은 `article` 태그 안에 정보가 담겨있다.
- `article` 상단부에는 margin이 없고 `margin-bottom`은 60px로 잡혀있다.
- `border-radius` 는 3px, `border` 자체는 1px solid, #e6e6e6 색으로 지정되어 있다.
- `background-color` 는 #fff 이다.
- `margin-left`, `margin-right`은 -1px로 설정되어 있다. 이는 border가 1px이므로 사진이 border를 가리게 하기 위한 것으로 보인다.


이제 지금까지 얻어낸 정보를 우리의 Instagram_clone에 적용해보자.


#### Instagram_clone 적용

상용 서비스에서는 feed 컨테이너 말고도 여러 컴포넌트가 존재하기 때문에 복수의 section 및 div가 필요하다. 하지만 우리의 구조에서는 컴포넌트 디자인을 통합가능하다.

현재 template에서 feed 컨테이너는 `<div class="instagram-container">`이고 각 feed 포스팅은 `<div class="card">`가 담당한다. 이 태그들에 위 스타일을 어떻게 녹여내야 할까?

1. `<body>` 태그
    이 태그는 상용 서비스의 첫번째 `<section>`과 같다. 우리는 네비게이션바가 없으므로 상용 서비스의 `<main>`은 의미가 없다. 따라서 첫번째 section, main을 통합하여 넣어준다.

```css
/* Root Container */
body {
    min-height: 100%;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    box-sizing: border-box;
    background-color: #fafafa;
    margin: 0;
    padding: 0;
    position: relative;
}

```

> 여기서 주의할 점은, 만일 다른 페이지들이 존재하여 위와 다른 `<body>`속성을 지정해야 한다면 이렇게 해선 안된다는 것이다. 위 css는 `index.html`에 적용되고 모든 template들은 이를 상속받으므로 위 스타일은 앞으로 작성할 모든 `<body>`태그에 적용된다. 현 예제에서는 사용할 페이지가 얼마없으므로 큰 상관이 없지만, 큰 서비스를 계획한다면 위 스타일은 개별 css로 추출되어야 햔다.


2. `instagram-container` div
이 태그는 상용 서비스의  `<div class="cGcGK">`와 같다. 상용서비스와 달리 우측 Story 컴포넌트가 필요없으므로 두번째 section과 div를 모두 통합하여 넣어준다.

```css
/* Main instagram feed container */
.instagram-container {
    margin: 0 auto;
    padding-top: 60px;
    flex-direction: column;
    position: relative;
    display: flex;
    max-width: 614px;
    width: 100%;
    flex-grow: 1;
    align-items: stretch;
    box-sizing: border-box;
}

```

- 원래 max-width가 컨테이는 935px, article들이 들어있는 div는 614로 잡혀있었다.
- 상기한대로 네비게이션 및 우측 컴포넌트가 필요없으므로 굳이 위와 같이 두번 max-width를 지정할 필요는 없어보인다.
- max-width를 기본인 614px로 지정하자.


3. `card` div
`feed_list.html`에서 상용 서비스의 `article` 태그 역할을 해주는 것은 `<div class="card">`이다. 따라서 이부분에 상용서비스와 같은 스타일을 적용해준다.

```css
/* Main instagram feed container */
.card {
    margin-bottom: 60px;
    padding: 0;
    border-radius: 3px;
    border: 1px solid #e6e6e6;
    background-color: #fff;
    margin-left: -1px;
    margin-right: -1px;
}

```

여기까지 하고 저장한 후 브라우저에서 새로고침 해보자.

{% include image.html path="documentation/insta-clone-pt5-9.png" path-detail="documentation/insta-clone-pt5-9.png" alt="image" %}


아까보다 훨씬 깔끔해졌다. 무엇보다 배경색과 테두리까 예쁘게 들어가니 훨씬 가독성이 좋아지고 상용서비스와 비슷해졌다.


## 4. 정리

- Instagram에 올라간 사진은 상용서비스와 같이 너비를 614px로 맞춘다.
- 이 때 모든 업로드된 InstagramPhoto에 대해 적용되어야 하므로 해당 모델의 `save()`메소드를 오버라이드하여 구현한다.
- 물론 상용서비스에서는 더욱 정밀한 리사이징이 가능하지만 가장 기본적인 것만 일단 적용한다.
- css파일을 적용할 때 전역으로 적용할 파일과 개별 template에 적용할 파일을 따로 구분하여 저장한다.
- 상용 Instagram 서비스를 개발자도구를 이용하여 스타일 분석한 후 적용한다.




