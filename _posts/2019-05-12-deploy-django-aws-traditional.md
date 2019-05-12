---
layout: post
title: "간단한 Django 어플리케이션 AWS에 배포하기"
description: "전통적인 방법으로 Django 어플리케이션 EC2 인스턴스에 배포하기 "
tags: [aws, devops, django, tutorial]

---

# 목표 정의
간단한 Django 어플리케이션을 AWS EC2 인스턴스를 이용하여 일반 사용자들이 웹사이트를 이용 가능하도록 배포한다. 가장 기본적이고 최소한의 방법만을 사용한다. 따라서 Docker 및 ECS 등은 이용하지 않는다(차후에 포스팅)

## 기술 스택
사용할 기술 스택은 다음과 같다.

1. git
2. nginx
3. gunicorn3
4. EC2 (Ubuntu 18)
5. Django 2.1

## 전제 조건
- AWS 계정이 있고 잘 만들어진(inbound, outbound규칙 등이 잘 세팅된) EC2 인스턴스가 존재한다.
- github등의 리모트 저장소에 소스코드가 존재하여 배포 브랜치의 이름은 `production`이다.
- 가상환경은 구성하지 않는다.
- DB는 Postgres를 이용하며 웹, DB 모두 단일 서버에 통합 설치한다.


# EC2

## Source 클론
EC2 인스턴스에 접속하여 소스코드를 클론받는다. 통상적으로 접속 명령어는 다음과 같을 것이다.

```bash
ssh -i [your-ec2-keypair].pem ubuntu@[your-ec2-ip-addr]
```

AWS EC2 인스턴스를 생성했을 시에는 ubuntu 사용자가 기본으로 설정되어 있고, /home/ubuntu/가 루트 디렉토리로 지정되어 있다. 개인적으로 루트 디렉토리를 바로 사용하기보다는, `data/`디렉토리를 만들고 그곳을 소스코드의 root으로 사용하는 걸 좋아한다. 디렉토리에서 `ll`등의 명령어를 사용하는 경우가 많은데, 루트 디렉토리에 기본으로 포함된 필요없는 숨김파일을 매번 보고싶지 않기 때문이다.

```bash
mkdir data && cd data
```

이제 git clone을 통해 djangotest라는 폴더를 만들고 그 안에 소스코드를 복사해온다.

```bash
git clone [git-url] djangotest && cd djangotest
```

## WSGI 확인
django의 경우 처음 프로젝트를 생성하면 프로젝트 이름과 동일한 이름을 가진 폴더가 자동으로 생성된다. 이 폴더 안에는 django가 구동하기 위한 중요한 파일들이 들어있다.

이 중, 배포과정에서는 `wsgi.py`가 중요하다. 이 파일은 `WSGI 포로토콜`을 구현한 웹서버(gunicorn 등)과 Python web framework(django, flask 등)간의 연결을 위한 일종의 엔트리포인트라고 보면 된다.

해당 폴더에 `wsgi.py`가 존재하고 내용도 정확한지 확인한다.


## Python 확인
우리는 가상환경을 사용하지 않으므로 python 버전 충돌 문제를 반드시 확인해야 한다. Ubuntu는 18버전부터는 Python3를 기본으로 내장하고 있다. 확인해보자.

```bash
# Python 2.7.15rc1
python -V

# Python 3.6.7
python3 -V
```

기본으로 내장하고 있다 뿐이지 python alias는 여전히 python2를 가르키고 있음을 알 수 있다.

그냥 가상환경을 세팅하면 참 편하겠지만 일단 가상환경을 사용하지 않으므로 다음 두가지 옵션 중 하나를 선택해야 한다.

1. python 관련 명령어를 실행할 시 python3를 명시적으로 사용한다.
2. /usr/bin 디렉토리의 python symlink를 python3로 교체한다.

개인적으로 옵션1을 권장한다. 현재 배포하는 Django앱은 상관없겠지만, python 및 python3를 명시적으로 구분하는 다른 시스템 상의 디펜던시가 존재할 수도 있기 때문이다.

이제 `pip3`도 설치해주자.

```bash
sudo apt install python3-pip
```

서버작업은 항상 귀찮음보다 조심스러움이 우선해야 한다고 생각한다. 앞으로 python3 및 pip3를 항상 생각하면서 작업한다.


이제 웹서버를 설치하자.

