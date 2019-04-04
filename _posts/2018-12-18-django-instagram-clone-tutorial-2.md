---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.2"
description: "Django view 구현 (CBV)"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기 Part.2


1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV) **(current)**
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

<hr>

## 1. View 설계

이제 `view`를 설계해본다. 우리에게 필요한 기능들을 TODO list의 형태로 작성해보자.

여기서 frontend와 관련된 task들은 따로 추려내어 가장 마지막으로 돌린다. 우리는 view를 먼저 설계할 것이다.

최종적으로 분리된 task들은 다음과 같다.

### TODO list

#### View Checklist

- [ ] Instagram 객체를 불러와서 화면에 뿌려준다
    - [ ] 이 때 `Instagram` 객체에 연결된 `InstagramPhoto` 객체들을 모두 로드해서 뿌린다.
    - [ ] `Instagram` 객체에 연결된 Comment 객체들을 모두 로드해서 뿌린다.
        - [ ] 이때 뿌리는 형식은 list여야 한다.
    - [ ] `Instagram` 객체 하단에 Comment input form이 들어가야 한다.
    - [ ] `Instagram` 객체에는 업로더 정보, 업로드 날짜, 내용이 들어가야 한다.

- [ ] `Instagram` 객체 여러개를 불러와서 화면에 뿌려준다.
    - [ ] 오로지 작성자와 superuser만이 객체를 수정/삭제 할 수 있어야 한다.


#### Frontend Checklist

- [ ] 1. `InstagramPhoto` 객체들을 뿌리는 형식은 carousel이어야 한다.
- [ ] 2. 댓글 입력 input 버튼은 따로 없고 한줄로 input이 표시되어야 한다.
- [ ] 3. `Instagram` 객체에는 편집/삭제 버튼이 위치해야 한다.
- [ ] 4. 편집/삭제버튼은 우측 상단 또는 우측 하단의 숨겨진 dropdown안에 있어야 한다.
- [ ] 5. 댓글 입력창에서 엔터를 치면 댓글이 자동 업데이트 되어야 한다.
- [ ] 6. 어느정도 스크롤을 내렸으면 다음 인스타그램 객체들을 로드해야 한다.(무한스크롤)


## 2. View 구현(CBV)

### 2.0. 생각해보기
먼저 구체적인 뷰를 구현하기 전에 생각을 해보자. `Generic View`로 생각해봤을 때 다음 `View`들이 필요할 것으로 '예상'된다.

* Instagram CreateView
* Instagram DetailView
* Instagram UpdateView
* Instagram DeleteView
* Instagram ListView


* Comment CreateView
* Comment ListView
* Comment UpdateView
* Comment DeleteView

#### InstagramPhoto는?

`InstagramPhoto`의 경우 따로 `View`를 통한 CRUD를 하지 않고 `Instagram` 객체를 CRUD할때 같이 동반되게 처리한다. 물론 이런 방식에 단점이 존재하는데,

- 사진을 수정할 수 없다. 기존에 올려진 사진을 수정하려면 싹다 지우고 다시 새로운 사진을 업로드 해야 한다.
- 특정한 사진만 삭제할 수 없다. 마찬가지로 수정을 하면서 싹다 지우고 재업로드 해야 한다.

그럼에도 불구하고 댓글의 수정/삭제보다 업로드한 이미지의 수정/삭제는 그 빈도수가 적으므로, 구현의 편리성을 위해 일단 이렇게 구성한다.

#### Instagram DetailView?

페이스북에서 제공하는 Instagram 상용서비스는 마이페이지에 들어가 **내가 업로드한** 인스타그램 오브젝트를 클릭하면 이미지와 댓글, 내용창의 레이아웃이 변경되며 상세페이지를 보여준다. 하지만 초기 리스트 페이지에서는 클릭하면 함께 있는 사용자의 id만 오버레이 될 뿐 아무 반응도 없다. 따라서 `DetailView`의 경우 이번 튜토리얼에서는 제외하도록 한다. 절대 귀찮아서가 아니다.

#### Comment ListView?

