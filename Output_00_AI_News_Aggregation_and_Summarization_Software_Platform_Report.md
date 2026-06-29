# AI 뉴스 자동 수집·요약 소프트웨어 플랫폼 구축 보고서  
---

## 목차

```text
1. 구축 개요
2. 구축 범위 및 제외 범위
3. 컴퓨터 자원 사양 검토
4. 시스템 구성 설계
5. 플랫폼 구축 절차
```

---

# 1. 구축 개요

## 1.1 구축 목적

본 보고서는 **AI 뉴스 자동 수집 및 요약 플랫폼**을 구축하기 위한 최종 보고서이다.

플랫폼의 핵심 목적은 다음과 같다.

| 목적        | 설명                                      |
| ----------- | ----------------------------------------- |
| 자동 수집   | RSS Feed를 통해 AI 관련 뉴스를 자동 수집  |
| 자동 필터링 | AI 관련 키워드를 기준으로 기사 선별       |
| 자동 요약   | Ollama 로컬 LLM을 이용해 한국어 요약 생성 |
| 자동 저장   | 요약 결과를 Notion 데이터베이스에 저장    |
| 중복 방지   | 이미 저장된 기사는 다시 저장하지 않음     |

---

## 1.2 구축 대상 플랫폼

본 플랫폼은 다음 구성으로 구축한다.

| 구분               | 사용 기술              |
| ------------------ | ---------------------- |
| 운영체제           | Ubuntu 24.04 LTS       |
| 컨테이너 환경      | Docker, Docker Compose |
| 자동화 도구        | n8n                    |
| 로컬 LLM 실행 도구 | Ollama                 |
| 요약 모델          | `qwen2.5:3b`           |
| 데이터 저장소      | Notion Database        |
| 뉴스 수집 방식     | RSS Feed               |

---

# 2. 구축 범위 및 제외 범위

## 2.1 구축 범위

본 보고서에서 다루는 범위는 **플랫폼 구축**에 한정한다.

| 구분               | 포함 내용                                    |
| ------------------ | -------------------------------------------- |
| 서버 환경 구축     | Ubuntu 기본 설정, 시간대 설정, 방화벽 설정   |
| 컨테이너 환경 구축 | Docker 및 Docker Compose 설치                |
| 자동화 환경 구축   | n8n Docker 컨테이너 구성                     |
| AI 요약 환경 구축  | Ollama 설치 및 모델 다운로드                 |
| 시스템 연동        | n8n에서 Ollama API 호출 설정                 |
| 저장소 연동        | Notion Database 저장 구조 설정               |
| 워크플로우 구성    | RSS 수집, 필터링, 중복 확인, 요약, 저장 흐름 |
| 자원 검토          | 현재 컴퓨터 사양의 적합성 평가               |
| ~~운영 설정~~          | ~~보안, 포트, 백업, 로그 확인 방안~~             |
| ~~검증~~               | ~~설치 및 동작 확인 체크리스트~~                 |

---

## 2.2 제외 범위

본 보고서에서는 다음 항목은 다루지 않는다.

| 구분            | 제외 내용                                |
| --------------- | ---------------------------------------- |
| 서비스 기획     | 사용자 서비스 전략, 수익 모델            |
| 뉴스 품질 평가  | 기사 신뢰도 평가, 언론사별 가중치        |
| 고급 AI 분석    | 감성 분석, 트렌드 예측, 주제 분류 고도화 |
| 대규모 배포     | 다중 사용자용 클라우드 서비스 운영       |
| 프론트엔드 개발 | 별도 웹 화면 또는 모바일 앱 개발         |

즉, 본 문서는 **로컬 서버 기반 AI 뉴스 자동화 플랫폼을 실제로 구축하고 운영 가능한 상태로 만드는 것**에 초점을 둔다.

---

# 3. 컴퓨터 자원 사양 검토

## 3.1 검토 대상 사양

사용 가능한 컴퓨터 자원은 다음과 같다.

| 항목   | 사양                   |
| ------ | ---------------------- |
| CPU    | Intel® Core™ i5-7300HQ |
| 메모리 | 16GB                   |
| GPU    | Nvidia GP107M 4GB      |
| VRAM   | 4GB                    |

Nvidia GP107M은 일반적으로 GTX 1050 Mobile 계열 GPU로 볼 수 있다.

---

## 3.2 CPU 적합성 검토

Intel® Core™ i5-7300HQ는 4코어 기반 모바일 CPU이다.

최신 고성능 CPU는 아니지만 다음 작업에는 충분하다.

```text
- Ubuntu 서버 운영
- Docker 실행
- n8n 워크플로우 실행
- RSS 데이터 수집
- 키워드 필터링
- Notion API 호출
- 하루 1~수 건 수준의 LLM 요약 작업
```

