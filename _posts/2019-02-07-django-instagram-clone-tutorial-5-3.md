---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.5 (3)"
description: "기타 짜잘한 디자인 완성시키기"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기 Part.5 (3)

1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV)
3. View develop하기
4. Django template 기본 구현
5. Django template 완전 구현
    1. 상용 서비스와 같은 디자인 골격 잡기
    2. 수정/삭제 Dropdown 및 댓글창 예쁘게 만들기
    3. 기타 짜잘한 디자인 완성시키기 **(current)**
    4. AJAX를 이용한 기본적인 댓글 시스템 구현
6. 좀더 멋있는 기능 추가하기
    1. Infinite scroll 구현하기
    2. 사진 Carousel 붙이기

<hr>


본 절은 바로 수정작업으로 들어간다. 지금까지 써놨던 TODO들을 최대한 없애보자.

## 1. 시간 표시 일정기간 지나기 전엔  'n 시간 전'으로 표시하기
상용 서비스에서는 다음의 규칙에 따라 포스팅 시간을 표시한다.

현재 시간으로부터


1. 1분이 지나지 않은 경우 `n seconds ago (n초 전)`
2. 1시간이 지나지 않은 경우 `n minutes ago (n분 전)`
3. 하루가 지나지 않은 경우 `n hours ago (n시간 전)`
4. 1주일이 지나지 않은 경우 `n days ago (n일 전)`
5. 그 이상 지난 경우 `포스팅 날짜`


간단한 알고리즘 문제로 볼 수 있다. 모든 Instagram 객체에 대해서 생성시간을 바꿔줘야 하므로 model method로 처리하는게 좋을 것 같다.

그 전에 해야할 일이 하나 있다. 데이터베이스에 저장할 때, 시간의 경우 `UTC`로 저장된다. 하지만 우리는 `Asia/Seoul`시간대에 살고 있다. 사실 위의 1,2,3,4의 경우 어차피 시간의 차이만 필요하므로 상관이 없지만 5의 경우 UTC시간이 아닌 로컬 시간 으로 변환해줘야 한다.

### 1.1. Django 시간 및 timezone

시간대 변환은 프로그래밍 언어 전반에 걸쳐 매우 번거롭고 짜증나지만 Django는 settings에 기본 옵션을 포함시켜서 사용자의 수고를 덜어주었다.

```python
# Instagram_clone/settings.py

TIME_ZONE = 'Asia/Seoul'

```

이렇게 변경하고 저장한 후 python console을 실행시켜보자. 다음 커맨드를 입력한다.

```python
# Wrong!
# from datetime import timezone
from django.utils import timezone
from datetime import datetime

# UTC와 Asia/Seoul간 9시간의 시간차에 주목한다
timezone.get_current_timezone()  # <DstTzInfo 'Asia/Seoul' LMT+8:28:00 STD>
timezone.localtime()             # datetime.datetime(2019, 2, 9, 15, 46, 35, 307211, tzinfo=<DstTzInfo 'Asia/Seoul' KST+9:00:00 STD>)

datetime.now()                   # datetime.datetime(2019, 2, 9, 15, 46, 8, 786711)
timezone.now()                   # datetime.datetime(2019, 2, 9, 6, 46, 59, 201354, tzinfo=<UTC>)

# Important! 기존의 Timezone 정보가 존재하는 datetime 객체의 Timezone을 바꿀 수 있다.
timezone.localtime(timezone.now(), timezone.get_current_timezone()) # datetime.datetime(2019, 2, 9, 15, 46, 9, 620556, tzinfo=<DstTzInfo 'Asia/Seoul' KST+9:00:00 STD>)
timezone.localtime(datetime.now(), timezone.get_current_timezone()) # ValueError: localtime() cannot be applied to a naive datetime
```

- `django.utils`의 `timezone`을 사용했음을 주목한다. `datetime`의 `timezone`이 아니다!
- `settings.py`에 적힌 시간대를 불러올 수 있다.
- `localtime()`메소드는 value, timezone의 두개의 파라미터를 받을 수 있는데, timezone이 명시된 경우 value를 해당 timezone으로 변환해준다.
- 이 때 당연히 value는 timezone 정보가 존재해야 한다. timezone 정보가 없는 naive datetime이 들어가면 에러를 발생시킨다.