인스타그램 객체들의 list는 처음에 받아와서 보여줘야되기에 `InstagramListView`가 있어야 함에는 의심의 여지가 없다. 그런데 과연 `CommentListView`도 필요할까? 각 인스타그램 객체들에 속한 `Comment`들의 list를 보여줘야 하므로 필요할 것 같다.

하지만 더 생각해보면 딱히 그럴 필요가 없다. Django는 FK관계에 있는 모델들을  `related_name`을 통해 부를 수 있게 한다. 예컨대 `pk=1`인 `Instagram` 객체가 가진 모든 `Comment(s)`들을 접근하기 위해서는 아래 코드와 같이 접근한다. 이는 `View`에서 뿐만아니라 `Template`에서도 적용된다.


#### 참고
```python
instagram = Instagram.objects.get(pk=1)
# Comment모델의 Instagram FK를 지정할 때 related_name을 'comments'로 지정했음을 기억한다
print(instagram.comments.all())    # returns <QuerySet [...]>

# iteration에서 접근
instagram_list = Instagram.objects
                          .prefetch_related('comments')
                          .all()
for instagram in instagram_list:
    print(instagram.comments.all()) # returns <QuerySet [...]> <QuerySet [...]> ....
    # TODO: 만일 각각의 comment를 보고싶다면 for문을 한번 더 돌면 될 것이다

```

따라서 `InstagramListView`에서 `Instagram objects`의 `queryset`을 `template`에 던질 때 **comments까지 함께 join하여 던져주면** 굳이 comment list가 필요하지 않다. `Template`에서 `instagram`객체들과 함께 던져진 `comments`들을 for문을 돌면서 표시하면 되기 때문이다.

이에 대해서는 후술한다.

> `InstagramListView`에서 `Instagram objects`의 `queryset`을 `template`에 던질 때 **comments까지 함께 join하여 던져주면** 굳이 comment > list가 필요하지 않다.


#### 결론

다음의 View를 구현한다.
* Instagram CreateView
* Instagram UpdateView
* Instagram DeleteView
* Instagram ListView

* Comment CreateView
* Comment UpdateView
* Comment DeleteView

## 2.1. Routing Rule 작성(urls.py)

가장 기본적인 url 라우팅을 정의한다.

#### Instagram_clone/urls.py
```python
# Instagram_clone(project root)/urls.py

urlpatterns = [
    path('admin/', admin.site.urls),
    # 각 앱 별 root url들을 추가해준다
    path('instagram/', include('instagrams.urls', namespace='instagrams')),
    path('comments/', include('comments.urls', namespace='comments')),
]
```

#### instagrams/urls.py
```python

app_name = 'instagrams'

urlpatterns = [
    # TODO: CBV들을 작성한다.
    path('', InstagramListView.as_view(), name='feed-list'),
    path('create/', InstagramCreateView.as_view(), name='feed-create'),
    path('update/<int:pk>/', InstagramUpdateView.as_view(), name='feed-update'),
    path('delete/<int:pk>/', InstagramDeleteView.as_view(), name='feed-delete'),
]

```

#### comments/urls.py
```python

app_name = 'comments'

urlpatterns = [
    # TODO: CBV들을 작성한다.

    # 주의! 댓글을 create할 때는 댓글을 작성하려는 'Instagram'객체를 찾아야 하므로 feed_pk를 사용한다.
    path('create/<int:feed_pk>/', CommentCreateView.as_view(), name='comment-create'),
    # 나머지 경우 댓글 자체를 업데이트하거나 삭제하기 때문에 그대로 comment의 pk를 사용한다.
    path('update/<int:pk>/', CommentUpdateView.as_view(), name='comment-update'),
    path('delete/<int:pk>/', CommentDeleteView.as_view(), name='comment-delete'),
]

```

## 2.2. View 작성


### 2.2.1. DeleteView

#### 2.2.1.1. InstagramDeleteView
쉬운것부터 먼저 처리하자. InstagramDeleteView는 굉장히 쉽다.

```python
class InstagramDeleteView(DeleteView):
    #TODO: 인증 및 처리
    model = Instagram
    success_url = reverse_lazy('instagrams:feed-list')
```
삭제하려는  모델은 `Instagram`이고 삭제에 성공하면 list로 돌아간다. 모델의 FK에 모두 CASCADE설정이 되어있으므로 따로 사진과 댓글 처리를 할 필요가 없다.