다만 Ollama가 CPU 중심으로 실행될 경우 요약 속도는 빠르지 않을 수 있다.

하지만 본 플랫폼은 실시간 응답 서비스가 아니라 **정해진 시간에 자동 실행되는 배치형 자동화 플랫폼**이므로 CPU 성능은 실사용에 큰 문제가 없다.

---

## 3.3 메모리 적합성 검토

메인 메모리 16GB는 본 플랫폼 구축에 적합하다.

예상 메모리 사용량은 다음과 같다.

| 구성 요소           | 예상 사용량      |
| ------------------- | ---------------- |
| Ubuntu 24.04        | 약 1.5~3GB       |
| Docker 및 n8n       | 약 0.5~1.5GB     |
| Ollama `qwen2.5:3b` | 약 2~5GB         |
| 시스템 여유 메모리  | 약 6GB 이상 예상 |

따라서 16GB 메모리 환경에서는 n8n과 Ollama를 동시에 실행하는 데 무리가 없다.

---

## 3.4 GPU 및 VRAM 적합성 검토

Nvidia GP107M 4GB GPU는 대형 LLM을 실행하기에는 부족하지만, 소형 모델 실행 보조 용도로는 활용 가능성이 있다.

| 항목             | 평가              |
| ---------------- | ----------------- |
| GPU 가속         | 제한적으로 가능   |
| VRAM 4GB         | 소형 모델에 적합  |
| 7B 이상 모델     | 비권장            |
| 3B급 모델        | 적합              |
| 본 프로젝트 모델 | `qwen2.5:3b` 적절 |

현재 사양에서는 7B, 14B 이상 대형 모델보다 `qwen2.5:3b`와 같은 경량 모델을 사용하는 것이 적절하다.

---

## 3.5 최종 자원 평가

| 항목                   | 평가   |
| ---------------------- | ------ |
| Ubuntu 운영            | 적합   |
| Docker 실행            | 적합   |
| n8n 실행               | 적합   |
| Ollama 실행            | 가능   |
| `qwen2.5:3b` 모델 실행 | 적합   |
| 대형 LLM 실행          | 비권장 |
| 장기 자동 운영         | 가능   |

최종적으로 해당 컴퓨터 사양은 본 플랫폼 구축 및 운영에 **충분히 적합**하다.

다만 안정적인 운영을 위해 다음 조건을 권장한다.

```text
- 요약 대상 기사 수를 1회 실행당 1~3건 수준으로 제한
- 입력 본문 길이를 3000자 내외로 제한
- 3B급 경량 모델 사용
- 동시에 여러 개의 LLM 작업을 실행하지 않음
```

---

# 4. 시스템 구성 설계

## 4.1 전체 시스템 구조

전체 시스템은 다음과 같이 구성한다.

```text
Ubuntu 24.04 Host
│
├── Docker
│   └── n8n Container
│       ├── RSS Feed 수집
│       ├── AI 키워드 필터링
│       ├── 최신 기사 선택
│       ├── Notion 중복 확인
│       ├── Ollama 요약 요청
│       └── Notion 저장
│
├── Ollama
│   └── qwen2.5:3b 모델
│
└── External Service
    └── Notion Database
```

---

## 4.2 구성 요소별 역할

| 구성 요소      | 역할                              |
| -------------- | --------------------------------- |
| Ubuntu 24.04   | 전체 플랫폼의 운영체제            |
| Docker         | n8n 실행 환경 제공                |
| Docker Compose | n8n 컨테이너 설정 관리            |
| n8n            | 뉴스 수집·필터링·요약·저장 자동화 |
| Ollama         | 로컬 LLM 실행                     |
| `qwen2.5:3b`   | 뉴스 요약 생성 모델               |
| Notion         | 요약 결과 저장소                  |
| RSS Feed       | 뉴스 데이터 입력원                |

---

## 4.3 데이터 처리 흐름

플랫폼의 데이터 흐름은 다음과 같다.

```text
RSS Feed
   ↓
n8n 수집
   ↓
AI 관련 기사 필터링
   ↓
최신 기사 선택
   ↓
Notion 중복 확인
   ↓
Ollama 요약
   ↓
Notion 저장
```

---

## 4.4 n8n과 Ollama 연결 구조

n8n은 Docker 컨테이너 내부에서 실행하고, Ollama는 Ubuntu Host에서 실행한다.

따라서 n8n에서 Ollama API를 호출할 때 다음 주소를 사용한다.

```text
http://host.docker.internal:11434/api/generate
```

연결 구조는 다음과 같다.

```text
n8n Container
    ↓
host.docker.internal:11434
    ↓
Ubuntu Host
    ↓
Ollama API
    ↓
qwen2.5:3b
```

