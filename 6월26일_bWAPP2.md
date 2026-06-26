# 🛡️ bWAPP 취약점 학습 정리 (A3 · A4) + Juice Shop 환경 구축

> 이전 README(A1 Injection, A2 Broken Authentication)에 이어, A3(Cross-Site Scripting)와 A4(Insecure/Direct Object Reference)를 정리합니다. 후반부에는 다음 실습 대상인 **OWASP Juice Shop** 환경 구축 과정과, 본격적인 공격 전에 진행한 **nmap 사전 정찰**을 정리했습니다.

---

## 📑 목차

1. [A3 요약표](#a3-요약표)
2. [A3 - Cross-Site Scripting (XSS)](#a3---cross-site-scripting-xss)
3. [A4 요약표](#a4-요약표)
4. [A4 - Insecure / Direct Object Reference (DOR)](#a4---insecure--direct-object-reference-dor)
5. [Juice Shop 실습 환경 구축 (Docker)](#juice-shop-실습-환경-구축-docker)
6. [사전 정찰 - nmap](#사전-정찰---nmap)

---

## A3 요약표

| # | 취약점 | 핵심 페이로드 | 한 줄 개념 |
|---|--------|----------------|------------|
| 1 | XSS - Reflected (GET) | `<h1>hack</h1>` | GET 파라미터가 그대로 HTML로 출력 |
| 2 | XSS - Reflected (JSON) | `</script><script>(document.cookie)</script><script>` | 기존 `<script>` 블록을 닫고 새 스크립트를 끼워 넣음 |
| 3 | XSS - Reflected (AJAX/JSON) | (확인 필요) | AJAX 응답값이 DOM에 그대로 삽입됨 |
| 4 | XSS - Reflected (XML) | `document.location.href='';alert(document.cookie);''"` | onerror 우회 실패 후 location 이동으로 재시도 |
| 5 | XSS - Reflected (Custom Header) | Burp에 `bWAPP: test` 헤더 추가 | 특정 커스텀 헤더가 있어야 통과되는 검증 로직 |
| 6 | XSS - Reflected (eval) | `?date=alert(document.cookie)` | 입력값을 `eval()`로 그대로 실행 |
| 7 | XSS - Reflected (href) | `<script>alert("hacker")</script>` | Vote 필드 값이 href 속성에 그대로 삽입 |
| 8 | XSS - Reflected (Login Form) | `'<script>document.cookie</script>` | 로그인 실패 메시지에 입력값이 그대로 반영 |
| 9 | phpMyAdmin 무인증 접근 | `test[a@http://172.16.11.216@]` 클릭 | (검증 필요) Referer/URL 파싱 우회 또는 기본 계정 노출 추정 |
| 10 | PHP_SELF XSS | `<script>(document.cookie)</script>` | `$_SERVER['PHP_SELF']`가 form action에 그대로 출력 |
| 11 | XSS - Reflected (Referer) | Referer 헤더에 `<script>alert(document.cookie)</script>` | Referer 헤더 값이 페이지에 그대로 반영 |
| 12 | XSS - Reflected (User-Agent) | User-Agent 헤더에 동일 페이로드 | User-Agent 헤더 값이 페이지에 그대로 반영 |
| 13 | XSS - Stored (Blog) | (A2와 동일 패턴) | DB 저장 후 모든 방문자에게 노출 |
| 14 | XSS - Stored (Change Secret) | `'<script>document.cookie</script>` | Secret 값이 저장된 뒤 로그인 시 렌더링되며 실행 |
| 15 | XSS - Stored (Cookies) | `?genre=<script>...</script>` | genre 파라미터가 쿠키 값으로 저장 → 재출력 시 실행 |
| 16 | SQLite (다음 실습 예정) | - | bWAPP 내 SQLite 기반 챌린지로 이동 |

---

## A3 - Cross-Site Scripting (XSS)

### 1. XSS - Reflected (GET)

**개념**: 가장 기본적인 형태로, GET 파라미터 값이 검증/인코딩 없이 그대로 HTML로 출력됩니다.

**핵심 페이로드**
```
<h1>hack</h1>
```

**결과**: 입력한 `<h1>` 태그가 실제 HTML 태그로 인식되어 큰 글씨(헤딩)로 렌더링됨 → 서버가 사용자 입력을 "글자"가 아니라 "코드"로 받아들인다는 증거.

---

### 2. XSS - Reflected (JSON)

**개념**: 응답이 AJAX/JSON 기반으로 내려오면서, 화면에 들어가는 값이 이미 `<script>...</script>` 블록 안에 있는 경우가 있습니다. 이때는 단순히 새로운 `<script>` 태그를 추가하는 게 아니라, **먼저 현재 열려있는 스크립트 블록을 닫고(`</script>`) → 내 코드를 실행시킬 새 `<script>` 태그를 열고 → 뒤에 남은 원래 코드가 깨지지 않도록 다시 `<script>`로 맞춰주는** 3단 구조가 필요합니다.

**핵심 페이로드**
```html
</script><script>(document.cookie)</script><script>
```

**어떻게 발견했나**: F12 개발자 도구로 응답 소스를 보니 `<script>` 태그가 제대로 닫혀있지 않은 채로 사용자 입력값이 그 안에 그대로 들어가고 있는 걸 확인 → "지금 내가 있는 곳이 HTML 텍스트 영역이 아니라 자바스크립트 코드 영역이구나"를 알아차린 게 핵심.

**개념 정리**: 같은 XSS라도 "내 입력값이 어느 컨텍스트(HTML 본문 / 속성값 / 스크립트 내부)에 떨어지는가"에 따라 탈출 방법이 완전히 달라짐. 이 챌린지는 "스크립트 컨텍스트 탈출"의 대표 예시.

---

### 3. XSS - Reflected (AJAX/JSON)

**개념**: 비동기(AJAX)로 호출된 JSON 응답의 특정 필드 값이 검증 없이 DOM에 다시 꽂히는 경우의 XSS. #2와 유사한 계열이지만 별도 엔드포인트.


---

### 4. XSS - Reflected (XML)

**개념**: "Quote of the Day" 같이 XML 데이터를 기반으로 화면에 문구를 출력하는 기능에서 발생. 1차로 시도한 `onerror` 기반 페이로드가 바로 안 먹혀서, **위치를 옮겨 실행되도록 유도하는 방식**으로 재시도했습니다.

**1차 시도 (결과 없음)**
```html
<img src="X" onerror="alert(document.cookie)">
```

**2차 시도 (재확인용)**
```html
document.location.href='';alert(document.cookie);''"
```

**개념 정리**: XML 기반 응답은 필터링 방식이나 출력되는 위치(속성/텍스트노드)가 HTML과 다를 수 있어서, 한 가지 페이로드가 안 통하면 **다른 컨텍스트를 가정하고 재시도**하는 게 일반적인 XSS 공략 패턴입니다. (`onerror` 우회 → 직접 `alert` 호출 / `location` 이동으로 우회 시도)

---

### 5. XSS - Reflected (Custom Header)

**개념**: 서버가 특정 커스텀 HTTP 헤더(`bWAPP: ...`)가 존재해야만 해당 입력값을 처리/반영하는 구조. 즉, 일반 브라우저 요청만으로는 트리거가 안 되고 **헤더를 직접 조작할 수 있는 프록시 도구(Burp Suite)가 필수**입니다.

**공략 절차**
1. Burp Suite Proxy로 요청을 가로챈다(Intercept on).
2. 요청 헤더에 `bWAPP: test` 한 줄을 추가한다.
3. Forward → 해당 헤더가 있을 때만 취약한 로직이 동작하며 페이로드가 반영됨.

**왜 해킹이 되는가**: 브라우저는 보통 이런 임의 커스텀 헤더를 자동으로 안 보내기 때문에, "일반 사용자는 절대 못 건드리는 값"이라고 개발자가 안심하고 검증을 생략한 경우입니다. 하지만 공격자는 Burp 같은 프록시로 HTTP 요청 자체를 자유롭게 조작할 수 있으므로 이런 가정은 보안적으로 의미가 없습니다.

---

### 6. XSS - Reflected (eval)

**개념**: 입력값(`date` 파라미터)을 서버 또는 클라이언트 측 자바스크립트 `eval()` 함수에 그대로 넘겨 "코드"로 실행시키는 구조.

**핵심 페이로드**
```
?date=alert(document.cookie)
```

**개념 정리**: `eval()`은 문자열을 코드로 변환해 실행하는 함수라서, 사용자 입력이 들어가는 순간 "데이터"와 "명령"의 구분이 사라집니다. PHP Code Injection(A1-10)과 본질적으로 같은 패턴이 자바스크립트 쪽에서 재현된 것.

---

### 7. XSS - Reflected (href)

**개념**: 투표(Vote) 기능의 입력값이 `<a href="...">` 같은 속성값 안에 그대로 삽입되는 경우.

**핵심 페이로드**
```html
<script>alert("hacker")</script>
```

**결과**: Vote 필드에 입력한 값이 그대로 화면(또는 링크) 안에 노출되며 스크립트가 실행됨.

---

### 8. XSS - Reflected (Login Form)

**개념**: 로그인 실패 시 보여주는 에러 메시지에 "입력했던 아이디 값"을 그대로 다시 보여주는 경우가 많은데, 이 값이 인코딩되지 않고 출력됨.

**핵심 페이로드**
```html
'<script>document.cookie</script>
```

---

### 9. phpMyAdmin 비정상 접근

**개념**: bee-box(bWAPP이 올라가 있는 VM)에는 데이터베이스 관리 도구인 phpMyAdmin이 같이 떠 있는데, 별도 로그인 절차 없이(또는 기본/취약 계정으로) 접근이 가능했던 사례입니다.

**시도한 방법**
```
test[a@http://172.16.11.216@] 클릭
```


---

### 10. PHP_SELF XSS

**개념**: 오래된 PHP 코드에서는 폼이 "제출 후 같은 페이지로 돌아오게" 하려고 아래처럼 작성하는 경우가 많습니다.
```php
<form action="<?php echo $_SERVER['PHP_SELF']; ?>" method="post">
```
`$_SERVER['PHP_SELF']`는 **현재 실행 중인 스크립트의 경로**를 돌려주는데, 여기에는 공격자가 URL 뒤에 추가로 붙인 경로까지 그대로 포함됩니다. 이 값이 `action="..."` 속성에 인코딩 없이 들어가기 때문에, 속성을 깨고 나와 새 태그를 주입할 수 있습니다.

**핵심 페이로드**
```html
<script>(document.cookie)</script>
```
(URL 경로 부분에 위 코드를 붙여서 요청 → form action 속성 안에서 탈출하며 실행)

**방어법**: `$_SERVER['PHP_SELF']` 출력 시 반드시 `htmlspecialchars()` 처리, 가능하면 action 속성 자체를 생략(빈 값이면 현재 페이지로 자동 제출됨).

---

### 11. XSS - Reflected (Referer)

**개념**: 일부 페이지(특히 관리자용 로그/통계 페이지)는 "어디서 들어왔는지" 분석용으로 `Referer` 헤더 값을 화면에 그대로 찍어주는 경우가 있습니다. 이 값은 브라우저 주소창으로는 절대 조작할 수 없고, **오직 프록시 도구로만 변조 가능**합니다.

**공략 절차**
1. Burp Suite Proxy로 요청 가로채기.
2. `Referer` 헤더 값을 아래처럼 변경.
```
Referer: http://172.16.11.216/bWAPP/portal.php<script>alert(document.cookie)</script>
```
3. Forward → 해당 값이 출력되는 페이지에서 스크립트 실행 확인.

**왜 해킹이 되는가**: 개발자가 "Referer는 브라우저가 자동으로 채워주는 값이니 사용자가 직접 조작할 수 없다"고 안심하고 검증을 생략한 케이스. 그러나 HTTP 요청 자체는 클라이언트(공격자)가 100% 제어 가능한 영역이라, 이런 가정은 항상 깨질 수 있습니다.

---

### 12. XSS - Reflected (User-Agent)

**개념**: 11번과 동일한 원리, 대상 헤더만 `User-Agent`로 바뀐 경우.

**핵심 페이로드**
```
User-Agent: http://172.16.11.216/bWAPP/portal.php<script>alert(document.cookie)</script>
```

**정리**: Referer, User-Agent, X-Forwarded-For 등 "서버 로그에 자주 찍히는 헤더"들은 대부분 클라이언트가 보내는 값이므로, 이런 헤더를 화면(특히 관리자 로그 뷰어)에 그대로 출력하는 모든 기능은 XSS 후보로 의심해야 합니다.

---

### 13. XSS - Stored (Blog)

**개념**: A2 요약표에서 이미 정리한 Stored XSS와 동일한 패턴. 블로그 글쓰기 폼에 스크립트를 넣고 등록하면, 이후 그 글을 보는 모든 사용자에게 실행됩니다.

---

### 14. XSS - Stored (Change Secret)

**개념**: "Secret" 값을 변경하는 기능에 스크립트를 저장해두면, 저장 시점에는 바로 실행되지 않지만 **이 값이 다시 화면에 표시되는 시점(로그인 후 등)** 에 실행되는 지연 발동형 Stored XSS입니다.

**공략 절차**
1. Secret 변경 폼에 페이로드 입력 후 저장.
```html
'<script>document.cookie</script>
```
2. 로그아웃 후 다시 로그인.
3. 로그인 후 화면에서 alert 창에 쿠키 값이 찍히는 것 확인.

**개념 정리**: Reflected XSS는 "그 요청 한 번"에만 영향을 주지만, 이 챌린지처럼 DB에 저장된 값이 나중에 다시 출력되면 **저장된 시점과 실행되는 시점이 분리**됩니다. 공격자가 직접 그 자리에 없어도, 피해자가 나중에 그 데이터를 볼 때마다 계속 실행되는 게 Stored XSS의 위력입니다.

---

### 15. XSS - Stored (Cookies)

**개념**: `genre` 파라미터 값이 서버에서 **쿠키 값으로 그대로 저장**되고, 이후 페이지를 다시 방문할 때 그 쿠키 값을 읽어와 화면에 다시 출력하는 구조입니다. "저장 장소가 DB가 아니라 사용자 자신의 쿠키"라는 점이 독특한 변형입니다.

**핵심 페이로드**
```
?genre=<script>...</script>
```

**공략 절차**
1. 위 파라미터로 요청을 보내면 서버가 그 값을 쿠키에 기록.
2. 개발자 도구(F12) → Application/Storage → Cookies에서 해당 쿠키 값에 스크립트 문자열이 그대로 들어가 있는 것 확인.
3. 이후 같은 사이트의 다른 페이지를 방문하면, 그 쿠키 값을 출력하는 곳에서 스크립트가 실행됨.

**방어법(13~15 공통)**: 저장 시점/출력 시점 모두에서 인코딩, 쿠키에 사용자 입력값을 직접 저장하지 않거나 저장 시 엄격히 검증.

---

### 16. SQLite (다음 실습)

bWAPP 메뉴에서 SQLite 기반 챌린지로 이동 예정. A1-17(SQL Injection - SQLite)에서 다룬 `'or 1=1--` (마지막 공백 필수) 패턴이 여기서도 동일하게 적용될 가능성이 높습니다.

---

## A4 요약표

| # | 취약점 | 핵심 동작 | 한 줄 개념 |
|---|--------|-----------|------------|
| 1 | DOR (Change Secret) | Burp로 요청 파라미터(대상 사용자 식별값) 변조 | 본인 확인 없이 타인의 Secret 변경 시도 |
| 2 | DOR (Reset Secret) | Burp로 수정 시도 → 미완료 | 서버 측 추가 검증이 있는 것으로 추정, 재검토 필요 |
| 3 | DOR (Order Tickets) | 수량/가격 파라미터를 Burp에서 임의로 변경 | 클라이언트가 보낸 값을 서버가 그대로 신뢰 |

---

## A4 - Insecure / Direct Object Reference (DOR)

> **DOR**은 A2 요약표의 28번 "Session Mgmt. – Administrative Portals (IDOR)"과 본질적으로 같은 개념입니다. "서버가 **인증(로그인 여부)** 은 확인하지만, **인가(이 데이터/기능에 접근할 권한이 있는지)** 는 확인하지 않는다"는 점이 핵심입니다. URL이나 요청 파라미터에 들어있는 ID·식별값만 바꾸면, 서버는 "이 요청을 보낸 사람이 그 객체의 진짜 주인인지"를 검증하지 않고 그냥 처리해버립니다.

### 1. DOR (Change Secret)

**개념**: Secret 값을 바꾸는 요청에 "어떤 계정의 Secret을 바꿀 것인지"를 나타내는 값(숫자 ID 등)이 파라미터로 같이 전달되는데, 이 값을 다른 사용자의 것으로 바꿔도 서버가 막지 않는지 확인하는 실습입니다.

**공략 절차**
1. 정상적으로 본인 Secret 변경 요청을 한 번 보내고 Burp로 캡처.
2. 요청 안에 들어있는 사용자 식별 파라미터를 다른 값으로 변경 후 Forward.
3. 다른 사용자의 Secret이 실제로 바뀌었는지(또는 조회 가능해졌는지) 확인.

---

### 2. DOR (Reset Secret)

**개념**: Change Secret과 유사하지만 "초기화(Reset)" 기능 쪽 엔드포인트.

> 📝 시도는 했지만 아직 성공 못함(파라미터를 바꿔도 반영이 안 됨) — 추정 원인:
> - 단순 ID 파라미터 외에 **토큰/세션 기반 추가 검증**이 걸려있을 가능성
> - Repeater로 보낸 요청에 **세션 쿠키가 같이 전달되지 않아서** 인증 자체가 깨졌을 가능성 (Burp Repeater는 기본적으로 원래 요청의 쿠키/헤더를 그대로 들고 가지만, 직접 수정하다가 빠뜨리는 경우가 흔합니다)
> - 요청 본문의 **Content-Length 값이 파라미터 길이 변경 후 자동으로 안 맞춰진** 경우 (Burp는 보통 자동 보정하지만 수동 편집 시 확인 필요)
>
> 다음 시도 시 Burp Repeater에서 **요청 헤더 전체(쿠키 포함)** 가 원본과 동일한지부터 한 번 확인해보시면 좋을 것 같습니다.

---

### 3. DOR (Order Tickets)

**개념**: 티켓 구매 시 수량과 가격이 **서버가 아니라 클라이언트(브라우저)가 계산해서 서버로 전달**하는 구조였다는 게 핵심 문제입니다. 정상적인 설계라면 "가격은 서버가 DB 기준으로 다시 계산"해야 하는데, 이 페이지는 클라이언트가 보낸 숫자를 그대로 믿고 결제를 진행합니다.

**공략 절차**
1. 정상적으로 티켓 1장을 담는 요청을 Burp로 캡처.
2. 요청 본문의 수량(quantity)·가격(price) 파라미터 값을 직접 원하는 숫자로 변경 (예: 수량 100, 가격 0.01).
3. Forward → 서버가 변경된 값을 그대로 받아들여 비정상적인 주문이 생성되는지 확인.

**개념 정리**: 이건 DOR(IDOR)이라기보다는 **클라이언트 측 신뢰(Client-Side Trust) 문제**에 더 가깝습니다. "가격/수량처럼 돈과 직결되는 값은 절대 클라이언트를 믿지 말고, 서버가 DB에 저장된 원본 가격으로 항상 재계산해야 한다"는 게 핵심 교훈입니다.

**방어법(A4 공통)**:
1. 서버가 매 요청마다 "현재 로그인한 사용자 == 이 객체(Secret/Ticket/Order)의 실제 소유자"인지 비교하는 로직을 반드시 추가
2. 가격·수량처럼 민감한 값은 클라이언트에서 받지 않고, 서버에 저장된 값을 기준으로 서버가 직접 계산
3. ID를 추측 가능한 순차적인 숫자로 노출하지 않기 (근본 해결책은 아니지만 추측 난이도 상승)

---

## Juice Shop 실습 환경 구축 (Docker)

다음 실습 대상인 OWASP Juice Shop을 우분투(공격 대상 서버 역할)에 Docker로 띄운 과정입니다.

```bash
sudo apt install -y docker.io        # 도커 엔진 설치
sudo systemctl start docker          # 도커 데몬(백그라운드 서비스) 시작
sudo docker images                   # 로컬에 받아둔 이미지 목록 확인 (처음엔 비어있음)
sudo docker pull bkimminich/juice-shop   # 공식 Juice Shop 이미지를 다운로드
sudo docker run --rm -p 3000:3000 bkimminich/juice-shop   # 컨테이너 실행
```

**각 명령어 의미**
| 명령어 | 의미 |
|---|---|
| `apt install docker.io` | 컨테이너를 띄우기 위한 도커 엔진 자체를 설치 |
| `systemctl start docker` | 설치된 도커 서비스를 실행 상태로 전환 |
| `docker images` | 현재 로컬에 캐시된 이미지(설계도) 목록 확인 |
| `docker pull` | Docker Hub에서 Juice Shop 이미지를 받아옴 |
| `docker run --rm -p 3000:3000` | 컨테이너 실행. `--rm`은 종료 시 컨테이너 자동 삭제(임시 실습용), `-p 3000:3000`은 컨테이너 내부 3000번 포트를 호스트(우분투)의 3000번 포트로 연결 |

이후 브라우저로 `http://[우분투IP]:3000` 접속 → 정상 쇼핑몰 화면 확인 → `score-board` 경로로 첫 챌린지(Score Board 찾기) 클리어, 1점 획득.

---

## 사전 정찰 - nmap

본격적으로 Juice Shop을 공격하기 전, 공격 대상(172.16.11.210)에 대한 정보 수집(Reconnaissance) 단계로 실행한 명령입니다.

```bash
nmap -sC -A -p- 172.16.11.210
```

**옵션별 의미와 "왜 이걸 쓰는가"**

| 옵션 | 의미 | 정찰(해킹) 관점에서의 역할 |
|---|---|---|
| `-p-` | 전체 포트(1~65535) 스캔 | 기본 nmap은 자주 쓰이는 상위 1000개 포트만 봄. 운영자가 일부러 비표준 포트(예: 8443, 3000 등)에 서비스를 숨겨놨을 수 있으므로, **숨겨진 진입점을 놓치지 않기 위해** 전체 포트를 다 훑음 |
| `-A` | OS 탐지 + 버전 탐지 + 스크립트 스캔(`-sC`와 중복 포함) + traceroute를 한번에 활성화 | 어떤 OS·서비스·버전을 쓰는지 알아야 **그 버전에 알려진 CVE(취약점)** 가 있는지 검색할 수 있음. 즉, "어디를 공격할지" 우선순위를 정하기 위한 정보 |
| `-sC` | Nmap의 기본 NSE(Nmap Scripting Engine) 스크립트 세트 실행 | 단순히 포트가 열려있다는 사실 외에, 그 서비스에 안전하게 질의해서 **배너 정보, 기본 설정, 알려진 취약점 시그니처** 등을 자동으로 추가 확인해줌 |

**이 명령이 "해킹"으로 이어지는 흐름**
```
① 전체 포트를 스캔해서 "어떤 문이 열려있는지" 파악 (-p-)
        ↓
② 열려있는 각 포트에서 어떤 서비스/버전이 도는지 확인 (-A)
        ↓
③ 알려진 버전이면 → 검색엔진/CVE 목록에서 "이 버전, 이 서비스"에
   해당하는 공개된 취약점(Exploit)이 있는지 조사
        ↓
④ NSE 스크립트(-sC)가 추가로 잡아준 배너·설정 정보로
   공격 가능한 지점(예: 디렉토리 리스팅, 기본 계정 등)을 좁힘
        ↓
⑤ 위 정보를 바탕으로 본격적인 취약점 공격(SQLi, XSS, 인증 우회 등) 시작
```

즉, nmap은 직접 시스템을 뚫는 도구가 아니라 **"어디를, 어떤 방법으로 공격할지" 결정하기 위한 지도 제작 단계**입니다. 실전 모의해킹에서도 항상 이 정찰(Reconnaissance) 단계가 가장 먼저 옵니다.

---

> ⚠️ 본 자료는 bWAPP / OWASP Juice Shop과 같이 학습을 위해 의도적으로 취약하게 설계된 환경에서의 실습 정리이며, 허가받지 않은 시스템에는 절대 적용해서는 안 됩니다.