문제는 `CommentDeleteView`인데, `Comment`를 삭제하면 어디로 리다이렉트되어야 할까?

#### 생각해보기
이 파트는 스킵해도 무관하다.

위와 같이 `feed-list`로 리다이렉트해도 이 경우에는 사실 큰 문제가 없다. 그러나 만일 이 앱에 다른 모델이 추가될 계획이며, 그 모델에 대한 comment도 구상해야 한다면 이런 접근은 문제가 있다.

예컨대 우리가 만들고 있는 인스타그램 서비스에 일기(diary)모델이 추가된다고 생각해보자. 이경우에 `Comment`를 어떻게 설계해야 할 것인가?

`GenericForeignKey`를 제외하고 가장 생각하기 쉬운 접근은 `DiaryComment`와 `InstagramComment`라는 분리된 model을 생성하여 각각 `Diary`와 `Instagram`을 FK로 저장하는 것이다. 이경우 `DiaryCommentDeleteView`와 `InstagramCommentDeleteView` 를 각각 따로 생성해서 각각 `diary-detail`과 `feed-list`로 `success_url`을 지정해야 할 것이다.

꽤 직관적인 접근이지만 이 방식은 만일 Comment를 달아야할 모델이 더 추가될 예정이라면 view의 갯수가 선형적으로 증가하는 문제가 있다. 댓글을 달아야 하는 모델이 10개만 되어도 이를 모두 처리해주는 뷰는 40개정도가 되어야 할 것이다. 이는 Fat model, slim view를 지향하는 Django의 원칙과 거리가 있다.

내가 선호하는 방식은 **비슷한 성격의 모델은 하나의 Comment모델 안에 FK로 참조한다**이다. 이때 한 Comment모델이 커버하는 FK의 수는 2개를 넘지 않도록 하는게 경험상 좋다.

예컨대 `instagram`과 `diary`의 유일한 차이는 사진의 존재유무이며 나머지 구현 및 사용자 경험도 비슷할 예정이라면, 두 모델은 하나의 app에 포함될 가능성이 높다. 이경우 굳이 각각을 delete할 뷰를 생성할 필요는 없을 것이다.


아래를 보자.

```python
class Comment(Postable):
    diary = models.ForeignKey('diaries.Diary', on_delete=models.CASCADE)
    instagram = models.ForeignKey('instagrams.Instagram', on_delete=models.CASCADE)

    def get_absolute_url(self):
        if self.instagram:
            # 인스타그램에 대한 댓글인 경우 feed-list로 리다이렉트 한다
            return reverse('instagrams:feed-list')
        elif self.diary:
            # 일기에 대한 댓글인 경우 diary detail로 리다이렉트 한다
            # NotImplemented
            return reverse('diaries:diary-detail')

    def save(self, force_insert=False, force_update=False, using=None,
             update_fields=None):
        if self.instagram is None and self.diary is None:
            # 어떤 이유 때문에 FK가 동시에 지정되지 않았다면 이상한 경우이므로 저장하지 않는다.
            return
        elif self.instagram and self.diary:
            # 어떤 이유 때문에 FK가 동시에 지정되었다면 이상한 경우이므로 저장하지 않는다.
            return
        super(Comment, self).save()
```

경우에 따라 CommentManager를 오버라이드 하여 수정해줘야 할 수도 있다. 어쨌든 이렇게 모델을 설계하면 적은 View로 한번에 처리가 가능하다.

이 방식의 문제점은 복잡한 view를 구현해야 할때 접근이 까다롭다는 점이다(다형성을 구현한 모델을 삭제할 때 등). 이에 따라 혹자의 경우 이런 접근방식을 anti-pattern 취급하기도 한다. 다만 view 로직이 간단한 경우에는 큰 문제가 되지 않는다.

다시 원래 문제로 돌아가서. 이런 경우에 DeleteView를 처리하는 방법은 어떻게 해야할 것인가?

### 2.2.1.2. CommentDeleteView


