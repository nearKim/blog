---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.1"
description: "아키텍쳐 설계 및 Django model 구현"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기

본 튜토리얼에서는 Django를 이용하여 Instagram을 60%정도 재현해본다. 단순히 이미지를 업로드하고 보여주는 것에서 끝내지 않고, AJAX를 이용한 Comment 모델, 여러 사진 업로드 지원, 한 포스팅에 여러 사진을 Carousel로 표현하는 법까지 다룬다. 즉, 본 튜토리얼 전체에서 다룰 내용은 다음과 같다.

1. 아키텍쳐 설계 및 Django model 구현 **(current)**
2. Django view 구현 (CBV)
3. View develop하기
4. Django template 기본 구현
5. Django template 완전 구현
    1. 상용 서비스와 같은 디자인 골격 잡기
    2. 수정/삭제 Dropdown 및 댓글창 예쁘게 만들기
    3. 기타 짜잘한 디자인 완성시키기
    4. AJAX를 이용한 기본적인 댓글 시스템 구현
6. 좀더 멋있는 기능 추가하기
    1. Infinite scroll 구현하기
    2. 사진 Carousel 붙이기

#### 사용할 django 버젼: 2.1

### Prerequisites:
1. Django에 대한 기본적 이해가 있는 것으로 가정한다. 예컨대 model이나 view가 무엇인지, ForeignKey가 무엇인지 등은 알고 있다고 전제한다.
2. javascript, jquery도 기본적 이해가 있는 것으로 가정한다. 예컨대 getElementById나 $('#test')가 무엇인지 알고 있다고 전제한다.
3. startapp이나 startproject 등과 같은 기본적인 django 명령어는 설명에서 생략한다.

#### Note
본 튜토리얼은 **프로덕션 가능한 범위 내에서 가장 단순한** 설계를 지향한다. 예컨대 model을 설계할 때, 기초적인 디자인 패턴은 지킬 것이지만 깊은 단계의 최적화는 하지 않는다. 또는 accounts라는 custom user를 다루는 앱을 만들 수도 있지만 만들지 않는다. 그리고 import문은 특별한 경우(예컨대 3rd party라이브러리 사용할 경우)를 제외하고 포함하지 않는다. 이 규칙은 app이나 template 등을 설계할 때도 지켜질 것이다.


<hr>

## 0.아키텍쳐 설계
상기 내용 정리를 보면 크게 2개의 app과 3개의 model이 필요할 것을 알 수 있다. 먼저 Instagram을 다룰 app과 Comment를 다룰 app이 필요할 것이다. 만일 필요에 따라 custom user를 정의해야 한다면 accounts를 하나더 만드는 것이 필요할 수도 있다. 하나의 Instagram 오브젝트는 여러장의 Photo를 담을 수 있으므로, Photo는 Instagram 모델을 FK로 가질 것이다. 마찬가지로, 하나의 Instagram 오브젝트는 여러개의 Comment를 가질 것이므로 Comment역시 Instagram을 FK로 가질 것이다.

> Instagram 앱 -> Instagram / InstagramPhoto

> Comment 앱  -> Comment

## 1. Model 설계

#### Coding convention

본격적인 Django코드를 짜기 전에 코딩 컨벤션을 정한다. 본 튜토리얼에서는 다음의 규칙을 따를 것이다.

1. app명: snake_case / 복수(plural)
2. model명: PascalCase / 단수(singular)
3. method명: snake_case
4. related_name: 복수(plural)

html이나 js 관련 컨벤션은 해당 문서에서 작성한다.

#### Instagram /  InstagramPhoto
```python
# <instagrams> app

# 인스타그램 사진을 업로드하면 저장할 경로를 지정한다
def get_instagram_photo_path(instance, filename):
    return 'media/feed/{:%Y/%m/%d}/{}'.format(now(), filename)


class Instagram(models.Model):
	# 작성자와 내용을 저장한다(반복)
    content = models.TextField(max_length=100, verbose_name='내용')
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, verbose_name='업로더')

    # 생성일시, 수정일시를 저장한다(반복)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)


class InstagramPhoto(models.Model):
    photo = models.ImageField(upload_to=get_instagram_photo_path, verbose_name='사진')
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='photos',
                                  verbose_name='인스타그램')
    # 생성일시, 수정일시를 저장한다(반복)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
```

#### Comment
```python
class Comment(models.Model):
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='comments',
                                  verbose_name='인스타그램')
    # 작성자와 내용을 저장한다(반복)
    author = models.ForeignKey(settings.AUTH_USER_MODEL, verbose_name='작성자')
    content = models.TextField(max_length=100, verbose_name='내용')

    # 생성일시, 수정일시를 저장한다(반복)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

```

### 1차 refactor

