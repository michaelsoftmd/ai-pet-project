FULLY ROOTLESS PODMAN LLM SETUP WITH LLAMA.CPP AND FULL VULKAN BACKEND SUPPORT

#===================================================================
# INITIAL SETUP (one-time only) - PREPARING YOUR AI PET'S NEW HOME
#===================================================================


# IMPORTANT! ENSURE YOUR SSD IS FORMATTED TO EXT4, NOT NTFS/EXFAT. DRIVER ISSUES MAY OCCUR IF NOT.


# 1. SSD MOUNTING
#--------------------
# Create mount point with user ownership
sudo mkdir -p /mnt/ssd
sudo chown $USER:$USER /mnt/ssd

# Add to /etc/fstab (replace /dev/sda1 with your device):
# /dev/sda1 /mnt/ssd ext4 defaults,user,exec 0 2

# Mount without sudo (thanks to 'user' option)
mount /mnt/ssd

# Verify ownership (should show your username, not root)
ls -la /mnt/ssd


# 2. INSTALL PODMAN
#--------------------
sudo apt update && sudo apt install -y podman
# Fedora: sudo dnf install -y podman
# Arch: sudo pacman -S podman


# 3. PODMAN CONFIGURATION
#--------------------
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


STEP 3: BUILD IMAGE
===========================================
podman build -t llama-vulkan .

STEP 4: CREATE INTERNAL NETWORK
===========================================
podman network create llm-network


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
  --security-opt label=disable \
  -v /mnt/ssd/podman/llama-vulkan/models:/models \
  -p 127.0.0.1:8080:8080 \
  llama-vulkan \
  /home/llamauser/llama.cpp/build/bin/llama-server \
    -m /models/devstral2507.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -ngl 99 \
    -c 4096

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
# 3. Settings → Admin → Connections IT'S BEHIND, NOT THE POPUP LOOKING THING
# 4. Add connection TO OPEN AI SETTING THING:
#    - OPEN AI API Base URL: http://llamafile:8080/v1
#    - OPEN AI API Key: not-needed
# 5. Your model appears in the chat dropdown (check actual model name with curl http://localhost:8080/v1/models)


STEP 3: RUN OPENHANDS (ROOTLESS)
=================================

# Create state directory
mkdir -p /mnt/ssd/podman/openhands-state

# Pull images
podman pull docker.all-hands.dev/all-hands-ai/runtime:0.17-nikolaik
podman pull docker.all-hands.dev/all-hands-ai/openhands:0.17

# Run OpenHands with rootless Podman socket
podman run -d \
  --name openhands \
  --network llm-network \
  -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.17-nikolaik \
  -e LOG_ALL_EVENTS=true \
  -e DOCKER_HOST=unix:///var/run/docker.sock \
  -v $XDG_RUNTIME_DIR/podman/podman.sock:/var/run/docker.sock \
  -v /mnt/ssd/podman/openhands-state:/.openhands-state \
  -p 127.0.0.1:3001:3000 \
  --add-host host.docker.internal:host-gateway \
  docker.all-hands.dev/all-hands-ai/openhands:0.17

# Configure OpenHands:
# 1. Go to http://localhost:3001
# 2. Settings → LLM Configuration
# 3. Provider: OpenAI Compatible
# 4. API Base URL: http://llamafile:8080/v1
# 5. API Key: not-needed
# 6. Model: (use the model name returned by your llamafile)


