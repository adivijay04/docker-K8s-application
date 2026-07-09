# syntax=docker/dockerfile:1

# ---------- Stage 1: build dependencies ----------
FROM python:3.12-slim AS builder

# Build tools needed only for compiling wheels (e.g. pydantic-core, uvloop, etc.)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Copy only the requirements file first to maximize layer caching
COPY requirements.txt .

# Build wheels instead of installing directly, so stage 2 doesn't need build tools at all
RUN pip install --no-cache-dir --upgrade pip \
    && pip wheel --no-cache-dir --wheel-dir /build/wheels -r requirements.txt


# ---------- Stage 2: minimal runtime image ----------
FROM python:3.12-slim AS runtime

# Metadata
LABEL maintainer="you@example.com" \
      description="FastAPI service runtime image"

# Runtime env hygiene
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# curl is needed for the HEALTHCHECK; strip apt lists immediately after
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create a non-root user/group with no login shell and no home dir clutter
RUN groupadd --gid 1000 appuser \
    && useradd --uid 1000 --gid appuser --shell /usr/sbin/nologin --no-create-home appuser

WORKDIR /app

# Bring in prebuilt wheels from the builder stage and install from them (no compiler needed here)
COPY --from=builder /build/wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels requirements.txt

# Copy application code last so code changes don't bust the dependency layer cache
COPY --chown=appuser:appuser ./app ./app

# Drop privileges
USER appuser

EXPOSE 8000

# Adjust the path if your health endpoint differs; this assumes GET /health returns 200
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Assumes your ASGI app object is `app` inside app/main.py — adjust if different
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
