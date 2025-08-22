LINUX/UBUNTU ROOTLESS AI SETUP: PODMAN WITH LLAMA.CPP AND FULL VULKAN BACKEND SUPPORT. ADDED OPEN WEBUI AND OPENHANDS
=====================================================================================================================
No sudo, no root, everything runs as your user! Now with integrated GPU support!

IMPORTANT: Before starting this guide, ensure your SSD is mounted to /mnt/ssd/
Also make sure you have podman set up. That's in the other README (PODMAN AI PET README)
This guide is supplementary, and is designed to go further than what ollama can do.
It is more private, can handle integrated gpus (it's stronger), and more complicated to set up.
The good news is that it fundamentally runs through podman so it is still quite simple to handle and monitor.

Enjoy!

#===================================================================
# INITIAL SETUP (one-time only) - PREPARING YOUR AI PET'S NEW HOME
#===================================================================


IMPORTANT! ENSURE YOUR SSD IS FORMATTED TO EXT4, NOT NTFS/EXFAT. DRIVER ISSUES MAY OCCUR IF NOT.


STEP 1: SSD MOUNTING
===========================================
# Create mount point with user ownership
sudo mkdir -p /mnt/ssd
sudo chown $USER:$USER /mnt/ssd

# Add to /etc/fstab (replace /dev/sda1 with your device):
# /dev/sda1 /mnt/ssd ext4 defaults,user,exec 0 2

# Mount without sudo (thanks to 'user' option)
mount /mnt/ssd

# Verify ownership (should show your username, not root)
ls -la /mnt/ssd


STEP 2: INSTALL PODMAN
===========================================
sudo apt update && sudo apt install -y podman


STEP 3: PODMAN CONFIG
===========================================
# Configure Podman storage location
mkdir -p ~/.config/containers
mkdir -p /mnt/ssd/podman

cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
driver = "overlay"
graphroot = "/mnt/ssd/podman/storage"
EOF

# Reset Podman to use new storage
podman system reset

# Fix missing policy.json
sudo mkdir -p /etc/containers
sudo tee /etc/containers/policy.json << 'EOF'
{
    "default": [{"type": "insecureAcceptAnything"}],
    "transports": {
        "docker": {
            "docker.io": [{"type": "insecureAcceptAnything"}]
        }
    }
}
EOF


#=============================================================
# BUILDING THE ENVIRONMENT
#=============================================================

STEP 1: MAKE DIRECTORY
===========================================
mkdir -p /mnt/ssd/podman/llama-vulkan/models

STEP 2: MAKE DOCKERFILE
===========================================
cd /mnt/ssd/podman/llama-vulkan
cat > Dockerfile << 'EOF'
ARG UBUNTU_VERSION=24.04

FROM docker.io/library/ubuntu:$UBUNTU_VERSION AS build

# Install build tools
RUN apt update && apt install -y git build-essential cmake wget

# Install Vulkan SDK and cURL
RUN wget -qO - https://packages.lunarg.com/lunarg-signing-key-pub.asc | apt-key add - && \
    wget -qO /etc/apt/sources.list.d/lunarg-vulkan-noble.list https://packages.lunarg.com/vulkan/lunarg-vulkan-noble.list && \
    apt update -y && \
    apt-get install -y vulkan-sdk libcurl4-openssl-dev curl

# Clone the repository
WORKDIR /app
RUN git clone https://github.com/ggml-org/llama.cpp.git . && \
    git submodule update --init --recursive

# Build it
RUN cmake -B build -DGGML_NATIVE=OFF -DGGML_VULKAN=1 -DLLAMA_BUILD_TESTS=OFF -DGGML_BACKEND_DL=ON -DGGML_CPU_ALL_VARIANTS=ON && \
    cmake --build build --config Release -j$(nproc)

RUN mkdir -p /app/lib && \
    find build -name "*.so" -exec cp {} /app/lib \;

RUN mkdir -p /app/full \
    && cp build/bin/* /app/full \
    && cp *.py /app/full \
    && cp -r gguf-py /app/full \
    && cp -r requirements /app/full \
    && cp requirements.txt /app/full \
    && cp .devops/tools.sh /app/full/tools.sh

## Base image
FROM ubuntu:$UBUNTU_VERSION AS base

RUN apt-get update \
    && apt-get install -y libgomp1 curl libvulkan-dev \
    && apt autoremove -y \
    && apt clean -y \
    && rm -rf /tmp/* /var/tmp/* \
    && find /var/cache/apt/archives /var/lib/apt/lists -not -name lock -type f -delete \
    && find /var/cache -type f -delete

COPY --from=build /app/lib/ /app

### Server target
FROM base AS server

ENV LLAMA_ARG_HOST=0.0.0.0

COPY --from=build /app/full/llama-server /app

WORKDIR /app

HEALTHCHECK CMD [ "curl", "-f", "http://localhost:8080/health" ]

ENTRYPOINT [ "/app/llama-server" ]

EOF


STEP 3: BUILD IMAGE FROM DOCKERFILE
===========================================
cd /mnt/ssd/podman/llama-vulkan
podman build -t llama-vulkan .


STEP 4: CREATE INTERNAL NETWORK
===========================================
podman network create llm-network


STEP 5: START SOCKET
===========================================
# Launch socket
systemctl --user is-active --quiet podman.socket || systemctl --user start podman.socket


#=============================================================
# STARTING CONTAINERS
#=============================================================

STEP 1: RUN LLAMA.CPP WITH GPU ACCELERATION
===========================================
# Run llama.cpp in server mode maybe remove -t, unsure
podman run -d -t\
  --name llama-cpp-server \
  --network llm-network \
  --device /dev/dri:/dev/dri \
  --device /dev/kfd:/dev/kfd \
  --group-add video \
  --group-add $(getent group render | cut -d: -f3) \
  --security-opt label=disable \
  --security-opt seccomp=unconfined \
  --privileged \
  -v /mnt/ssd/podman/models/gguf:/models:ro \
  -p 127.0.0.1:8080:8080 \
  -e HSA_OVERRIDE_GFX_VERSION=10.3.0 \
  -e GGML_VULKAN_DEVICE=0 \
  llama-vulkan \
  /home/llamauser/llama.cpp/build/bin/llama-server \
    -m /models/Jan-v1-4B-Q8_0.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -ngl 999 \
    -t 4 \
    -c 2048 \
    -b 512 \
    --flash-attn \
    --mlock

STEP 2: RUN OPEN WEBUI
======================
podman run -d -t\
  --name open-webui \
  --network llm-network \
  --security-opt=no-new-privileges \
  -p 127.0.0.1:3000:8080 \
  -v open-webui:/app/backend/data \
  --restart always \
  ghcr.io/open-webui/open-webui:main

# Configure Open WebUI:
# 1. Go to http://localhost:3000
# 2. Create account (first user becomes admin)
# 3. User - Settings - Admin settings - Connections
# 4. Under OPEN AI setting:
#    - OPEN AI API Base URL: http://llama-cpp-server:8080/v1
#    - OPEN AI API Key: not-needed
# 5. Your model appears in the chat dropdown (check actual model name with curl http://localhost:8080/v1/models)


STEP 3: RUN OPENHANDS (ROOTLESS)
=================================
# Create state directory
mkdir -p /mnt/ssd/podman/openhands-state
mkdir -p ~/.openhands

# Pull images
podman pull docker.all-hands.dev/all-hands-ai/openhands:0.47

podman pull pull docker.all-hands.dev/all-hands-ai/runtime:0.47-nikolaik

# Run OpenHands with rootless Podman socket
podman run -d \
  --name openhands \
  --network llm-network \
  --security-opt label=disable \
  -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.47-nikolaik \
  -e LOG_ALL_EVENTS=true \
  -v $XDG_RUNTIME_DIR/podman/podman.sock:/var/run/docker.sock \
  -v /mnt/ssd/podman/openhands-state:/.openhands-state \
  -p 127.0.0.1:3001:3000 \
  -v ~/.openhands:/.openhands \
  docker.all-hands.dev/all-hands-ai/openhands:0.47


## Configure OpenHands for llama-cpp
# 1. Go to http://localhost:3001
# 2. Settings → LLM Configuration
# 3. Configure:
#    - Provider: OpenAI Compatible
#    - API Base URL: http://llama-cpp-server:8080/v1
#    - API Key: not-needed
#    - Model: openai//models/devstral2507.gguf (or your choice)

# OpenHands container control
podman stop openhands
podman rm openhands


#=============================================================
# STOP AND REMOVE CONTAINERS
#=============================================================
# Stops and removes all containers
podman stop -a
podman rm -a

# Stops and removes server
podman stop llama-cpp-server
podman rm llama-cpp-server

# Check (you should see nothing)
podman ps -a


#=============================================================
# GPU and CPU MONITORING
#=============================================================
# GPU monitoring (AMD)
radeontop

# CPU monitoring  
htop

# Container-specific stats
podman stats llama-cpp-server

#=============================================================
# IMPORTANT NOTES
#=============================================================
There are many more commands for managing and using podman. Many of these are in the PODMAN readme setup, down the bottom of that file. Please consult that AND official documentation if you get stuck. Frankly, everything is brand new and supercedes information you'll find in github etc. This guide will probably be redundant in a month.



ROOTLESS LIMITATIONS IN OPENHANDS
==================================

Openhands is currently still in development. I use both 0.47 of the client and runtime, but these can probably be updated to support a weird podman socket error I found in the making of this guide. Assume this version is already outdated. This is a problem directly relating to this being a rootless project.

When OpenHands runs in rootless mode, both the OpenHands container and its 
runtime containers operate without root privileges. The runtime container 
(all-hands-ai/runtime) runs as a regular user inside the rootless Podman 
environment, maintaining the security benefits of rootless operation.

WILL FAIL:
- apt/yum install (system packages)
- Binding to ports below 1024
- Modifying system files (/etc/*)
- Creating system services

WILL WORK:
- pip install (Python packages → ~/.local)
- npm install (Node packages → ./node_modules)
- Running servers on high ports (3000, 8080, etc)
- Building and running containers (via Podman)
- All normal development tasks

The AI will adapt and work around these limitations automatically! How fucked is that!