위 모델들에서 많은 필드들이 겹치는 것을 발견할 수 있다. 모든 필드가 `created`, `updated` 필드를 가지고 있으며 `content`가 있는 모델들은 반드시 `author`가 존재한다. 이때 내가 자주 쓰는 패턴은 각각을 캡슐화하는 `Mixin`을 정의하는 것이다. 이 `Mixin`들은 글로벌하게 사용될 것이므로, 기존 프로젝트 폴더에 새로운 파일 `mixins.py`를 만들어서 넣어준다. 프로젝트 디렉토리 구조는 다음과 같을 것이다.(Instagram_clone은 본 튜토리얼의 프로젝트 이름이다)

```bash
# django가 기본생성하는 파일들은 표시하지 않는다.
.
├── Instagram_clone
│   └── mixins.py
├── comments
│   └── migrations
├── instagrams
│   └── migrations
```

```python
# <mixins.py>

class TimeStampedMixin(models.Model):
    # 생성일시, 수정일시를 저장한다
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    # 주의!
    class Meta:
        abstract=True


class PostableMixin(models.Model):
    # 작성자와 내용을 저장한다
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, verbose_name='작성자')
    content = models.TextField(max_length=100, verbose_name='내용')

    # 주의!
    class Meta:
        abstract=True

```

자 이제 각 instagrams 및 comments 안에 있는 모델들에서는 위 mixin들을 상속받으면 된다. 여기서 주의할 것은 반드시 mixin들에는 `abstract=True`속성을 설정해줘야 한다. 그렇지 않고 기존 모델이 위 mixin들을 상속받을 경우, django는 이를 [multi-table inheritance](https://docs.djangoproject.com/ko/2.1/topics/db/models/#multi-table-inheritance)로 이해하여 각 mixin에도 데이터베이스 테이블을 할당할 것이다. 우리는 단순히 각 테이블에 `created`, `updated` 필드를 삽입하고 싶을 뿐이므로, 추상클래스로 mixin을 정의해야 한다.

> mixin들은 반드시 추상클래스로 정의한다.

이제 기존 모델들은 다음과 같이 축약된다.

#### Instagram / InstagramPhoto
```python
# <instagrams> app
class Instagram(TimeStampedMixin, PostableMixin):
    pass


class InstagramPhoto(TimeStampedMixin):
    photo = models.ImageField(upload_to=get_instagram_photo_path, verbose_name='사진')
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='photos',
                                  verbose_name='인스타그램')
```


#### Comment
```python
# <comments> app
class Comment(TimeStampedMixin, PostableMixin):
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='comments',
                                  verbose_name='인스타그램')
```

### 2차 refactor
생각해보면 어차피 `created`, `updated`는 모든 테이블에 삽입될 것이다. 그렇다면 `author`, `content`가 존재하는 테이블에는 무조건 두 필드가 들어갈 것이란 얘기 아닌가? 그렇다면 `author`, `content`, `created`, `updated`를 하나로 묶을 수 있는 방법이 있다면 좋을 것이다. `PostableMixin`이 `TimestamptedMixin`을 상속하면 자동으로 우리가 원하는 시나리오가 얻어진다. 다음을 보자.

```python
# mixins.py
class TimeStampedMixin(models.Model):
    # 생성일시, 수정일시를 저장한다
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        abstract=True


class Postable(TimeStampedMixin):
    # 자동으로 생성일시, 수정일시 필드가 추가된다
    # 작성자와 내용을 저장한다
    author = models.ForeignKey(settings.AUTH_USER_MODEL, verbose_name='작성자')
    content = models.TextField(max_length=100, verbose_name='내용')

    class Meta:
        abstract=True
```

그리고 원래의 모델은 다음과 같이 변경된다.

#### Instagram / InstagramPhoto
```python
# <instagrams> app
class Instagram(Postable):
    pass

class InstagramPhoto(TimeStampedMixin):
    photo = models.ImageField(upload_to=get_instagram_photo_path, verbose_name='사진')
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='photos',
                                  verbose_name='인스타그램')

```

#### Comment
```python
# <comments> app
class Comment(Postable):
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='comments',
                                  verbose_name='인스타그램')
```

굉장히 깔끔해졌다.

만일 여기에 `likes` 필드를 추가해야된다고 가정해보자. 현재 instagram은 댓글과 instagram게시글 각각에 대해 likes를 허용하고 있다. `Postable`클래스에 likes필드를 추가하면 한줄 코드 추가로 모든 일이 편해질 것이다. 만일 처음 설계였다면 일일히 likes를 구현할 모델에 해당 필드를 구현했어야 할 것이다.

이로써 기능 추가 및 삭제에 유연한 scalable한 model 설계를 얻었다.

이제 **view** 설계로 넘어가보자.