### 1.2. Django 시간 변환 적용

이를 이용해서 위 로직을 model method로 추가해보자.

```python
# instagrams/models.py
from django.utils import timezone
from datetime import datetime, timedelta

class Instagram(Postable):
    @property
    def timedelta_string(self):
        # settings에서 Timezone을 'Asia/Seoul'로 변경했을 경우 now()는 로컬타임이지만 DB에 저장된 객체들의 시간은 UTC이다.
        # 차이값을 구하기위해 timezone을 맞춰준다. 이 때 datetime의 timezone이 아닌 django.utils의 timezone을 사용한다.
        delta = datetime.now(tz=timezone.utc) - self.created

        if delta < timedelta(minutes=1):
            return str(delta.seconds) + ' 초 전'
        elif delta < timedelta(hours=1):
            return str(int(delta.seconds / 60)) + ' 분 전'
        elif delta < timedelta(days=1):
            return str(int(delta.seconds / 3600)) + ' 시간 전'
        elif delta < timedelta(days=7):
            # 날짜만 따로 빼서 날짜간의 차이를 계산한 후 리턴한다.
            delta = datetime.now(tz=timezone.utc).date() - self.created.date()
            return str(delta.days) + ' 일 전'
        else:
            # settings.py에 변경한 로컬 시간대를 기준으로 UTC 시간을 변환하여 리턴한다.
            return timezone.localtime(self.created, timezone.get_current_timezone())
```

- 한국사람들이 'n 일전'이라고 얘기할 때는 현재 시각과 상관없이 날짜의 차이만을 말하는 경우가 많다.
- 예컨대 만일 [2월 3일 오전 6시]에 [2월 1일 오후 9시]에 작성된 게시글을 볼 경우, 33시간 전이므로 `delta.days`는 이를 내림한 '1'으로 표시된다.
- 이 경우 일상에서 2일전으로 많이 얘기하므로 단순히 `delta.days`를 리턴하지 말고 적절하게 변환해줘야 한다.


이제 template의 instagram iteration에서 해당 모델 메소드를 불러서 시간을 표시할 수 있다.

```django
<!-- feed_list.html -->

{%raw%}
... 전략
    {# 6. 6번째 row: instagram 작성 시간이 현재시간과의 '시간차'로 나타난다. #}
    <div class="created-time row">
        <div class="col-12">{{ instagram.timedelta_string }}</div>
    </div>
    {# 7. 7번째 row: 댓글 입력 form이 들어간다#}

...후략

{%endraw%}
```

이제 새로고침하여 페이지를 다시 살펴보자.

{% include image.html path="documentation/insta-clone-pt5-17.png" path-detail="documentation/insta-clone-pt5-17.png" alt="image" %}

잘 변경된 것을 알 수 있다.


## 2. 말풍선 모양 아이콘 넣고 댓글 입력창과 연동

상용 서비스에서는 Like버튼(하트 모양), 댓글버튼(말풍선 모양), 공유버튼이 존재한다. 여기서 말풍선을 누르면 자동적으로 댓글 입력창이 활성화되고 깜박이게 된다.

### 2.1. 말풍선 모양 아이콘 넣기
현재 우리는 못생긴 일반 버튼을 사용하고 있다. Fonstawesome 스타일 시트를 글로벌에 추가해줬으므로 해당 아이콘을 사용하자.

```django
<!-- feed_list.html -->
{%raw%}
...전략
    {# 3. 3번째 row: 말풍선 모양 버튼들이 들어간다. 추후 좋아요 등을 구현하면 여기에 넣는다.  #}
    <div class="button-container row">
        <button class="add-comment-balloon"><i class="far fa-comment"></i></button>
    </div>
...후략
{%endraw%}
```

```css
<!-- feed_list_style.css -->

.add-comment-balloon {
    margin-left: 15px;
    padding-left: 0;

    border: none;
    background: none;
    font-size: 30px;
    cursor:pointer;
}

.add-comment-balloon:focus {
    outline: none;
}
```

