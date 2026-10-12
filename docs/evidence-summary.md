# 웹 서비스 설치 및 접속 검증 기록

## 1. SSH 접속

Windows PowerShell의 SSH 클라이언트로 EC2 퍼블릭 IP에 접속하였다. 로그인 로그에서 운영체제와 프라이빗 IP를 확인하였다.

```text
Welcome to Ubuntu 26.04 LTS (GNU/Linux 7.0.0-1006-aws x86_64)
IPv4 address for ens5: 10.0.1.242
ubuntu@ip-10-0-1-242:~$ pwd
/home/ubuntu
```

Ubuntu 셸과 사용자 홈 디렉터리가 표시되어 서버 내부 접속이 성공했음을 확인하였다.

## 2. Nginx 설치

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

설치 로그:

```text
Fetched 32.4 MB in 4s (8005 kB/s)
Setting up nginx-common (1.28.3-2ubuntu1.11) ...
Setting up nginx (1.28.3-2ubuntu1.11) ...
```

패키지 저장소에서 자료를 다운로드하고 Nginx 설치를 완료하였다.

## 3. 서비스 실행 상태

```bash
systemctl status nginx --no-pager
```

```text
Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
Active: active (running) since Mon 2026-10-12 15:22:41 UTC
Main PID: 1985 (nginx)
```

Nginx가 실행 중이며 서버 부팅 시 자동으로 시작하도록 설정되어 있음을 확인하였다. 로그의 UTC 시각은 한국 시간으로 2026-10-13 00:22:41이다.

## 4. 서버 내부 HTTP 응답

```bash
curl -I http://localhost
```

```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Mon, 12 Oct 2026 15:22:53 GMT
Content-Type: text/html
Content-Length: 615
```

한국 시간 2026-10-13 01:30:48에 해당하는 두 번째 확인에서도 동일하게 `200 OK`가 반환되었다.

```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Mon, 12 Oct 2026 16:30:48 GMT
```

HEAD 요청으로 EC2 내부의 웹 서버 응답 헤더를 확인하였다.

## 5. 외부 브라우저 접속

- 검증 방식: (A) 브라우저 접속
- URL: http://3.38.244.69
- 표시된 페이지: Welcome to nginx!

![외부 접속 결과](evidence/browser-access.png)

퍼블릭 IP를 이용한 외부 브라우저 접속에서 Nginx 기본 페이지가 정상적으로 표시되었다.

## 6. 검증 결과

| 확인 항목 | 결과 |
|---|---|
| SSH 서버 접속 | Ubuntu 셸 진입 성공 |
| Nginx 설치 | 설치 완료 로그 확인 |
| Nginx 실행 | active (running) |
| 자동 시작 설정 | enabled |
| 내부 HTTP 응답 | 200 OK |
| 외부 웹페이지 접속 | Nginx 환영 페이지 표시 |
