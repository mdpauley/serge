# ---------------------------------------
# Base image for dragonflydb
FROM ghcr.io/dragonflydb/dragonfly:v1.7.1 as dragonfly

# ---------------------------------------
# Build frontend
FROM node:20-bookworm-slim as frontend

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY ./web/package.json /usr/src/app
COPY ./web/package-lock.json /usr/src/app
RUN npm ci

COPY ./web/ /usr/src/app/
WORKDIR /usr/src/app/web/
RUN npm run build

# ---------------------------------------
# Build environment
FROM nvidia/cuda:12.2.0-devel-ubuntu22.04 as build

# We need to set the host to 0.0.0.0 to allow outside access
ENV HOST 0.0.0.0

# Install dependencies
RUN apt-get update && apt-get upgrade -y \
    && apt-get install -y git build-essential \
    python3 python3-pip gcc wget \
    ocl-icd-opencl-dev opencl-headers clinfo \
    libclblast-dev libopenblas-dev \
    libxml2 dumb-init \
    libnvidia-opencl-dev \
    && mkdir -p /etc/OpenCL/vendors && echo "libnvidia-opencl.so.1" > /etc/OpenCL/vendors/nvidia.icd

COPY . .


# Set ENV
ENV NODE_ENV='production'
ENV TZ=Etc/EDT 

ENV FORCE_CMAKE=1
ENV CUDA_DOCKER_ARCH=all
ENV LLAMA_CUBLAS=1

# setting build related env vars
ENV CUDA_DOCKER_ARCH=all
ENV CMAKE_ARGS="-DLLAMA_CUBLAS=ON -DLLAMA_NATIVE=ON -DLLAMA_FMA=OFF -DLLAMA_F16C=OFF -DLLAMA_AVX2=OFF"


# Install depencencies
RUN python3 -m pip install --upgrade pip pytest cmake scikit-build setuptools fastapi uvicorn sse-starlette pydantic-settings starlette-context

# Install llama-cpp-python (build with cuda)
RUN pip install --no-index --find-links=/usr/local/bin/wheels llama-cpp-python

# ---------------------------------------
# Runtime environment
FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04 as release

WORKDIR /usr/src/app

# Copy artifacts
COPY --from=dragonfly /usr/local/bin/dragonfly /usr/local/bin/dragonfly
COPY --from=frontend /usr/src/app/web/build /usr/src/app/api/static/
COPY --from=build /usr/local/bin/wheels /usr/local/bin/wheels
ADD ./api /usr/src/app/
ADD ./scripts/deploy.sh /usr/src/app/


# Install api dependencies
RUN pip install --no-cache-dir ./api \
    && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* \
    && chmod 755 /usr/src/app/deploy.sh /usr/local/bin/dragonfly

EXPOSE 8008
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/bin/bash", "-c", "/usr/src/app/deploy.sh"]