- 아이콘 위치는 말풍선 아래에 위치한 author의 시작점과 동일하게 맞춘다.
- author정보는 `col-3`에 해당하는 bootstrap grid에 들어있고 디폴트 좌측정렬 되어 있으므로 해당 div의 `margin-left`만큼 띄워주면 된다. 각자 개발자도구를 열어서 확인하고 넣어준다.
- 기존의 button의 스타일을 모두 없애주기 위해 border, background를 없애준다.
- 마우스를 얹으면 포인터모양으로 바뀌어야 한다.
- 폰트는 30px정도면 좋을 것 같다.
- 클릭하면(focus) 파란색 테두리가 나타나는데 이를 없애준다.(outline: none)

저장하고 새로고침 해보자.

{% include image.html path="documentation/insta-clone-pt5-18.png" path-detail="documentation/insta-clone-pt5-18.png" alt="image" %}

이제 정말 뭔가 인스타그램스러워지고 있다!

### 2.2. 누르면 아래쪽 댓글창 포커싱되도록 하기
어려운 작업이 아니므로 바로 js작업에 들어간다.

```javascript
<!-- feed_list_js.js -->
$(document).ready(function () {
    $(window).on('click', function (event) {
        // 0. 공통 변수 초기화
        let target = $(event.target)

...생략

        // 클릭한게 icon이 아닌 경우 window를 클릭한 것이다.
        if (!event.target.matches('i')) {
            ...중략

        // TODO: 아래와 같이 수정한다.
        } else if (target.hasClass('fa-ellipsis-v')) {
            ...중략

        // TODO: 아래 구문을 추가한다.
        } else if (target.hasClass('fa-comment')) {
            // 1. 말풍선 기준으로 가장 가까운 .button-container div를 찾는다.
            // 2. 그것과 동일한 위치의 .comment-creator-div div를 찾는다.
            // 3. 하위 요소를 순회하며 textarea 를 찾으면 그것이 포커싱하고자하는 댓글창이다
            target.closest('.button-container')
                .siblings('.comment-creator-div')
                .find('textarea')[0]
                .focus()
        }

...후략

```

- 댓글 아이콘을 클릭은 윈도우를 클릭 이벤트에 잡히므로 해당 이벤트를 분석한다
- 클릭 이벤트를 여러번 분기할 가능성이 있으므로 공통변수를 초기화한다. (기존 코드 refactor)
- 만일 클릭한 fontawesome 아이콘이 '...' 모양이었다면 기존의 드롭다운 여닫는 로직을 수행한다.
- 만일 클릭한 fontawesome 아이콘이 말풍선 모양이었다면 `textarea`에 포커싱을 맞춘다.
- 댓글을 받는 input은 form으로 처리하고 있는데 개발자도구를 활용해보면 textarea로 되어있는 것을 알 수 있다.
- jquery를 적절히 활용하여 해당 textarea를 포커싱해준다.

저장한 후 새로고침을 해보고 아이콘을 눌러보자.



{% include image.html path="documentation/insta-clone-pt5-19.gif" path-detail="documentation/insta-clone-pt5-19.gif" alt="image" %}

## 3. 레이아웃 예쁘게 하기
Template 설계 할 때 작성한 row 중 말풍선 아래쪽 row 부터 스타일을 적용한다.

### 3.1. 4번째 row
4번째 row는 업로더 정보와 본문을 담고 있다. 편의상 col-3과 col-9로 나눈 결과 업로더와 내용 사이 여백이 많아졌다. 상용 Instagram은 어떻게 이 정보들을 관리할까?

{% include image.html path="documentation/insta-clone-pt5-20.png" path-detail="documentation/insta-clone-pt5-20.png" alt="image" %}

위 사진을 보면 긴 글의 경우 업로더 정보와 내용이 딱히 column의 구분없이 함께 나오는 것으로 보인다. 즉 하나의 col-12안에 두 정보를 하나의 string으로 연결하여 넣어주면 될거 같다.

물론 업로더는 이경우 bold처리하고 스페이스 하나가 들어가야 한다.

리팩터링을 시작해보자.

