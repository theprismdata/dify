# Dify 개발 환경 서비스 가동 가이드

이 문서는 Dify 플랫폼을 개발자 모드로 실행하기 위한 모든 서비스들의 가동 방법을 설명합니다.

## 📋 목차
1. [필수 사전 요구사항](#필수-사전-요구사항)
2. [미들웨어 서비스 (Docker)](#미들웨어-서비스-docker)
3. [백엔드 API 서비스](#백엔드-api-서비스)
4. [프론트엔드 서비스](#프론트엔드-서비스)
5. [LLM 서비스 (Ollama)](#llm-서비스-ollama)
6. [백그라운드 워커 서비스](#백그라운드-워커-서비스)
7. [서비스 상태 확인](#서비스-상태-확인)
8. [문제 해결](#문제-해결)

---

## 필수 사전 요구사항

### 시스템 요구사항
- **CPU**: >= 2 Core
- **RAM**: >= 4 GiB
- **macOS**: 15.5+ (ARM64)

### 설치된 도구들
- **Docker & Docker Compose**
- **Python 3.12+**
- **Node.js >= v22.11.x**
- **UV** (Python 패키지 매니저)
- **pnpm** (Node.js 패키지 매니저)
- **Ollama** (로컬 LLM 서버)

---

## 미들웨어 서비스 (Docker)

### 1. 환경 설정
```bash
cd /Users/prismdata/Documents/entec/dify/docker
cp middleware.env.example middleware.env
```

### 2. 포트 설정 (필요 시 수정)
```bash
# middleware.env 파일에서 포트 설정 확인/수정
EXPOSE_POSTGRES_PORT=5432
EXPOSE_REDIS_PORT=6379
EXPOSE_WEAVIATE_PORT=8081  # 8080에서 8081로 변경됨
EXPOSE_SANDBOX_PORT=8194
EXPOSE_SSRF_PROXY_PORT=3128
```

### 3. Docker Compose 실행
```bash
cd /Users/prismdata/Documents/entec/dify/docker

# 모든 미들웨어 서비스 시작
docker compose -f docker-compose.middleware.yaml --profile weaviate -p dify up -d
```

### 4. 포함된 서비스들
- **PostgreSQL** (포트 5432) - 메인 데이터베이스
- **Redis** (포트 6379) - 캐시 및 세션 스토어  
- **Weaviate** (포트 8081) - 벡터 데이터베이스
- **Sandbox** (포트 8194) - 코드 실행 환경
- **SSRF Proxy** (포트 3128) - 보안 프록시
- **Plugin Daemon** (포트 5002-5003) - 플러그인 관리

### 5. 서비스 중지
```bash
docker compose -f docker-compose.middleware.yaml --profile weaviate -p dify down
```

---

## 백엔드 API 서비스

### 1. 디렉터리 이동
```bash
cd /Users/prismdata/Documents/entec/dify/api
```

### 2. 환경 설정
```bash
# 환경 변수 파일 생성
cp .env.example .env

# SECRET_KEY 생성 및 설정 (macOS)
secret_key=$(openssl rand -base64 42)
sed -i '' "s/^SECRET_KEY=$/SECRET_KEY=${secret_key}/" .env
```

### 3. 의존성 설치
```bash
# UV 패키지 매니저로 의존성 설치
uv sync --dev
```

### 4. 데이터베이스 마이그레이션
```bash
# 데이터베이스를 최신 버전으로 업데이트
uv run flask db upgrade
```

### 5. Flask API 서버 실행
```bash
# 개발 모드로 실행 (디버그 모드 포함)
uv run flask run --host 0.0.0.0 --port=5001 --debug
```

**접속 주소**: 
- 로컬: http://127.0.0.1:5001
- 네트워크: http://192.168.0.83:5001

### 6. 백그라운드 실행 (권장)
```bash
# 백그라운드에서 실행하려면 nohup 사용
nohup uv run flask run --host 0.0.0.0 --port=5001 --debug > flask.log 2>&1 &
```

---

## 프론트엔드 서비스

### 1. 디렉터리 이동
```bash
cd /Users/prismdata/Documents/entec/dify/web
```

### 2. 의존성 설치
```bash
# pnpm으로 의존성 설치
pnpm install
```

### 3. 환경 설정
```bash
# 환경 변수 파일 생성
cp .env.example .env.local

# 기본 설정 (API 엔드포인트)
echo 'NEXT_PUBLIC_API_PREFIX=http://localhost:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://localhost:5001/api
NEXT_TELEMETRY_DISABLED=1
NEXT_PRIVATE_STANDALONE=true' > .env.local
```

### 4. Next.js 개발 서버 실행
```bash
# 메모리 할당 최적화와 함께 실행
NODE_OPTIONS='--max-old-space-size=4096' pnpm run dev
```

**접속 주소**:
- 로컬: http://localhost:3000
- 네트워크: http://192.168.0.83:3000

### 5. 백그라운드 실행 (권장)
```bash
# 백그라운드에서 실행
nohup NODE_OPTIONS='--max-old-space-size=4096' pnpm run dev > nextjs.log 2>&1 &
```

---

## LLM 서비스 (Ollama)

### 1. Ollama 서버 실행
```bash
# Ollama 서비스 시작
ollama serve
```

**포트**: 11434

### 2. 사용 가능한 모델 확인
```bash
# 설치된 모델 목록 확인
ollama list
```

### 3. 새 모델 다운로드 (필요 시)
```bash
# 예시: Gemma 모델 다운로드
ollama pull gemma2:2b
ollama pull gemma3:4b
```

### 4. 백그라운드 실행 (이미 실행 중)
Ollama는 일반적으로 시스템 서비스로 자동 실행됩니다.

---

## 백그라운드 워커 서비스

### 1. Celery 워커 (문서 처리)
```bash
cd /Users/prismdata/Documents/entec/dify/api

# 문서 인덱싱 및 처리를 위한 워커 실행
uv run celery -A app.celery worker -P gevent -c 1 --loglevel INFO -Q dataset,generation,mail,ops_trace,app_deletion,plugin,workflow_storage
```

### 2. Celery Beat (스케줄러) - 선택사항
```bash
cd /Users/prismdata/Documents/entec/dify/api

# 정기 작업 스케줄러 실행
uv run celery -A app.celery beat
```

### 3. 백그라운드 실행
```bash
# Celery 워커 백그라운드 실행
nohup uv run celery -A app.celery worker -P gevent -c 1 --loglevel INFO -Q dataset,generation,mail,ops_trace,app_deletion,plugin,workflow_storage > celery_worker.log 2>&1 &

# Celery Beat 백그라운드 실행
nohup uv run celery -A app.celery beat > celery_beat.log 2>&1 &
```

---

## 서비스 상태 확인

### 1. Docker 컨테이너 상태
```bash
# 실행 중인 Docker 컨테이너 확인
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 2. Python 프로세스 상태
```bash
# Flask 및 Celery 프로세스 확인
ps aux | grep -E "(flask|celery)" | grep -v grep
```

### 3. Node.js 프로세스 상태
```bash
# Next.js 프로세스 확인
ps aux | grep next | grep -v grep
```

### 4. Ollama 상태
```bash
# Ollama 프로세스 확인
ps aux | grep ollama | grep -v grep

# Ollama API 테스트
curl -s http://localhost:11434/api/tags
```

### 5. 전체 서비스 상태 한번에 확인
```bash
echo "=== Docker 컨테이너 ===" && docker ps --format "table {{.Names}}\t{{.Status}}" && \
echo -e "\n=== Flask API ===" && curl -s http://localhost:5001/console/api/setup && \
echo -e "\n=== Next.js ===" && curl -s -o /dev/null -w "%{http_code}" http://localhost:3000 && \
echo -e "\n=== Ollama ===" && curl -s http://localhost:11434/api/tags | head -1
```

---

## 문제 해결

### 포트 충돌 해결
```bash
# 특정 포트를 사용하는 프로세스 확인
lsof -i :8080  # 예시: 8080 포트 확인

# 프로세스 종료
kill -9 <PID>
```

### 서비스 재시작
```bash
# Docker 서비스 재시작
docker compose -f docker-compose.middleware.yaml --profile weaviate -p dify restart

# Flask API 재시작
pkill -f flask
cd /Users/prismdata/Documents/entec/dify/api
uv run flask run --host 0.0.0.0 --port=5001 --debug

# Next.js 재시작  
pkill -f next
cd /Users/prismdata/Documents/entec/dify/web
NODE_OPTIONS='--max-old-space-size=4096' pnpm run dev
```

### 로그 확인
```bash
# Docker 로그
docker logs dify-db-1
docker logs dify-redis-1
docker logs dify-weaviate-1

# 백그라운드 프로세스 로그
tail -f flask.log
tail -f nextjs.log  
tail -f celery_worker.log
```

### 데이터베이스 초기화 (주의!)
```bash
cd /Users/prismdata/Documents/entec/dify/api

# 데이터베이스 완전 초기화 (모든 데이터 삭제됨)
uv run flask db downgrade base
uv run flask db upgrade
```

---

## 🚀 빠른 시작 스크립트

전체 서비스를 한 번에 시작하는 스크립트:

```bash
#!/bin/bash
# dify_start_all.sh

echo "🚀 Dify 전체 서비스 시작..."

# 1. Docker 미들웨어 시작
echo "📦 Docker 미들웨어 시작..."
cd /Users/prismdata/Documents/entec/dify/docker
docker compose -f docker-compose.middleware.yaml --profile weaviate -p dify up -d

# 2. Flask API 시작
echo "🐍 Flask API 시작..."
cd /Users/prismdata/Documents/entec/dify/api
nohup uv run flask run --host 0.0.0.0 --port=5001 --debug > flask.log 2>&1 &

# 3. Celery 워커 시작
echo "⚙️ Celery 워커 시작..."
nohup uv run celery -A app.celery worker -P gevent -c 1 --loglevel INFO -Q dataset,generation,mail,ops_trace,app_deletion,plugin,workflow_storage > celery_worker.log 2>&1 &

# 4. Next.js 시작
echo "🌐 Next.js 프론트엔드 시작..."
cd /Users/prismdata/Documents/entec/dify/web
nohup NODE_OPTIONS='--max-old-space-size=4096' pnpm run dev > nextjs.log 2>&1 &

# 5. Ollama 확인
echo "🤖 Ollama 상태 확인..."
if ! pgrep -f "ollama serve" > /dev/null; then
    echo "⚠️ Ollama가 실행되지 않았습니다. 수동으로 시작하세요: ollama serve"
fi

echo "✅ 모든 서비스 시작 완료!"
echo "📍 웹 인터페이스: http://localhost:3000"
echo "📍 API 서버: http://localhost:5001"

sleep 10
echo "🔍 서비스 상태 확인..."
docker ps --format "table {{.Names}}\t{{.Status}}"
```

스크립트 실행:
```bash
chmod +x dify_start_all.sh
./dify_start_all.sh
```

---

## 📞 추가 도움

- **공식 문서**: https://docs.dify.ai
- **GitHub**: https://github.com/langgenius/dify
- **Discord**: https://discord.gg/FngNHpbcY7

---

*이 가이드는 macOS ARM64 환경에서 테스트되었습니다. 다른 환경에서는 일부 명령어나 경로가 다를 수 있습니다.*
