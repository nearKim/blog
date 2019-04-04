---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.3"
description: "View develop하기: Basic Authentication"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기 Part.3



1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV)
3. View develop하기 **(current)**
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

## 1. Authentication 개요

개인적으로 FBV보다 CBV를 좋아한다. 물론 FBV는 빠른 개발이 가능하고 직관적이라는 큰 장점이 있다. 하지만 View가 해야 하는 기능들을 모듈화하여 코드의 간결성을 유지한다는 측면에서는 CBV, 특히 generic CBV가 좋은 선택이다.

예컨대 누군가 `form_valid()`를 오버라이드했다면, 이사람은 form을 저장하는 과정에서 어떤 처리를 하고 있다는 것을 바로 알 수 있다. 이러한 간결성은 PHP와 같은 언어에서는 기대하기 힘든 Django의 강점이다. 그렇다면 `Authentication`은 CBV의 어느 부분에서 다뤄져야 할까?


이미 Django는 기본적인 Auth기능을 제공하기 위한 심플한 Mixin을 구현해놓았다. 예컨대 로그인하지 않은 유저(Anonymous User)가 해당 View에 접근하는 것을 막고 싶다면 [`LoginRequiredMixin`](https://docs.djangoproject.com/en/2.1/topics/auth/default/#the-loginrequired-mixin)를 상속받으면 된다. 그럼 이 `LoginRequiredMixin`은 도대체 무슨일을 하는걸까?

코드를 보자.

```python
class LoginRequiredMixin(AccessMixin):
    """Verify that the current user is authenticated."""
    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return self.handle_no_permission()
        return super().dispatch(request, *args, **kwargs)
```

코드는 심플하다. `dispatch()`메소드에서 만일 user가 `is_authenticated`되지 않았다면 `handle_no_permission()`을 호출한다. 이 핸들러 메소드는 물론 `PermissionDenied()`를 띄울 것이다. 만일 로그인이 되어있었다면 그대로 `dispatch()`로 진행한다. 그럼 이 메소드는 도대체 뭐하는 메소드일까?


### dispatch() 메소드

`dispatch(request, *args, **kwargs)`메소드는 `request`와 다른 인자들을 받아서 `HTTP response`를 리턴한다. 즉, 이 메소드는 `request`를 분석하여 해당 요청이 `GET`인지 `POST`인지 등을 파악하고 알맞는 View내의 다른 메소드들로 `request`를 전달한다. 대략적인 플로우는 다음과 같다.

1. 사용자가 정의된 url pattern에 맞게 입력하여 request를 보낸다.
2. Django의 url resolver가 해당 url을 분석하여 `urls.py`에 명시된 View **function**으로 `request`와 인자들을 넘긴다.
    - 여기서 url resolver는 항상 function으로 넘길 것을 기대하므로 CBV를 사용할 경우 `as_view()`메소드를 urlpattern에서 사용해야 한다.
3. 전달된 인자들(pk 등)은 `as_view()`메소드에서 처리하여 `kwargs`나 `args`로 넘겨준다.
4. `as_view()`에서 처리가 완료되면 `dispatch()`를 호출하여 `request`, `kwargs`, `args`를 넘겨준다.
5. `request`의 메소드에 따라 적절한 핸들러를 호출한다.


이제 왜 `LoginRequiredMixin`이 `dispatch`메소드를 오버라이드 했는지 알거 같다. 권한이 없는 사용자의 접근은 View 로직에 들어가기도 전에 차단하는게 타당할 것이다. `dispatch()`는 라우팅된 url로 전해진 request를 받는 수문장의 역할을 하고 있는 것이다. 통과하지 못했다면 뒤돌아볼 것도 없는 것이다.

이제 이 `dispatch()`메소드를 수정해보자.


## 2. ValidAuthorRequiredMixin 만들기

일단 상식적으로 로그인한 사람만이 객체를 보거나 생성할 수 있어야 할 것이다. 그리고 지난 포스팅에서 마지막으로 남은 체크리스트를 생각해보자.

> 1. 객체 생성과 객체정보(리스트) 보기는 로그인한 사용자만 가능하다.
> 2. 오로지 작성자와 superuser만이 객체를 수정/삭제 할 수 있어야 한다.



### 객체 생성과 객체정보(리스트) 보기는 로그인한 사용자만 가능하다
첫째 조건은 매우 쉽다. 위의 `LoginRequiredMixin`을 `InstagramCreateView`, `CommentCreateView`, `InstagramListView`에 추가하면 된다.

```python
# instagrams/views.py
class InstagramCreateView(LoginRequiredMixin, CreateView):
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


class InstagramListView(LoginRequiredMixin, FormMixin, ListView):
    form_class = CommentForm
    paginate_by = 20
    # TODO: 템플릿 작성
    template_name = 'instagram/feed_list.html'

    def get_context_data(self, **kwargs):
        # superclass의 get_context_data를 부른다
        context = super(InstagramListView, super).get_context_data(**kwargs)
        # CommentForm을 context_data에 넣어준다
        context['comment_form'] = self.get_form()
        return context

    def get_queryset(self):
        # Instagram 객체들을 가져오는데 FK에 대한 joining을 모조리 해서 가져온다.
        # Reverse relationship을 join해서 가져오기 위해 2번의 prefetch_related를 한다
        # 마지막으로 단순 FK를 join해서 가져오기 위해 1번의 select_related를 한다
        # 최신 객체가 가장 위에 올라와야 하고 그다음으로는 최근 수정된 순으로 객체를 뿌려줘야 한다.
        queryset = Instagram.objects \
            .prefetch_related('photos') \
            .prefetch_related('comments__author') \
            .select_related('author') \
            .order_by('-created', '-updated')
        return queryset

# comments/views.py
class CommentCreateView(LoginRequiredMixin, CreateView):
    model = Comment
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

### 오로지 작성자와 superuser만이 객체를 수정/삭제 할 수 있어야 한다.

작성자와 superuser는 로그인한 사람 중의 일부이므로 `LoginRequiredMixin`으로는 커버할 수 없다. 바로 떠오르는 생각으로는, 일단 `LoginRequiredMixin`을 각 View에서 상속받은 후, 다시 `dispatch()`를 오버라이드하여 관련 로직을 추가하는 방법이 있겠다. 그런데 지금 수정해야 하는 뷰가 4개 아닌가? 만일 app이 확장돼서 더욱 많은 View가 추가된다면?

#### ValidAuthorRequiredMixin

우리는 원하는 로직을 구현하기 위해서 `dispatch()`만 건드리면 충분하다는 걸 안다. 그렇다면 그냥 새로운 `Mixin`을 만들면 되는 것 아닌가?

가장 처음 만들었던 `instagram_clone/mixins.py`를 열고 다음 코드를 추가하자.

```python
class ValidAuthorRequiredMixin(AccessMixin):
    """상속받은 객체의 author가 운영진이거나 객체의 author가 아니면 403을 반환한다"""

    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            # 애초에 로그인을 안했으면 거부한다.
            return self.handle_no_permission()
        elif self.get_object().author != request.user and not request.user.is_staff:
            # 상속받은 객체의 author가 현재 user가 아니고 운영진도 아니라면 거부한다.
            raise PermissionDenied
        else:
            return super(ValidAuthorRequiredMixin, self).dispatch(request, *args, **kwargs)

```

- if문의 첫부분은 `LoginRequiredMixin`과 동일하다
- 이 Mixin을 상속받을 클래스의 object의 `author`필드가 현재 유저가 아니고 운영진도 아니라면 거부한다.
- 그 외의 경우 그대로 진행한다.

물론 `is_staff`대신 `is_superuser`를 사용하면 최고관리자만 접근하도록 할 수 있다. 필요에 따라 적용한다.


```python
# instagrams/views.py
class InstagramUpdateView(ValidAuthorRequiredMixin, UpdateView):
    model = Instagram
    form_class = InstagramForm
    success_url = reverse_lazy('instagrams:feed-list')

    def form_valid(self, form):
        instance = form.save()
        # 현재 인스타그램과 연결된 사진들을 삭제하고 다시 만들어준다.
        InstagramPhoto.objects.filter(instagram=instance).delete()
        if self.request.FILES:
            for f in self.request.FILES.getlist('images'):
                feed_photo = InstagramPhoto(instagram=instance, photo=f)
                feed_photo.save()

        return super(InstagramUpdateView, self).form_valid(form)


class InstagramDeleteView(ValidAuthorRequiredMixin, DeleteView):
    model = Instagram
    success_url = reverse_lazy('instagrams:feed-list')

# comments/views.py
class CommentUpdateView(ValidAuthorRequiredMixin, UpdateView):
    model = Comment
    # TODO: 템플릿 작성
    template_name_suffix = '_update_form'
    fields = ['content']

    def get_success_url(self):
        # 현재 Comment의 absolute_url로 리다이렉트 한다.
        return Comment.objects.get(pk=self.kwargs['pk']).get_absolute_url()


class CommentDeleteView(ValidAuthorRequiredMixin, DeleteView):
    model = Comment

    def get_success_url(self):
        # 이전 페이지로 이동
        # POST request로 넘어온 'next'인자의 값을 읽어서 success_url로 던져준다
        to = self.request.POST.get('next', '/')
        return to
```

## 2. Authentication Views

지금까지 작성한 모든 View들은 로그인 한 사용자들만 접근 할 수 있다. 이제 로그인 및 로그아웃 등을 처리해주는 View를 만들어보자.
Django는 기본적으로 authentication에 사용되는 기능들을 자체적으로 래핑하여 제공하는데, 이러한 View들은 `django.contrib.auth`에 들어있다. 이 View들을 한꺼번에 사용하고 싶으면 다음과 같이 url을 추가한다.

```python
# instagram_clone/urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('instagram/', include('instagrams.urls', namespace='instagrams')),
    path('comments/', include('comments.urls', namespace='comments')),
    path('accounts/', include('django.contrib.auth.urls')), # 추가됨
]
```

`accounts`라는 네임스페이스로 authentication에 필요한 모든 url들을 라우팅 맵에 추가하였다. Django 공식 문서에 따르면 저 한줄은 다음과 같은 url pattern들을 자동으로 추가해준다.
[참고](https://docs.djangoproject.com/ko/2.1/topics/auth/default/#module-django.contrib.auth.views)

```
accounts/login/ [name='login']
accounts/logout/ [name='logout']
accounts/password_change/ [name='password_change']
accounts/password_change/done/ [name='password_change_done']
accounts/password_reset/ [name='password_reset']
accounts/password_reset/done/ [name='password_reset_done']
accounts/reset/<uidb64>/<token>/ [name='password_reset_confirm']
accounts/reset/done/ [name='password_reset_complete']
```

url의 이름만 봐도 모든 웹사이트가 제공하는 기본적인 회원관리 기능임을 알 수 있다. 이제 사용자는 저 url들로 접근하여 로그인 등의 처리가 가능하다.
물론 각 View에서 참조할 **template들을 추가**하는 일이 남았다.

## 3. 정리

- CBV에서 기본적인 Authentication은 Mixin의 형태로 제공된다.
- 이 Mixin들은 대부분 `dispatch()`를 오버라이드 하여 원하는 기능을 제공한다.
- `dispatch()`메소드는 url resolver로부터 받은 `request`와 각종 `args`, `kwargs`를 래핑하여 전달되어야 하는 CBV내의 다른 메소드로 전파(dispatch)하는 역할을 한다.
- 이에 따라 View 로직이 구현되기 앞서 실행되는 수문장의 역할을 한다고 볼 수 있다.
- 인증되지 않은 request는 뷰 로직이 실행되기 전에 종료시키는게 타당하므로, 우리는 `dispatch()`를 오버라이드하여 원하는 기능을 구현한다.
- 기본적인 Auth기능의 경우 따로 View를 작성할 필요 없이 Django가 기본으로 제공하는 url들을 path로 지정해준다.
- 만일 이 기능들을 오버라이드 하고 싶다면, `django.contrib.auth`에 포함된 View들을 오버라이드 하여 path안에 넣어준다.
