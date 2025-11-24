# Multi-stage build를 활용한 최적화된 Dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app

# 의존성 파일 복사
COPY requirements.txt .

# 독립된 venv 생성 → user 권한 문제 없음 + runtime에서 깔끔
# 불필요한 빌드 도구 제거로 이미지 경량화
RUN python -m venv /opt/venv && \
    /opt/venv/bin/pip install --upgrade pip setuptools wheel && \
    /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# 실행 단계
FROM python:3.11-slim

# 보안을 위한 non-root 유저 생성
RUN groupadd -r appuser && useradd -r -g appuser -u 1000 appuser

WORKDIR /app

# builder에서 생성된 lightweight venv만 복사
COPY --from=builder /opt/venv /opt/venv

# 애플리케이션 코드 복사
COPY --chown=appuser:appuser main.py .

# PATH 환경변수에 pip 설치 경로 추가
ENV PATH="/opt/venv/bin:$PATH"

# 애플리케이션 포트
EXPOSE 8000

# non-root 유저로 전환
USER appuser

# docker compose에 헬스체크를 설정되어 있으므로 주석 처리
# # 헬스체크 설정 (curl 대신 python 사용)
# HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
#     CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# 애플리케이션 실행
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
