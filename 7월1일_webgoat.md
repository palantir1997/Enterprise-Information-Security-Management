# 🛡️ WebGoat 실습 정리 + ModSecurity(WAF) 구축 실습

> 오늘은 **SliTaz 환경에서 WebGoat 서버를 직접 띄우고** 대표적인 웹 취약점 8종을 실습했고, 후반부에는 **ModSecurity를 설치해 실제로 공격을 차단하는 WAF(Web Application Firewall)**를 구성해봤습니다. 공격자 관점에서 뚫어보던 것과 반대로, **방어자 관점에서 규칙을 작성해 막아보는** 실습이라 접근 방식이 달랐습니다.

---

## 📑 목차

1. [환경 설정 - SliTaz 네트워크 & WebGoat 실행](#환경-설정---slitaz-네트워크--webgoat-실행)
2. [WebGoat 실습 요약표](#webgoat-실습-요약표)
3. [WebGoat 실습 상세](#webgoat-실습-상세)
4. [ModSecurity(WAF) 구축](#modsecurity-waf-구축)
5. [오늘의 회고](#오늘의-회고)

---

## 환경 설정 - SliTaz 네트워크 & WebGoat 실행

```bash
vi /etc/network.conf
```

SliTaz의 `eth0` IP를 수동(static)으로 설정합니다.

```
static - yes
ip - <할당할 IP>
gateway - <게이트웨이 주소>
```

```bash
/etc/init.d/network.sh restart
```

재시작하면 설정한 IP로 인터페이스가 올라옵니다. IP가 잡힌 걸 확인한 뒤 WebGoat를 실행합니다.

```bash
ls
./start-webgoat.sh
```

브라우저에서 `http://172.16.11.219:8080/WebGoat/` 접속 후 `guest / guest`로 로그인.

---

## WebGoat 실습 요약표

| # | 실습 | 핵심 기법 | 한 줄 개념 |
|---|------|-----------|------------|
| 1 | Bypass a Path Access Control Scheme | Burp + `../` 경로조작 | 파일 파라미터에 상위 경로를 넣어 임의 파일(`/etc/shadow`) 열람 |
| 2 | DOM-Based XSS (AJAX Security) | `<img src=...>` 태그 우회 | `<script>` 필터는 걸리지만 `<img>` 태그 속성 삽입은 통과 |
| 3 | Multi Level Login (Password Strength) | Burp에서 파라미터 변조 | 2차 인증 단계에서 세션의 user 값을 바꿔 다른 계정으로 인증 우회 |
| 4 | Code Quality | 페이지 소스보기 | 클라이언트에 노출된 소스 코드/주석에 계정정보 하드코딩 |
| 5 | Phishing with XSS | `<iframe>` 삽입 | 가짜 로그인 폼을 iframe으로 심어 피싱 페이지 구성 |
| 6 | Injection Flaws | Command Injection / SQL Injection | OS 명령 체이닝(`;id;`), Numeric/String SQLi로 인증 우회 |
| 7 | Challenge | - | 미해결 (난이도 높음) |
| 8 | ModSecurity WAF 구축 | `mod_security`, CRS | 공격 실습에서 방어 실습으로 전환 |

---

## WebGoat 실습 상세

### 1. Bypass a Path Access Control Scheme

```
file 파라미터 = ../../../../../../../../../etc/passwd
file 파라미터 = ../../../../../../../../../etc/shadow
```

**왜 되는가**: 파일을 불러오는 파라미터가 사용자 입력을 검증 없이 그대로 파일 경로 조합에 사용하기 때문에, `../`를 충분히 반복하면 웹 애플리케이션이 원래 허용하려던 디렉토리 범위를 벗어나 시스템 루트까지 거슬러 올라갈 수 있습니다. Burp로 요청을 잡아서 파라미터 값만 바꿔주면 되므로 실습 난이도는 낮은 편입니다.

**후속 작업**: `/etc/shadow`에서 얻은 해시를 `pass.txt`로 저장한 뒤,

```bash
sudo john pass.txt
```

John the Ripper로 크래킹을 시도했습니다. 경로 조작 취약점 하나가 **파일 열람 → 크리덴셜 획득 → 오프라인 크래킹**으로 이어지는 전형적인 체인을 보여주는 실습이었습니다.

---

### 2. DOM-Based XSS (AJAX Security)

**증상**: 입력창에 `<script>`를 넣으면 필터에 걸려 문자가 `.`으로 치환됨.

**우회**: `<script>` 태그 대신, 이미지 요소의 `src` 속성 안에 페이로드를 넣는 방식으로 우회했습니다. 실습에서는 실제 이미지 주소를 복사해

```html
<img src="복사한 이미지 주소">
```

형태로 넣었더니 필터를 통과하면서 이미지가 정상적으로 렌더링되는 걸 확인했습니다.

**왜 되는가**: 필터가 `<script>` 키워드 자체만 블랙리스트로 잡고 있어서, **DOM에 삽입 가능한 다른 태그/속성(`<img>`, `onerror` 등)을 통한 이벤트 기반 실행**은 걸러내지 못하는 전형적인 블랙리스트 우회 패턴입니다.

---

### 3. Multi Level Login (Password Strength)

**절차**:
1. 1차 아이디/비밀번호로 로그인 시도, Burp로 요청 캡처.
2. 1차 인증을 통과한 상태에서 2차 인증 단계로 넘어갈 때, 요청에 담긴 `user` 파라미터 값을 다른 계정으로 변조.

**왜 되는가**: 서버가 2차 인증 단계에서 "이 사용자가 1차 인증을 통과한 바로 그 사용자인가"를 세션 서버 측에서 재검증하지 않고, **요청에 실려온 파라미터 값을 그대로 신뢰**하기 때문입니다. 그 결과 2차 인증 자체는 유효하게 통과하면서도 **인증 주체(계정)만 슬쩍 바꿔치기**할 수 있습니다.

**왜 중요한가**: 다단계 인증이라도 각 단계가 독립적으로 "누구에 대한 검증인지"를 서버 세션에 고정해두지 않으면 오히려 우회 지점이 늘어날 수 있다는 걸 보여줍니다.

---

### 4. Code Quality

브라우저의 **페이지 소스 보기(View Selection Source)** 만으로 HTML 주석이나 숨겨진 필드에 하드코딩된 계정정보를 확인했습니다. 별도의 도구 없이도 "클라이언트에 내려가는 모든 것은 노출된다"는 원칙을 보여주는 가장 단순한 실습이었습니다.

---

### 5. Phishing with XSS

```html
<iframe frameborder="0" height="500" width="500" src="http://<내_IP>:8000/exam.html"></iframe>
```

미리 만들어둔 가짜 로그인 폼(`exam.html`)을 iframe으로 삽입해서, 정상 페이지처럼 보이는 화면 안에 피싱 폼을 끼워넣는 실습입니다. XSS 취약점이 있는 입력 지점에 이 iframe 태그를 넣으면, 페이지를 보는 사용자는 진짜 사이트 안에 있다고 착각한 채로 가짜 폼에 자격증명을 입력하게 됩니다.

**왜 중요한가**: XSS는 스크립트 실행 자체보다, **이렇게 신뢰받는 도메인 위에서 피싱 콘텐츠를 노출시킬 수 있다는 점**이 더 위험할 때가 많습니다. 사용자는 URL 창의 도메인만 보고 신뢰하기 때문입니다.

---

### 6. Injection Flaws

**Command Injection**

```
파일 경로 파라미터 = ";id;"
```

또는 URL 인코딩:
```
%22%26 /etc/passwd%26%22
```

세미콜론(`;`)으로 명령어를 체이닝해서 `id` 명령의 실행 결과(uid/gid)가 그대로 응답에 출력되는 걸 확인했습니다. 서버가 파일명 파라미터를 셸 명령어 조합에 그대로 사용하고 있다는 뜻입니다.

**SQL Injection - Numeric**

```bash
sqlmap --cookie="<쿠키값>" -u "<브루트포스로 잡은 URL>" -p station
```

→ 이번 시도에서는 자동화 툴로는 성공하지 못했습니다 (파라미터/쿠키 조합이 맞지 않았던 것으로 추정, 추후 재시도 필요).

**SQL Injection - String**

```
Smith' or 1=1 --
```

→ 로그인/검색 쿼리에 삽입하니 조건절이 항상 참이 되면서 인증 우회에 성공했습니다.

**왜 되는가**: 두 경우 모두 사용자 입력이 **명령어 문자열 또는 쿼리 문자열에 검증/이스케이프 없이 그대로 이어붙여지기(concatenation)** 때문입니다. Command Injection은 셸 메타문자(`;`, `&`)로 명령을 체이닝하고, SQLi는 쿼리 문법 자체를 깨서 논리 조건을 조작합니다.

---

### 7. Challenge

이번에는 풀지 못했습니다. 앞선 실습들처럼 단일 취약점이 아니라 여러 단계를 조합해야 하는 것으로 보여서, 별도 시간을 두고 다시 도전할 예정입니다.

---

## ModSecurity(WAF) 구축

지금까지는 공격자 입장이었다면, 이번엔 반대로 **WAF 규칙을 직접 작성해서 공격을 막아보는** 실습입니다.

### 설치

```bash
sudo apt install libapache2-mod-security
sudo a2enmod security2
ls /etc/modsecurity
```

### 기본 설정 켜기

```bash
sudo vi /etc/modsecurity/modsecurity.conf
```

기본적으로 `SecRuleEngine`이 `DetectionOnly`(로그만 남기고 차단은 안 함)로 되어있는 경우가 많아서, 실제로 **차단까지 하려면 `On`으로 변경**해야 합니다. (실습 메모에서 "abc C 지우고 D 넣기"라고 되어 있는 부분은 `DetectionOnly` → `On`으로 바꾼 걸 축약해 적어둔 것으로 보입니다.)

```bash
sudo systemctl restart apache2
```

### OWASP CRS(Core Rule Set) 개념

직접 규칙을 하나하나 작성하는 대신, OWASP에서 미리 만들어둔 **강력한 공용 규칙 세트(Core Rule Set)** 를 가져다 쓰는 방식도 있습니다. ModSecurity 규칙의 기본 문법 구조는 다음과 같습니다.

```
지시어	변수	연산자	동작
SecRule	Variables	Operator	Actions
```

**예시** - 주소(URI)에 `admin`이라는 단어가 들어가면 접속을 차단(403):

```apache
SecRule REQUEST_URI "@rx admin" "id:1000001,phase:1,deny,status:403,msg:'Block Admin Access'"
```

- 지시어: `SecRule`
- 변수: `REQUEST_URI` (요청 경로를 검사)
- 연산자: `@rx admin` (정규표현식으로 `admin` 패턴 매칭 여부 확인)
- 동작: 규칙 ID `1000001`, `phase:1`(헤더 단계)에서 검사, 매칭 시 `deny` + `403` 응답, 로그에 메시지 기록

### 실제로 작성해본 규칙들

`sudo vi /etc/modsecurity/modsecurity.conf` 맨 아래에 아래 규칙들을 추가했습니다.

```apache
SecStatusEngine Off

# 1) 특정 경로 차단 테스트
SecRule REQUEST_URI "@contains admin" "id:100001,phase:1,log,deny,msg:'admin is not access - Web Hack'"

# 2) XSS 방어
SecRule ARGS "@rx <script" "id:100002,phase:2,log,deny,status:403,msg:'XSS Attacking'"

# 3) SQL Injection 방어
SecRule ARGS "@contains UNION SELECT" "id:100003,phase:2,deny,log,status:403,msg:'SQL Injection'"
```

문법 검증 후 적용:

```bash
sudo apachectl configtest
sudo tail -f /var/log/apache2/modsec_audit.log
```

**테스트 1 - 경로 차단**: `/login-admin` 경로로 접속 시도 → `admin` 문자열이 `REQUEST_URI`에 포함되어 있어 즉시 403으로 차단됨.

**테스트 2 - XSS 차단**:
```
http://172.16.11.210/id=%3Cscript%3Ealert(document.cookie)%3C/script%3E
```
→ `<script` 패턴이 `ARGS`(요청 파라미터)에서 매칭되어 차단됨.

**테스트 3 - SQLi 차단**:
```
http://172.16.11.210/id='union select test from testing
```
→ `UNION SELECT` 패턴이 매칭되어 차단됨.

**왜 되는가**: ModSecurity는 Apache 요청 처리 파이프라인에 **훅을 걸어, 각 단계(phase)마다 정의된 규칙과 요청 내용을 대조**합니다. `REQUEST_URI`, `ARGS` 같은 변수로 요청의 특정 부분을 지정하고, `@rx`(정규식) / `@contains`(문자열 포함) 같은 연산자로 패턴을 검사한 뒤, 매칭되면 `deny` 액션으로 즉시 응답을 차단합니다. 직접 규칙을 짜보니, 오늘 오전까지 실습했던 XSS/SQLi 페이로드들이 **바로 그 패턴 자체가 탐지 대상**이 된다는 걸 체감할 수 있었습니다.

**왜 중요한가**: 커스텀 규칙은 빠르게 특정 패턴을 막을 수 있지만, 블랙리스트 방식이라 **표현을 조금만 바꿔도(인코딩, 대소문자, 공백 삽입 등) 우회될 여지**가 있습니다. 그래서 실무에서는 직접 짠 규칙보다 **OWASP CRS처럼 지속적으로 업데이트되는 공용 규칙 세트**를 기반으로 삼는 경우가 많습니다.

```bash
ls /usr/share/modsecurity-crs/rules/
ls /etc/modsecurity/crs
```

두 경로에서 CRS 규칙 파일 목록을 확인해뒀고, 다음 실습에서는 이 규칙들을 참조해서 **직접 짠 규칙과 CRS 규칙을 비교/보완**해볼 예정입니다.

---

## 오늘의 회고

- 오전(WebGoat)까지는 **뚫는 입장**, 오후(ModSecurity)에는 **막는 입장**이라 같은 페이로드(`<script>`, `UNION SELECT` 등)를 공격자/방어자 양쪽 시각에서 모두 다뤄본 게 특히 도움이 됐습니다.
- 다단계 인증 우회(3번)처럼, **인증 로직이 여러 단계로 나뉘어 있어도 각 단계가 서로를 검증하지 않으면 의미가 없다**는 걸 실습으로 확인한 게 인상 깊었습니다.
- sqlmap 자동화가 실패했던 부분(6번 Numeric SQLi)과 Challenge(7번)는 다음에 파라미터/쿠키 값을 다시 점검하며 재도전할 계획입니다.
- ModSecurity 커스텀 규칙은 만들기는 쉽지만 우회도 쉬운 편이라, 다음 단계로 **OWASP CRS 규칙 구조를 직접 뜯어보는 것**을 다음 학습 목표로 잡았습니다.