```django
{%raw%}
..전략
    {# 4. 4번째 row: 업로더 정보가 한번 더 나타나고 내용이 들어간다. #}
    <div class="description-container row">
        <div class="content-container col-12">
            <b>{{ instagram.author.username }}</b> &nbsp;
            {{ instagram.content }}
        </div>
    </div>
..후략
{%endraw%}
```
- 기존의 column들을 삭제하고 `col-12`로 대체한다.
- author는 볼드처리하고 내용과 author사이에 스페이스바를 하나 삽입해준다.


테스팅 시에는 **overflow가 잘 되는지 체크** 해봐야 하기 때문에 크롬에서 content를 늘려서 넣어줘본다.


{% include image.html path="documentation/insta-clone-pt5-21.png" path-detail="documentation/insta-clone-pt5-21.png" alt="image" %}

긴 문자열을 넣었더니 overflow가 일어난다. 이를 예방하기 위해 아예 부모 컨테이너에 `word-break `를 넣어버리자.

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
    /* Added! */
    word-wrap: break-word;
}
```

다시 새로고침해보자.


{% include image.html path="documentation/insta-clone-pt5-22.png" path-detail="documentation/insta-clone-pt5-22.png" alt="image" %}

잘 되는 것을 알 수 있다.

### 3.2. 6번째 row
1번에서 수정한 작성 시간정보가 들어있는 row를 살펴보자. 상용 인스타그램에서는 해당 정보가 그레이톤에 작은 글씨로 나타난다. 약간 공간도 띄워주자.

```css
.created-time {
    padding-top: 10px;
    padding-bottom: 10px;

    font-weight: lighter;
    font-size: small;
}
```

- font-weight과 size를 상대적으로 조절한다. 큰 프로젝트라면 lighter, small말고 정확한 수치를 넣는게 좋을 것이다.

새로고침 해보자.


{% include image.html path="documentation/insta-clone-pt5-23.png" path-detail="documentation/insta-clone-pt5-23.png" alt="image" %}

예뻐졌다.

### 3.3. 7번째 row
긴 텍스트를 입력했을 때 content가 삐져나왔던 것을 상기해보자. 댓글창에서도 그러지 않으리란 보장이 없다.

직관적으로 긴 텍스트를 댓글창에 입력하면 text창이 자동으로 확장되며 내가 쓴 글을 모두 육안으로 확인가능해야 할 것이다. 그러나 현재 우리의 구성에서는 그러지 않다.

고쳐보자. 제타위키에 [굉장히 잘 정리된 글](https://zetawiki.com/wiki/HTML_textarea_%EC%9E%90%EB%8F%99_%EB%86%92%EC%9D%B4_%EC%A1%B0%EC%A0%88)이 있다.

```javascript
$(document).ready(function () {
    ...중략
    // ADDED!
    $("textarea").on('keydown keyup', function () {
        $(this).height(1).height($(this).prop('scrollHeight') + 12);
    });
}
```
- 어차피 textarea는 댓글창밖에 없으므로 모든 textarea에 적용해도 무방하다.

새로고침 해보자.


{% include image.html path="documentation/insta-clone-pt5-24.png" path-detail="documentation/insta-clone-pt5-24.png" alt="image" %}


잘된다.

## 4. 정리
- `settings.py`에서 `TIME_ZONE`을 세팅해주면 Django는 알아서 로컬시간대 기준으로 연산을 해준다.
- 이 경우 `django.utils`에 포함된 `timezone`을 이용하면 편하다.
- 일반적으로 DB에 저장되는 시간대는 `UTC`로 저장된다.
- DB에서 읽어온 객체의 시간을 이용하기 위해서는 위 `timezone`에 포함된 `localtime` 메소드를 이용해 변환해서 사용한다.
- 클릭이벤트는 jQuery를 이용하면 편하다.
- 버튼을 아이콘으로 바꿔주려면 button의 `background`를 none으로 해준다.
- textarea의 overflow를 잡기위해 `word-break`을 넣어준다.



다음 튜토리얼에서는 댓글시스템을 구현해본다. 이것만 완료되면 Instagram과 거의 비슷해진다!
