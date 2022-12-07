# Docker 실습
Docker에대한 실습정보 이다. <br/>
cli 환경에서 작업 할 수 있게 작업해야 한다.


```
git clone https://gitlab.com/yalco/practice-docker.git
```
> 해당 프로젝트를 clone 해야 한다.

<br/>

---

<br/>


## Docker cli use
첫번째 로그인과 이미지 다운로드 작업을 시작한다.

<br/>
<br/>



```
docker login
```
>로그인 명령어

<br>

```
docker run -it node
```
> 이미지가 없을경우 이미지를 다운로드 받으면서 컨테이너를 실행시킨다.

<br>

```
docker ps
```
> 실행중인 컨테이너를 확인

|CONTAINER ID|IMAGE|COMMAND|CREATED|STATUS|PORTS|NAMES|
|:---|:---|:---|:---|:---|:---|:---|
|36a177abc074|node|docker-entrypoint.s…|3 hours ago|Up 3 hours|0.0.0.0:3306->3306/tcp, 33060/tcp|stoic_euler|

<br>

```
docker exec -it stoic_euler bash
```
>해당 셀로 접속이 가능하다. (리눅스 기반이라는걸 알 수 있다.)

<br>
<br>

---

<br>
<br>

## 01. Dockerfile을 이용한 build : forntEnd 
실질적으로 돌릴 설정파일, 세션파일, 데이터파일, json 파일등을 작성해 새로운 이미지를 생성한다.

```
FROM node:12.18.4

# 이미지 생성 과정에서 실행할 명령어
RUN npm install -g http-server 

# 이미지 내에서 명령어를 실행할(현 위치로 잡을) 디렉토리 설정
WORKDIR /home/node/app

# 컨테이너 실행시 실행할 명령어
CMD ["http-server", "-p", "8080", "./public"]

# 이미지 생성 명령어 (현 파일과 같은 디렉토리에서)
# docker build -t {이미지명} .

# 컨테이너 생성 & 실행 명령어
# docker run --name {컨테이너명} -v $(pwd):/home/node/app -p 8080:8080 {이미지명}
```
> /fonrend/Dockerfile 이다. 해당 파일을 이용해 전역설정, 제어할 파일 및 실행 명령어가 존재한다.

<br>

```
docker build -t fontend -img .
```
> -t 뒤에 [원하는 이미지명] Dockerfile로의 상대경로 실행시 새로운 이미지가 생성됩니다.

<br>

```
docker images
```
> 도커 이미지들이 나타난다.

<br>

```
docker run --name frontend-con -v /home/node/app -p 8080:8080 frontend-img
```
>docker run --name [생성할 컨테이너 명] -v [이 폴더의 내용을 저장할 폴더] -p [내선 포트:컨테이너 포트] [실행할 이미지] <br>
-v : volunm '컨테이너와 특정 폴더를 공유한다. <br>
-p : port 내선번호를 컨테이너의 것과 연결한다.

<br>

---

<br>

## 02. Dockerfile을 이용한 build : database 
ENV 명령어로 생성될 컨테이너 안의 환경변수를  미리 지정한다
```
FROM mysql:5.7

# 이미지 환경변수들 세팅
# 실전에서는 비밀번호 등을 이곳에 입력하지 말 것!
# 서버의 환경변수 등을 활용하세요.

ENV MYSQL_USER mysql_user
ENV MYSQL_PASSWORD mysql_password
ENV MYSQL_ROOT_PASSWORD mysql_root_password
ENV MYSQL_DATABASE visitlog

# 도커환경에서 컨테이너 생성시 스크립트를 실행하는 폴더로
# 미리 작성된 스크립트들을 이동
COPY ./scripts/ /docker-entrypoint-initdb.d/

# 이미지 빌드 명령어 (현 파일과 같은 디렉토리에서)
# docker build -t {이미지명} .

# 실행 명령어 (터미널에 로그 찍히는 것 보기)
# docker run --name {컨테이너명} -it -p 3306:3306 {이미지명}

# 실행 명령어 (데몬으로 실행)
# docker run --name {컨테이너명} -p 3306:3306 -d {이미지명}
```

<br>

```
docker build -t database-img .
```
>이번에도 마찬가지로 dockerFile 을 build 해서 새로운 이미지를 생성하자

<br>

```
docker run --name database-con -it -p 3306:3306 database-img
```
>이미지를 실행시켜 컨테이너를 만들어준다.

<br>

```
docker logs -f database-con
```
>컨테이너의 로그를 확인 할 수 있다.

<br>

---

<br>

## 03. Dockerfile을 이용한 build : backend 
이제 backend 서버만 생성해주면 된다. 하지만 문제가 되는 부분은 같은 네트워크를 사용하고 있지 않다는거다. 그럴때 docker-compose.yml 이 동작한다.

```
version: '3'
services:
  database:
    # Dockerfile이 있는 위치
    build: ./database
    # 내부에서 개방할 포트 : 외부에서 접근할 포트
    ports:
      - "3306:3306"
  backend:
    build: ./backend
    # 연결할 외부 디렉토리 : 컨테이너 내 디렉토리
    volumes:
      - ./backend:/usr/src/app
    ports:
      - "5000:5000"
    # 환경변수 설정
    environment: 
      - DBHOST=database
  frontend:
    build: ./frontend
    # 연결할 외부 디렉토리 : 컨테이너 내 디렉토리
    volumes:
      - ./frontend:/home/node/app
    ports:
      - "8080:8080"
```