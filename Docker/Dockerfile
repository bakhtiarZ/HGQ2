# ========= builder: build Verilator from source =========
FROM ubuntu:24.04 AS verilator-builder
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ARG DEBIAN_FRONTEND=noninteractive
ARG VERILATOR_VERSION=5.026

RUN apt-get update && apt-get install -y --no-install-recommends \
    git ca-certificates wget curl build-essential autoconf automake help2man \
    libtool pkg-config g++ bison libfl2 libfl-dev zlib1g zlib1g-dev \
    python3 make cmake ninja-build ccache \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /build
RUN git clone --depth 1 --branch v${VERILATOR_VERSION} https://github.com/verilator/verilator.git
WORKDIR /build/verilator
RUN autoconf && ./configure && make -j"$(nproc)" && make install DESTDIR=/out


# ========= final image =========
FROM ubuntu:24.04
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG DEBIAN_FRONTEND=noninteractive
ARG PYTHON_VERSION=3.12
ARG USERNAME=dev
ARG HOST_UID=1000
ARG HOST_GID=1000
ARG JAX_PLATFORMS=cpu
ARG PY_EXTRAS=""   # e.g. "test" or "dev"

ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONDONTWRITEBYTECODE=1

# --- System dependencies ---
RUN apt-get update && apt-get install -y --no-install-recommends \
    python${PYTHON_VERSION} python3-pip python3-venv \
    git curl wget bash-completion nano vim tmux rsync zip unzip \
    htop ripgrep fd-find tree ca-certificates \
    build-essential pkg-config cmake ninja-build ccache bison \
    gtkwave \
 && rm -rf /var/lib/apt/lists/*

RUN ln -s /usr/bin/fdfind /usr/local/bin/fd || true

# Copy Verilator from builder
COPY --from=verilator-builder /out/ /

# --- Environment variables for Keras/JAX ---
ENV PATH="/usr/local/bin:${PATH}" \
    KERAS_BACKEND=jax \
    JAX_PLATFORMS=${JAX_PLATFORMS} \
    MKL_NUM_THREADS=1 \
    OPENBLAS_NUM_THREADS=1

# --- Optional non-root user (commented out per your preference) ---
# RUN groupadd -g ${HOST_GID} ${USERNAME} && \
#     useradd -m -s /bin/bash -u ${HOST_UID} -g ${HOST_GID} ${USERNAME}
# USER ${USERNAME}

# --- Python + venv setup ---
ENV VENV_PATH=/opt/venv
RUN python${PYTHON_VERSION} -m venv ${VENV_PATH}
ENV PATH="${VENV_PATH}/bin:${PATH}"

RUN pip install --upgrade pip setuptools wheel

WORKDIR /workspace

# Copy metadata first for caching, then rest of repo
COPY pyproject.toml README.md ./
COPY . /workspace

# --- hls4ml >=1.2 fallback (PyPI → GitHub) and pin via constraints ---
RUN set -eux; \
    python -m pip install --upgrade pip wheel; \
    if python -m pip install "hls4ml>=1.2" --pre; then \
      echo "Using PyPI hls4ml >= 1.2"; \
    else \
      python -m pip install "hls4ml @ git+https://github.com/fastmachinelearning/hls4ml@main"; \
    fi; \
    python - <<'PY'
import importlib.metadata as m, pathlib, sys
v = m.version('hls4ml')
pathlib.Path('/tmp/constraints.txt').write_text(f'hls4ml=={v}\n')
print(f"Pinned hls4ml -> {v}", file=sys.stderr)
PY


# --- Install project (with optional extras) honoring constraints ---
# The --pre flag allows pip to accept the 1.2.0.dev… pre-release
RUN if [ -n "${PY_EXTRAS}" ]; then \
      python -m pip install -e ".[${PY_EXTRAS}]" --pre -c /tmp/constraints.txt; \
    else \
      python -m pip install -e . --pre -c /tmp/constraints.txt; \
    fi

CMD [ "bash" ]
