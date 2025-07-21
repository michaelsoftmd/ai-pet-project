ROOTLESS AI SETUP: LLAMAFILE + OPEN WEBUI + OPENHANDS for LINUX MINT/UBUNTU
=============================================================================
No sudo, no root, everything runs as your user! Now with integrated GPU support!

IMPORTANT: Before starting this guide, ensure your SSD is mounted to /mnt/ssd/
Also make sure you have podman set up. That's in the other README (PODMAN AI PET README)
This guide is supplementary, and is designed to go further than what ollama can do.
It is more private, can handle integrated gpus (it's stronger), and more complicated to set up.
The good news is that it fundamentally runs through podman so it is still quite simple to handle and monitor.

Enjoy!

Doesn't work on my GPU. It should work on yours if yours is supported by CLBlast. Try a dockerfile with ROCm support.


STEP 1: SET UP LLAMAFILE
========================

# Create directories
mkdir -p /mnt/ssd/podman/llamafile/{models,bin,config}

# Download llamafile tools
cd /mnt/ssd/podman/llamafile/bin
wget https://github.com/Mozilla-Ocho/llamafile/releases/download/0.9.3/llamafile-0.9.3.zip
unzip llamafile-0.9.3.zip
cp llamafile-0.9.3/bin/* ./
rm -rf llamafile-0.9.3*
chmod +x *

# Download your preferred GGUF model(s)
cd /mnt/ssd/podman/llamafile/models

# Example: Download Devstral for coding
wget https://huggingface.co/mistralai/Devstral-Small-2507/resolve/main/devstral2507.gguf

# Create simple container image
cd /mnt/ssd/podman/llamafile
cat > Dockerfile << 'EOF'
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /models
EXPOSE 8080
ENTRYPOINT ["/bin/bash", "-c"]
EOF

podman build -t llamafile-runner .


STEP 2: ENABLE ROOTLESS PODMAN SOCKET
======================================

# Enable your user's podman socket - no root
systemctl --user enable --now podman.socket

# Verify it's running
systemctl --user status podman.socket

# The socket is at: $XDG_RUNTIME_DIR/podman/podman.sock
# Usually: /run/user/1000/podman/podman.sock


STEP 3: CREATE NETWORK
======================

podman network create llm-network


STEP 4: RUN YOUR CHOSEN MODEL
==============================

# First, ensure your container can see the Hugging Face .gguf
# It uses GPU by '--group-add video \' and IMPORTANT! you can edit the performance options with '-ngl -t -b -c' as below. These directly control how much GPU the LLM uses.
podman run -d -t \
  --name llamafile \
  --network llm-network \
  --device /dev/dri \
  --device /dev/kfd \
  --group-add video \
  -v /mnt/ssd/podman/llamafile/models:/models \
  -v /mnt/ssd/podman/llamafile/bin:/bin/llamafile \
  -p 127.0.0.1:8080:8080 \
  llamafile-runner \
  "/bin/llamafile/llamafile -m /models/devstral2507.gguf --host 0.0.0.0 --port 8080 -ngl 20 -t 8 -b 256 -c 4096"


STEP 5: RUN OPEN WEBUI
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


STEP 6: RUN OPENHANDS (ROOTLESS)
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


ROOTLESS START/STOP SCRIPTS
===========================

cat > /mnt/ssd/podman/start.sh << 'EOF'
#!/bin/bash
MODEL=${1:-devstral2507.llamafile}

echo "Starting AI services with model: $MODEL"
echo "Running completely rootless!"

# Convert to .elf extension
MODEL_ELF="${MODEL%.llamafile}.llamafile.elf"

# Check if .elf exists
if [ ! -f "/mnt/ssd/podman/llamafile/models/$MODEL_ELF" ]; then
    echo "Error: $MODEL_ELF not found!"
    echo "Did you forget to assimilate? Run: ./$MODEL --assimilate"
    exit 1
fi

# Ensure user podman socket is running
systemctl --user start podman.socket

# Create network if needed
podman network create llm-network 2>/dev/null

# Remove existing containers if they exist
podman rm -f llamafile 2>/dev/null || true
podman rm -f open-webui 2>/dev/null || true
podman rm -f openhands 2>/dev/null || true

# Start model (using assimilated .elf with AMD GPU)

# Start Open WebUI
podman run -d \
  --name open-webui \
  --network llm-network \
  --security-opt=no-new-privileges \
  -p 127.0.0.1:3000:8080 \
  -v open-webui:/app/backend/data \
  --restart always \
  ghcr.io/open-webui/open-webui:main

# Start OpenHands (rootless)
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

echo ""
echo "Services available at:"
echo "  Open WebUI: http://localhost:3000 (container port 8080 mapped to host port 3000)"
echo "  OpenHands: http://localhost:3001 (container port 3000 mapped to host port 3001)"
echo "  Model API: http://localhost:8080/v1"
echo ""
echo "All running as user: $USER (no root!)"
EOF

chmod +x /mnt/ssd/podman/start.sh

# Stop EVERYTHING and CLEAN UP AFTERWARDS
cat > /mnt/ssd/podman/stop.sh << 'EOF'
#!/bin/bash
podman stop llamafile open-webui openhands
podman rm llamafile open-webui openhands
echo "All services stopped"
EOF

chmod +x /mnt/ssd/podman/stop.sh

CONVERT GGUF TO LLAMAFILE
==================================
# Create in config directory
cat > ../config/devstral2507.args << 'EOF'
-m
devstral2507.gguf
--host
0.0.0.0
--port
8080
--nobrowser
-ngl
30
-t
8
-c
4096
-b
256
EOF

# Copy it as .args for zipalign
cp ../config/devstral2507.args .args

# Then proceed with zipalign
cp ../bin/llamafile devstral2507.llamafile
../bin/zipalign -j0 devstral2507.llamafile devstral2507.gguf .args
chmod +x devstral2507.llamafile


ROOTLESS LIMITATIONS IN OPENHANDS
==================================

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

The AI will adapt and work around these limitations automatically!


TROUBLESHOOTING ROOTLESS SETUP
===============================

# If OpenHands can't connect to Podman socket
echo $XDG_RUNTIME_DIR  # Should show /run/user/1000
ls -la $XDG_RUNTIME_DIR/podman/podman.sock

# Test Podman socket
curl --unix-socket $XDG_RUNTIME_DIR/podman/podman.sock http://localhost/version

# View OpenHands logs
podman logs openhands

# If socket not found, restart it
systemctl --user restart podman.socket

# If you get "exec format error" - did you assimilate?
cd /mnt/ssd/podman/llamafile/models
file devstral.llamafile.elf  # Should show "ELF 64-bit"

# TRY ALSO READING PODMAN AI PET README GUIDE, OR CONSULTING OFFICIAL DOCUMENTATION


ADDING MORE MODELS
==================

# Download more GGUFs
cd /mnt/ssd/podman/llamafile/models
wget [huggingface-url-to-gguf]

# Convert to llamafile
# IMPORTANT: The filename in the args MUST match your actual GGUF filename!
cat > ../config/newmodel.args << 'EOF'
-m
your-exact-gguf-filename.gguf
--host
0.0.0.0
--port
8080
--nobrowser
-c
4096
--ctx-size
4096
--batch-size
512
--threads
8
--mlock
--no-mmap
EOF

cp ../bin/llamafile newmodel.llamafile
../bin/zipalign -j0 newmodel.llamafile your-exact-gguf-filename.gguf ../config/newmodel.args
chmod +x newmodel.llamafile

# ASSIMILATE. THIS MAY NOT WORK WITH YOUR LLAMAFILE VERSION. RUN AS GGUF IF NOT.
./newmodel.llamafile --assimilate
echo "Created newmodel.llamafile.elf for use in containers"




GPU SUPPORT - ONCE I HAVE INSTALLED ROCM
=============================================

This setup is configured for AMD integrated GPUs (like AMD 680M). The settings use:
- `-ngl 30`: Offloads 30 layers to GPU (moderate usage)
- `-t 8`: Uses 8 CPU threads
- `-c 8192`: 8K context size (good for long conversations)
- `-b 256`: Batch size 256 (balanced performance)
- `--gpu amd`: AMD GPU backend

Flash Attention (--flash-attn) can improve performance but may not be available
in all llamafile builds. If you get an error, just remove it.

To modify GPU settings, edit the podman run commands to change:
- `-ngl N`: Number of layers on GPU (0-999)
- `-t N`: Number of CPU threads (1-16)
- `-c N`: Context size in tokens
- `-b N`: Batch size for processing


WHAT YOU GET
============

✓ NO root/sudo required anywhere
✓ Everything runs as your regular user
✓ All data on /mnt/ssd/ with user permissions
✓ Llamafile on port 8080
✓ Open WebUI on port 3000
✓ OpenHands on port 3001
✓ Simple container setup
✓ GPU support if available through CLBlast

This is as secure and simple as it gets!
