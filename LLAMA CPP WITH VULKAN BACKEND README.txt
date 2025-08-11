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
FROM docker.io/library/ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Install base dependencies
RUN apt-get update && apt-get install -y \
    build-essential cmake git wget curl libcurl4-openssl-dev \
    mesa-vulkan-drivers mesa-utils \
    libvulkan1 vulkan-tools \
    libgl1-mesa-glx libgl1-mesa-dri libglx-mesa0 \
    python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Download and install Vulkan SDK
RUN wget -qO- https://packages.lunarg.com/lunarg-signing-key-pub.asc | tee /etc/apt/trusted.gpg.d/lunarg.asc && \
    wget -qO /etc/apt/sources.list.d/lunarg-vulkan-jammy.list https://packages.lunarg.com/vulkan/lunarg-vulkan-jammy.list && \
    apt-get update && \
    apt-get install -y vulkan-sdk && \
    rm -rf /var/lib/apt/lists/*

ENV VULKAN_SDK=/usr
ENV VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json:/usr/share/vulkan/icd.d/radeon_icd.i686.json
ENV VK_LAYER_PATH=/usr/share/vulkan/explicit_layer.d

RUN useradd -m -s /bin/bash llamauser
USER llamauser
WORKDIR /home/llamauser

RUN git clone https://github.com/ggerganov/llama.cpp.git && \
    cd llama.cpp && \
    mkdir build && \
    cd build && \
    cmake .. -DGGML_VULKAN=ON && \
    cmake --build . --config Release -j$(nproc)

ENV PATH="/home/llamauser/llama.cpp/build/bin:${PATH}"
CMD ["/bin/bash"]

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
# Run llama.cpp in server mode
podman run -d -t \
  --name llama-cpp-server \
  --network llm-network \
  --userns=keep-id \
  --device /dev/dri \
  --device /dev/kfd \
  --group-add video \
  --group-add keep-groups \
  --security-opt label=disable \
  -v /mnt/ssd/podman/llama-vulkan/models:/models \
  llama-vulkan \
  /home/llamauser/llama.cpp/build/bin/llama-server \
    -m /models/devstral2507.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -ngl 25 \
    -t 8 \
    -c 4096 \
    --flash-attn

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


