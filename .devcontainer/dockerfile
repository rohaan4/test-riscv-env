# Set this for intel/amd chips
ARG CPU_TYPE="x86"
# Set this for apple m-series chips
# ARG CPU_TYPE="arm64"

FROM ubuntu:22.04 AS builder

# Consume build args
ARG CPU_TYPE

# Install general dependencies
RUN apt-get update && apt-get install -y \
    wget \
    git \
    clang \
    default-jre \
    && rm -rf /var/lib/apt/lists/*

# Clone mpact sim repo
RUN git clone https://github.com/google/mpact-riscv.git
WORKDIR /mpact-riscv

# Install dependencies for mpact sim
RUN if [ "$CPU_TYPE" = "arm64" ]; then \
        wget -O /usr/local/bin/bazelisk https://github.com/bazelbuild/bazelisk/releases/download/v1.27.0/bazelisk-linux-arm64 && \
        chmod +x /usr/local/bin/bazelisk; \
        ln -s /usr/local/bin/bazelisk /usr/local/bin/bazel; \
    elif [ "$CPU_TYPE" = "x86" ]; then \
        wget -O /usr/local/bin/bazelisk https://github.com/bazelbuild/bazelisk/releases/download/v1.27.0/bazelisk-linux-amd64 && \
        chmod +x /usr/local/bin/bazelisk; \
        ln -s /usr/local/bin/bazelisk /usr/local/bin/bazel; \
    else \
      echo "No valid cpu specified"; \
      exit 1; \
    fi

# Build the simulator
RUN if [ "$CPU_TYPE" = "arm64" ]; then \
        bazel build --cpu=darwin_arm64 --verbose_failures ...:all //riscv:rv32g_sim; \
    elif [ "$CPU_TYPE" = "x86" ]; then \
        bazel build --verbose_failures ...:all //riscv:rv32g_sim; \
    else \
      echo "No valid cpu specified"; \
      exit 1; \
    fi

FROM ubuntu:22.04 AS runner

# Install general dependencies
RUN apt-get update && apt-get install -y \
    wget \
    git \
    curl \
    build-essential \
    libreadline-dev \
    libgmp-dev \
    z3 \
    pkg-config \
    cmake \
    clang \
    lld \
    && rm -rf /var/lib/apt/lists/*

# Install prebuilt RISCV toolchain
# Note this toolchain only runs on x86 CPUs
RUN wget https://github.com/stnolting/riscv-gcc-prebuilt/releases/download/rv32i-131023/riscv32-unknown-elf.gcc-13.2.0.tar.gz
RUN mkdir riscv32-unknown-elf-gcc-13.2.0
RUN tar -xzvf riscv32-unknown-elf.gcc-13.2.0.tar.gz -C riscv32-unknown-elf-gcc-13.2.0
RUN rm -rf tar riscv32-unknown-elf.gcc-13.2.0.tar.gz
RUN mv riscv32-unknown-elf-gcc-13.2.0 /opt/riscv
RUN echo 'export PATH="/opt/riscv/bin:$PATH"' >> ~/.bashrc
# RUN source ~/.bashrc

# Copy over simulator from build stage
COPY --from=builder /mpact-riscv/bazel-bin/riscv/rv32g_sim /usr/local/bin/mpact_rv32g_sim
RUN chmod +x /usr/local/bin/mpact_rv32g_sim

# DEV set root password
RUN echo "root:root" | chpasswd

# Setup Xmake
RUN useradd -m -s /bin/bash xmake_user
USER xmake_user
WORKDIR /home/xmake_user
RUN curl -fsSL https://xmake.io/shget.text | bash
RUN echo 'source /home/xmake_user/.xmake/profile' >> /home/xmake_user/.bashrc
RUN /bin/bash -c "source /home/xmake_user/.xmake/profile && xmake --version"

# Add prebuilt toolchain to xmake user path
RUN echo 'export PATH="/opt/riscv/bin:$PATH"' >> /home/xmake_user/.bashrc