# Gunicorn
Django는 `./maange.py runserver` 커맨드를 통해 경량 웹서버를 로컬머신에 구동하는 기능을 제공한다. [공식문서](https://docs.djangoproject.com/en/1.8/ref/django-admin/#runserver-port-or-address-port)에 따르면 이 기능은 보안 및 성능면에서 배포용으로 적합하지 않다. 따라서 우리는 배포에 최적화된 웹서버인 gunicorn을 통해 배포할 것이다.


## Gunicorn 설치
EC2 인스턴스에 접속하여 gunicorn을 설치한다.

```bash
pip3 install gunicorn
```

> `apt-get`을 이용한 gunicorn 설치 시 python3와 호환되지 않는 문제가 있었다. 이 경우 `sudo apt-get install gunicorn3` 를 통해 `gunicorn3`를 설치해야 한다.

## Gunicorn 실행
`gunicorn`은 `nginx`가 프록시하여 로컬호스트로 던져주는 request들을 받아 처리하게 된다. 컨벤션에 따라 8000번 포트를 사용하도록 하자. 따라서 `gunicorn`서비스는 `127.0.0.1:8000`에 바인딩되어야 한다.


```bash
gunicorn [wsgi.py] --daemon --bind 127.0.0.1:8000 --reload
```
- gunicorn이 바인딩할 wsgi.py 파일(정확히는 application callable)을 명시적으로 지정한다.
- 소스코드의 변화에 따라 worker들을 다시 생성하고 싶다면 `--reload`옵션을 준다. runserver와 비슷하다고 생각하면 된다.
- daemon 모드로 실행시키고 싶은 경우 `--daemon`옵션을 준다. gunicorn이 실행되고 터미널을 바로 사용가능하다.
- workers 갯수는 `--workers` 옵션으로 지정가능하다.
- gunicorn3를 설치했다면 gunicorn3 명령어로 실행한다.

gunicorn 프로세스 및 포트 바인딩이 정확히 잘 되었는지 확인한다.

```bash
ps -ax | grep gunicorn
sudo lsof -i -P -n | grep gunicorn
```


# Nginx

## Nginx 설치
홈페이지에 나와있는대로 잘 설치해준다.

```bash
sudo apt-get update
sudo apt-get install nginx
```

## settings.py 확인
nginx 설정을 하기 위해서는 현재 Django가 static file들을 어디에 저장하는지, static file을 요청하는 request는 어떻게 던져주는지 알아야 한다. 프로젝트 루트의 `settings.py`를 확인한다.

> 만일 개발환경, 배포환경에서 서로 다른 settings파일을 사용하고 있는 경우 `wsgi.py`에서 로딩하고 있는 settings파일을 확인해야한다.

일반적으로 Static 파일 세팅은 다음과 같이 되어있을 것이다.

```python
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
```
- STATIC_ROOT은 `./manage.py collectstatic`명령어를 실행하여 모든 static file들을 모아줄 장소를 지정한다. 현재는 프로젝트 디렉토리의 staticfiles 폴더로 지정되어 있다.
- STATIC_URL은 STATIC_ROOT에 모인 파일을 찾고자 할 때 사용할 URL을 지정한다. 예컨대 test app의 index.js를 불러오고자 한다면 (setting하기 나름이지만) http://[domain or IP]/static/test/index.js 로 static request이 생성될 것이다.


## conf 파일 설정
Nginx는 사용자들이 요청한 request중 dynamic request만 따로 프록시하여 gunicorn에게 던져준다. static request들은 알아서 Nginx가 서빙한다. 따라서 우리는 conf 파일에 2가지 설정을 해야 한다.

1. Static request인 경우: Django의 `STATIC_ROOT`에 있는 파일들을 서빙하도록 설정한다.
2. Dynamic request인 경우: proxy하여 `localhost:8000`으로 라우팅 시킨다.

`/etc/nginx/sites-available/`디렉토리에 `django-test` 파일을 만든다. 해당 디렉토리는 root유저 소유이기에 sudo로 생성해야 한다.

```bash
cd /etc/nginx/sites-available
sudo touch django-test
```
이제 `settings.py`에서 정보들에 맞추어 설정을 추가한다. `django-test`파일을 열어 다음을 입력한다.

```bash
server {
  listen 80;
  server_name [domain or IP];

  # Rest framework을 사용할 경우
  # https://stackoverflow.com/a/27616305
  # location /static/rest_framework {
  #   alias /home/ubuntu/.local/lib/python3.6/site-packages/rest_framework/static/rest_framework/;
  # }

  location /static {
    alias /home/ubuntu/data/djangotest/staticfiles;
  }

  location / {
    include proxy_params;
    proxy_pass http://127.0.0.1:8000;
  }

}
```
- STATIC_URL이 /static/ 이므로 해당 URL을 통한 static request은 `staticfiles` 디렉토리 내부에 존재하는 static file들을 탐색해야 한다.
- 그 외의 루트 URL을 포함한 dynamic request들은 모두 localhost에서 돌고 있는 `gunicorn`으로 프록시해서 던져준다.
- 만일 Django REST Framework를 사용중이라면 명시적으로 rest_framework에 존재하는 staticfile을 가리키도록 추가해준다.

## Sites enable
새롭게 추가된 conf 파일을 symlink로 연결해준다.

```bash
sudo ln -s /etc/nginx/sites-available/django-test /etc/nginx/sites-enabled/
```

문법이 제대로 되었는지 확인하고 nginx 서비스를 재시작한다.

```bash
sudo nginx -t
sudo service nginx restart
```

# 배포 확인
EC2 instance의 Public DNS로 이동하여 사이트가 잘 나오는지 확인한다. 물론 SSL이 적용되지 않은 상태이므로 크롬으로 접속한다면 브라우저가 불평을 할 것이다. 무시하고 잘 뜨는지 확인한다.

 `/var/logs/nginx/` 디렉토리로 이동하여 `access.log`가 잘 갱신되는지 확인한다.