사용자는 내가 작성한 댓글을 삭제한 후에는 내가 원래 보던 페이지로 이동하기를 기대한다. 즉, 댓글 삭제에 성공한 경우 사용자가 삭제하기 직전에 보고 있던 페이지의 주소로 리다이렉트시켜줘야 한다. 이를 구현하기 위해서 우리는 template의 comment delete template에서 `hidden input`으로 `next`라는 인자에 **현재 보고있는 페이지의 주소**를 함께 넘겨줄 것이다. `CommentDeleteView`에서는 이 `next`인자로 넘어온 주소를 받아 `success_url`로 활용할 것이다.

> 댓글 삭제할 경우, `form`에서 **hidden input**으로 현재 보고 있는 페이지 주소(`request.path`)를 form submit하면서 함께 넘긴다. `DeleteView`에서는 이를 참조하여 `success_url`로 활용한다. 이는 CreateView등 에서도 활용 가능하다.

따라서 구현은 다음과 같다.

```python
class CommentDeleteView(DeleteView):
    model = Comment

    def get_success_url(self):
        # 이전 페이지로 이동
        # POST request로 넘어온 'next'인자의 값을 읽어서 success_url로 던져준다
        to = self.request.POST.get('next', '/')
        return to
```

이제 우리는 댓글을 달아야 하는 모델이 늘어도 저 뷰 하나로 대응이 가능해졌다. 참고로 `create`한 이후에 리다이렉트 url을 지정해 줄때도 똑같은 패턴으로 작업이 가능하다. 그 예를 곧 볼 것이다.

### 2.2.2. CreateView

#### 2.2.2.1. CommentCreateView

댓글 작성 뷰는 간단하다.
- form으로 들어온 input을 `content`로 저장하는데, `author`를 현재 `request`를 넘긴 `user`로 지정해서 저장해주면 된다.
- 댓글을 생성한 이후에는 댓글을 달고 있던 바로 그 페이지로 리다이렉트 해야 한다.

```python
class CommentCreateView(CreateView):
    model=Comment
    fields = ['content']
    # TODO: 템플릿 작성
    template_name = 'comments/comments_container.html'

    def form_valid(self, form):
        # 잠깐 db 저장을 멈춘다
        comment = form.save(commit=False)

        # 현재 request를 요청한 user를 댓글의 작성자로 넣어준다
        comment.author = self.request.user

        # 현재 댓글이 달릴 instagram 객체의 pk는 routing rule의 <int:feed_pk>로 넘어온다
        comment.instagram = Instagram.objects.get(pk=self.kwargs.get('feed_pk'))
        comment.save()
        # 댓글을 생성한 이후 댓글을 달고 있었던 request url로 리다이렉트한다.
        return HttpResponseRedirect(self.request.POST.get('next', '/'))
```

#### 2.2.2.2. InstagramCreateView

`Instagram`을 생성할 때 우리는 `InstagramPhoto`객체들도 함께 생성해줘야 한다. 즉, `CreateView`가 제공하는 form에 더하여 image들을 받을 수 있는 input을 추가해야 한다는 뜻이다.

해야할 일은 명확하다. `CreateView`가 기본적으로 제공하는 form대신 `image`를 포함하는 새로운 form을 생성해야한다.

> `CreateView`나 `UpdateView`가 기본적으로 제공하는 form에 불만족스럽다면 새로운 form을 만들고 각 View에서 `form_class`로 참조한다.


#### instagrams/forms.py
```python
class InstagramForm(ModelForm):
    class Meta:
        model=Instagram
        fields=['content']

    # 여러개의 FileInput을 허용하는 새로운 'images'필드를 정의한다.
    images = forms.FileField(widget=ClearableFileInput(attrs={'multiple': True}), label='사진')

```


- form에서 view로 넘길 건 content 하나뿐이다. author등은 view에서 처리한다
- Django forms에서 제공하는 FileField를 이용해서 images라는 새로운 필드를 정의한다. View는 이 필드에서 넘어온 파일들을 `InstagramPhoto`로 저장할 것이다.
- `multiple`이 `True`인 것에 유의한다. 이는 template에서 form이 렌더링될 때 input에 multiple을 삽입해준다.

#### InstagramCreateView

