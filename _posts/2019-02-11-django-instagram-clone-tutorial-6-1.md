---
layout: post
title: "Django를 이용한 Instagram clone 만들기 Part.6 (1)"
description: "Infinite scroll 추가하기"
tags: [tutorial, django, python]

---

# Django를 이용한 Instagram 클론 만들기 Part.6 (1)

1. 아키텍쳐 설계 및 Django model 구현
2. Django view 구현 (CBV)
3. View develop하기
4. Django template 기본 구현
5. Django template 완전 구현
    1. 상용 서비스와 같은 디자인 골격 잡기
    2. 수정/삭제 Dropdown 및 댓글창 예쁘게 만들기
    3. 기타 짜잘한 디자인 완성시키기
    4. AJAX를 이용한 기본적인 댓글 시스템 구현
6. 좀더 멋있는 기능 추가하기
    1. Infinite scroll 구현하기 **(current)**
    2. 사진 Carousel 붙이기

<hr>

## 1. Pagination
현재 우리가 만든 서비스는 페이지네이션에 대한 구현이 전혀 없다. 따라서 데이터가 많아지면 많아질수록 퍼포먼스가 선형적으로 감소할 것이다.

이를테면 서비스 이용자 10명이 각각 100개의 인스타그램을 올렸을 때, 총 1000개의 인스타그램 객체가 한꺼번에 로딩되는데, 하나당 500kb라 가정해도 500mb의 용량을 로드해야 한다.

이는 웹에서도 엄청난 용량이고, 모바일은 사실상 보지 말라고 하는 것과 같다. 이상황을 타개하기 위해 pagination을 사용한다.



#### Django default pagination
Django는 pagination이 직관적이어서 따라하기 어렵지 않다. Django를 이용한 구현은 [공식문서](https://docs.djangoproject.com/en/2.1/topics/pagination/)에 잘 설명되어있다. 이미 우리는 ListView를 구현할 때 `paginate_by`를 통해 20개씩 불러오겠다고 선언한 바 있다. 이제 template에서 나머지를 구현해보자.

`feed_list.html`에 다음을 추가한다.

```django
{%raw %}
...전략
    {% if is_paginated %}
        <div class="pagination">
            <span class="page-links">
                {% if page_obj.has_previous %}
                    <a href="/instagrams/?page={{ page_obj.previous_page_number }}">previous</a>
                {% endif %}
                <span class="page-current">
                    Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}.
                </span>
                {% if page_obj.has_next %}
                    <a href="/instagrams/?page={{ page_obj.next_page_number }}">next</a>
                {% endif %}
            </span>
        </div>
    {% endif %}
 {% endblock %}
 ...후략
{%endraw %}
```
- endblock 바로 위, 즉 body태그가 닫히기 바로 전에 해당 코드를 작성한다.
- 우리는 인스타그램의 `ListView`를 '[루트 url]/instagrams/' 부르고 있으므로 (urls.py 참조) href의 소스는 그 뒤에 쿼리스트링을 붙인 형태이다.


이제 `InstagramListView`에서 `paginate_by`를 1로 설정하고 새로고침을 해보자.

{% include image.html path="documentation/insta-clone-pt6-1.png" path-detail="documentation/insta-clone-pt6-1.png" alt="image" %}

인스타그램 객체가 한개씩 보이고, 하단에 새로 생긴 페이지 버튼을 누르면 다음 페이지로 이동한다. 그렇다면 n개씩 불러온 후에 일정 위치에 이르면 다음 n개를 불러오면 무한스크롤이 되지 않을까?

이 원리를 응용해서 무한 스크롤을 구현해보자.


## 2. Waypoint
무한스크롤을 구현하기 위해  [Waypoint](http://imakewebthings.com/waypoints/) jquery 라이브러리를 사용한다. 사이트에서 라이브러리를 다운로드받아 static에 추가해도 되지만 귀찮으므로 CDN으로 대체한다.

```django
{%raw %}
...전략
{% block javascript %}
    <script type="text/javascript" src="{% static 'instagrams/js/feed_list_js.js' %}"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/waypoints/4.0.1/jquery.waypoints.js"></script>
{% endblock %}

{%endraw%}
```

새로고침한 후 개발자도구의 Networks에서 `jquery.waypoints.js`가 제대로 불러와지는지 확인한다.


