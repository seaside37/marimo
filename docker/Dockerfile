# syntax=docker/dockerfile:1.12
# Stage 1: Build stage
FROM python:3.13-slim AS builder

# Install Node.js
RUN apt-get update && apt-get install -y curl \
    && curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
    && apt-get install -y nodejs \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY . .

# Build frontend
WORKDIR /app/frontend
RUN npm install -g pnpm
RUN pnpm install

WORKDIR /app
RUN pnpm run build

# Ensure marimo/_static directory exists
RUN mkdir -p /app/marimo/_static \
    && cp -r /app/frontend/dist/* /app/marimo/_static/

# Install build tools
RUN pip install --upgrade --no-cache-dir pip setuptools wheel build

# Build wheel
RUN python -m build --wheel

# Stage 2: Base stage for all target images
FROM python:3.13-slim AS base
# Make `uv` and `uvx` available in the PATH for all target images
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

ENV MARIMO_SKIP_UPDATE_CHECK=1
ENV UV_SYSTEM_PYTHON=1

# Copy only the built wheel from builder stage
COPY --from=builder /app/dist/*.whl /tmp/

# Install the wheel without build dependencies
RUN pip install --no-cache-dir /tmp/*.whl \
    && rm -f /tmp/*.whl

ENV PORT=8080
EXPOSE $PORT

ENV HOST=0.0.0.0

# -slim entry point
FROM base AS slim
CMD marimo edit --no-token -p $PORT --host $HOST

# -data entry point
FROM base AS data
RUN uv pip install --no-cache-dir marimo[recommended,lsp]
CMD marimo edit --no-token -p $PORT --host $HOST

# -sql entry point, extends -data
FROM data AS sql
RUN uv pip install --no-cache-dir marimo[recommended,lsp,sql] pandas numpy sqlalchemy psycopg2-binary
CMD marimo edit --no-token -p $PORT --host $HOST