```python
class InstagramCreateView(CreateView):
    model = Instagram
    form_class = InstagramForm
    success_url = reverse_lazy('instagrams:feed-list')

    def form_valid(self, form):
        # 잠깐 save를 막고 현재 user를 author로 넣어준다
        instagram = form.save(commit=False)
        instagram.author = self.request.user
        instagram.save()

        # 만일 requst로 FILE이 넘어온다면 InstagramPhoto로 간주하고 객체들을 생성한다
        if self.request.FILES:
            # forms.py에서 images라는 이름으로 FileField를 정의했음을 기억한다
            for f in self.request.FILES.getlist('images'):
                feed_photo = InstagramPhoto(instagram=instagram, photo=f)
                feed_photo.save()
        # 수퍼클래스의 form_valid 구현을 따로 수정할 필요는 없다.
        return super(InstagramCreateView, self).form_valid(form)
```


- form에서 따로 author를 input으로 받지 않으므로 view에서 처리한다.
- `type=file`로 넘어온 데이터는 `request.FILES`에 담겨서 넘어온다.[참고](https://docs.djangoproject.com/ko/2.1/topics/http/file-uploads/#uploading-multiple-files)
- 그외에 `form_valid`의 행동을 바꿀 이유는 없으므로 그대로 super class의 `form_valid`를 불러준다


### 2.2.3. UpdateView
#### 2.2.3.1. CommentUpdateView

기본적으로 `UpdateView`는 구현이 `CreateView`와 비슷하지만 더 간단하다. 이미 우리는 `Comment`의 `author`가 있기 때문에 업데이트 해줄 필요가 없다.

근데 Update가 완료된 이후에는 어디로 리다이렉트 시켜줘야 할까? 상기한 `next`인자를 이용한 처리는 힘들다. 왜냐하면 update는 `comment_update_form.html`에서 이루어지기 때문이다. 이 html파일에서 `next`인자를 이용하면 update가 완료된 후 직전 페이지인 `comment_update_form.html`을 다시 부르게 될 것이다. 이는 우리가 원하는 로직이 아니다.

생각해보면 `Instagram`에 `Comment`를 달았으면 다시 feed로 돌아가는 것이 이치에 합당하다.
따라서 `Comment`모델에 `get_absolute_url()`을 feed-list로 재정의하여 `success_url`을 그곳으로 리다이렉트 시켜줄 것이다.


#### comments/views.py
```python
class CommentUpdateView(UpdateView):
    model = Comment
    # TODO: 템플릿 작성
    template_name_suffix = '_update_form'
    fields = ['content']

    def get_success_url(self):
        # 현재 Comment의 absolute_url로 리다이렉트 한다.
        return Comment.objects.get(pk=self.kwargs['pk']).get_absolute_url()

```

#### comments/models.py
```python
class Comment(Postable):
    instagram = models.ForeignKey('instagrams.Instagram',
                                  on_delete=models.CASCADE,
                                  related_name='comments',
                                  verbose_name='인스타그램')

    def get_absolute_url(self):
        # Instagram feed-list로 리다이렉트한다.
        return reverse('instagrams:feed-list')
```


- `Comment`의 update가 완료되면 현재 `Comment`의 `absolute_url`로 리다이렉트한다
- 현재 `Comment`는 항상 `feed-list`를 가리킨다.


눈치가 빠른 독자들이라면 굳이 이렇게 할 필요 없이 바로 `CommentUpdateView`의 `get_success_url`에서 `feed-list`를 reverse하여 던져주면 된다는 것을 눈치챘을 것이다. 하지만 그렇게 하지 않은 이유는 다음과 같다.

1. 일반적으로 model에 `absolute url`을 정의하는 것이 코드 확장성, 간결성 측면에서 좋다.
2. 상기한 한 Comment모델이 2개 이상의 FK를 가지는 아키텍쳐를 구상했다면 더더욱 코드 확장성 측면에서 `absolute url`을 활용하는 것이 월등하다.

예컨대 다음 코드를 보자. 이는 실제로 웹사이트 구축 프로젝트에서 필자가 사용한 코드이다.

#### 참고 코드
```python
class Comment(Postable):
    instagram = models.ForeignKey('core.Instagram', default=None, null=True, blank=True, related_name='comments',
                                  on_delete=models.CASCADE)
    meeting = models.ForeignKey('core.Meeting', default=None, null=True, blank=True, related_name='comments',
                                on_delete=models.CASCADE)
    objects = CommentManager()

    class Meta:
        verbose_name = _('댓글')
        verbose_name_plural = _('댓글')

    def __str__(self):
        return self.content

    def get_absolute_url(self):
        if self.instagram:
            # partner:meeting-list로 redirect
            return reverse('partners:meeting-list')
        elif self.meeting:
            # downcast후 각 클래스의 get_absolute_url로 redirect
            return self.meeting.cast().get_absolute_url()

```

각 FK중 무엇이 존재하느냐에 따라 `absolute_url`을 다르게 던져줬다. 이제 어떠한 View에서도 저 함수만 호출하면 손쉽게 리다이렉트 가능하다. `absolute_url`이 필요한 모든 view에서 저런식으로 분기를 나눈다고 상상해보자. 짜증이 날 것이다.


#### 2.2.3.2. InstagramUpdateView

`UpdateView`는 간단하다. form은 이전에 정의한 `InstagramForm`을 그대로 갖다쓰면 되고 명세에서 정의한대로 Update시에는 기존 사진들을 모두 삭제하고 새 사진들로 채워준다.

```python
class InstagramUpdateView(UpdateView):
    model = Instagram
    form_class = InstagramForm
    success_url = reverse_lazy('instagrams:feed-list')

    def form_valid(self, form):
        instance = form.save()
        # 현재 인스타그램과 연결된 사진들을 삭제하고 다시 만들어준다.
        InstagramPhoto.objects.filter(instagram=instance).delete()
        if self.request.FILES:
            for f in self.request.FILES.getlist('images'):
                feed_photo = InstagramPhoto(instagram=instance, image=f)
                feed_photo.save()

        return super(InstagramUpdateView, self).form_valid(form)
```



### 2.2.4. InstagramListView

이 부분에서는 생각해야 할 지점이 많다. 체크리스트를 보면 이 뷰는 다음 사항들을 보여줘야 한다.
1. Instagram 모델 자체
2. InstagramPhotos 모델
3. Comments 리스트
4. Comment를 작성하게 해주는 input form

Django에 익숙한 사람이라면 1,2는 크게 어렵지 않을 것이다. 그리고 상기한 논의에서 3역시 크게 문제가 되지 않는다는 사실도 알았다.

그런데 4는 어떻게 처리해야 할까?

#### FormMixin

Django는 request에서 form을 보여주고 처리할 수 있는 심플한 [`FormMixin`](https://docs.djangoproject.com/ko/2.1/topics/class-based-views/mixins/#using-formmixin-with-detailview)을 제공한다. 당연히 form을 렌더링할 것이 예상된는 `UpdateView`나 `CreateView`도 소스코드를 뜯어보면 이 `FormMixin`을 상속받은 `ModelFormMixin`을 상속받고 있음을 알 수 있다.

이를 이용하면 우리가 이용하려는 `ListView`에서도 Form을 handling할 수 있을 것 같다. `DetailView`에서 이 mixin을 이용한 예제는 위 django 공식문서를 참고한다.

이를 사용하려면 당연히 `Comment`를 생성하는 `Form`을 먼저 생성해야 할 것이다.


#### comments/forms.py
바로 코드로 넘어가자.

```python
class CommentForm(ModelForm):
    class Meta:
        model = Comment
        fields = ['content']

    def __init__(self, *args, **kwargs):
        super(CommentForm, self).__init__(*args, **kwargs)

        # 최대한 Instagram과 비슷하게 스타일링한다
        self.fields['content'].widget.attrs['class'] = 'textfield-comment'
        self.fields['content'].widget.attrs['placeholder'] = '댓글 달기...'
        self.fields['content'].label = ''
        # FIXME: 필요에 따라 Text필드가 너무 넓어보인다면 한줄로 제한한다.
        # self.fields['content'].widget.attrs['rows'] = 1
```


- `ModelForm`을 이용해서 폼을 작성한다.
- input은 댓글내용(content)밖에 없다. 작성자 등의 정보는 view에서 알아서 넣어줄 것이다.
- html 설정으로 `class`와 `placeholder`를 넣어준다. class는 나중에 css 스타일링에서 필요할 것이다.
- `label`은 form을 template에 렌더링할때 input박스 앞에 나타나는 각 필드의 이름값이다. 이를 없애줘야 한다.
- 현재 `content`는 텍스트필드로 선언되었으므로 일반적으로 보는 길쭉한 text input창이 나올 것이다. 댓글 입력창은 한줄이 예쁘므로 필요한 경우 1줄로 세팅해준다.

label 예시:

![라벨](https://i.stack.imgur.com/JeRno.png)

자 이제 모든 준비가 끝났으니 진짜 view를 구현해보자.

#### InstagramListView 구현

```python
class InstagramListView(FormMixin, ListView):
    form_class = CommentForm
    paginate_by = 20
    # TODO: 템플릿을 만들자
    template_name = 'instagram/feed_list.html'

    def get_context_data(self, **kwargs):
        # superclass의 get_context_data를 부른다
        context = super(InstagramListView, self).get_context_data(**kwargs)
        # CommentForm을 context_data에 넣어준다
        context['comment_form'] = self.get_form()
        return context

    def get_queryset(self):
        # Instagram 객체들을 가져오는데 FK에 대한 joining을 모조리 해서 가져온다.
        # Reverse relationship을 join해서 가져오기 위해 2번의 prefetch_related를 한다
        # 마지막으로 단순 FK를 join해서 가져오기 위해 1번의 select_related를 한다
        # 최신 객체가 가장 위에 올라와야 하고 그다음으로는 최근 수정된 순으로 객체를 뿌려줘야 한다.
        queryset = Instagram.objects\
            .prefetch_related('photos')\
            .prefetch_related('comments__author')\
            .select_related('author')\
            .order_by('-created', '-updated')
        return queryset
```


- FormMixin을 먼저 상속받고 ListView를 상속받는다.
- `self.get_form()`메소드는 `FormMixin`의 메소드로 위에서 선언한 `form_class`객체를 가져온다
- 이제 `instagram/feed_list.html`에서 `{{comment_form}}` 으로 form을 렌더링 가능할 것이다
- `queryset`의 경우 `Instagram`과 연관된 모든 모델을 join하여 가져와야 한다.
- `InstagramPhoto`와 `Comment`의 경우 reverse relationship이므로 `prefetch_related`를 이용한다.(`related_name`에 유의한다)
- 일반적으로 객체들을 리스트로 뿌려줄때 최신순 - 수정순 으로 뿌려주므로 해당 처리를 해준다.


## 3. 정리

지금까지 우리는 다음 View들을 정의했다.

1. DeleteView
    1. InstagramDeleteView
    2. CommentDeleteView
        - `next`인자 hidden input 패턴
        - one Model multiple FK (anti)패턴

2. CreateView
    1. InstagramCreateView
        - `request.FILES.getlist('images')`
    2. CommentCreateView
        - `next`인자 hidden input 패턴

3. UpdateView
    1. InstagramUpdateView
    2. CommentUpdateView
        - `get_absolute_url()`

4. ListView
    1. InstagramListView
        - `queryset`에서 `prefetch_related`, `select_related`활용
        - `FormMixin`으로 `CommentForm`을 넣어줌

맨 처음 작성했던 checklist에서 얼마나 구현되었는지 보자.

- [x] Instagram 객체를 불러와서 화면에 뿌려준다
    - [x] 이 때 `Instagram` 객체에 연결된 `InstagramPhoto` 객체들을 모두 로드해서 뿌린다.
    - [x] `Instagram` 객체에 연결된 Comment 객체들을 모두 로드해서 뿌린다.
        - [x] 이때 뿌리는 형식은 list여야 한다.
    - [x] `Instagram` 객체 하단에 Comment input form이 들어가야 한다.
    - [x] `Instagram` 객체에는 업로더 정보, 업로드 날짜, 내용이 들어가야 한다.

- [x] `Instagram` 객체 여러개를 불러와서 화면에 뿌려준다.
    - [ ] 오로지 작성자와 superuser만이 객체를 수정/삭제 할 수 있어야 한다.


이제 Authentication 관련한 처리만 남았다. 이는 다음 포스팅에서 처리하도록 하자.