---

# 5. 플랫폼 구축 절차

# 5.1 Ubuntu 기본 설정

## 5.1.1 시스템 업데이트

```bash
sudo apt update
sudo apt upgrade -y
```

## 5.1.2 기본 도구 설치

```bash
sudo apt install -y curl wget git ca-certificates gnupg lsb-release ufw
```

## 5.1.3 시간대 설정

```bash
sudo timedatectl set-timezone Asia/Seoul
```

확인:

```bash
timedatectl
```

정상 예시:

```text
Time zone: Asia/Seoul
```

---

# 5.2 방화벽 설정

## 5.2.1 SSH 허용

```bash
sudo ufw allow OpenSSH
```

## 5.2.2 n8n 포트 허용

```bash
sudo ufw allow 5678/tcp
```

## 5.2.3 방화벽 활성화

```bash
sudo ufw enable
```

## 5.2.4 상태 확인

```bash
sudo ufw status
```

Ollama 포트 `11434`는 외부에 직접 공개하지 않는 것을 권장한다.

---

# 5.3 Docker 및 Docker Compose 설치

## 5.3.1 Docker 설치 준비

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

## 5.3.2 Docker GPG 키 등록

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

## 5.3.3 Docker 저장소 추가

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 5.3.4 Docker 설치

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 5.3.5 설치 확인

```bash
docker --version
docker compose version
```

## 5.3.6 현재 사용자 권한 설정

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 5.3.7 Docker 실행 테스트

```bash
docker run hello-world
```

---

# 5.4 n8n 설치

## 5.4.1 작업 디렉터리 생성

```bash
mkdir -p ~/n8n
cd ~/n8n
```

## 5.4.2 환경 변수 파일 생성

```bash
nano .env
```

예시:

```env
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=change_this_strong_password

N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http

GENERIC_TIMEZONE=Asia/Seoul
TZ=Asia/Seoul
```

운영 환경에서는 `N8N_BASIC_AUTH_PASSWORD` 값을 반드시 강력한 비밀번호로 변경해야 한다.

---

## 5.4.3 Docker Compose 파일 생성

```bash
nano docker-compose.yml
```

내용:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    env_file:
      - .env
    volumes:
      - n8n_data:/home/node/.n8n
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  n8n_data:
```

여기서 중요한 설정은 다음이다.

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

이 설정을 통해 n8n 컨테이너가 Ubuntu Host에서 실행 중인 Ollama에 접근할 수 있다.

---

## 5.4.4 n8n 실행

```bash
docker compose up -d
```

## 5.4.5 실행 상태 확인

```bash
docker ps
```

## 5.4.6 로그 확인

```bash
docker logs -f n8n
```

## 5.4.7 n8n 접속

로컬 환경:

```text
http://localhost:5678
```

원격 서버 환경:

```text
http://서버IP:5678
```

---

# 5.5 Ollama 설치

## 5.5.1 Ollama 설치

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

## 5.5.2 서비스 상태 확인

```bash
systemctl status ollama
```

정상 상태:

```text
Active: active (running)
```

---

# 5.6 LLM 모델 설치

## 5.6.1 `qwen2.5:3b` 모델 다운로드

```bash
ollama pull qwen2.5:3b
```

## 5.6.2 모델 목록 확인

```bash
ollama list
```

정상 예시:

```text
qwen2.5:3b
```

---

# 5.7 Ollama API 테스트

## 5.7.1 모델 목록 API 확인

```bash
curl http://localhost:11434/api/tags
```

## 5.7.2 요약 생성 테스트

```bash
curl http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5:3b",
    "prompt": "다음 문장을 한국어로 요약해줘. Artificial intelligence is changing the software industry.",
    "stream": false
  }'
```

한국어 요약 응답이 출력되면 Ollama와 모델이 정상 동작하는 것이다.

---

# 5.8 n8n과 Ollama 연결 설정

## 5.8.1 Ollama Host 바인딩 설정

n8n 컨테이너에서 Ubuntu Host의 Ollama API에 접근할 수 있도록 Ollama 서비스 환경변수를 설정한다.

```bash
sudo systemctl edit ollama
```

다음 내용을 입력한다.

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

## 5.8.2 설정 적용

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## 5.8.3 포트 확인

```bash
ss -tulnp | grep 11434
```

정상 예시:

```text
LISTEN 0 4096 0.0.0.0:11434
```

## 5.8.4 n8n 컨테이너 내부 연결 테스트

```bash
docker exec -it n8n sh
```

컨테이너 내부에서 실행:

```bash
wget -qO- http://host.docker.internal:11434/api/tags
```

모델 목록이 출력되면 n8n과 Ollama 연결이 정상이다.
