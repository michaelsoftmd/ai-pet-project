ROOTLESS AI SETUP: LLAMAFILE + OPEN WEBUI + OPENHANDS for LINUX MINT/UBUNTU
=============================================================================
No sudo, no root, everything runs as your user! Now with integrated GPU support!

IMPORTANT: Before starting this guide, ensure your SSD is mounted to /mnt/ssd/
Also make sure you have podman set up. That's in the other README (PODMAN AI PET README)
This guide is supplementary, and is designed to go further than what ollama can do.
It is more private, can handle integrated gpus (it's stronger), and more complicated to set up.
The good news is that it fundamentally runs through podman so it is still quite simple to handle and monitor.

Enjoy!



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
wget https://huggingface.co/mistralai/Devstral-Small-2505/resolve/main/Devstral-Small-2505-Q4_K_M.gguf

# Convert GGUF to llamafile
cat > ../config/devstral.args << 'EOF'
-m
Devstral-Small-2505-Q4_K_M.gguf
--host
0.0.0.0
--port
8080
--nobrowser
-c
128000
...
EOF

cp ../bin/llamafile devstral.llamafile
../bin/zipalign -j0 devstral.llamafile Devstral-Small-2505-Q4_K_M.gguf ../config/devstral.args
chmod +x devstral.llamafile

# Create container image
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

# Enable your user's podman socket (NO SUDO!)
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

podman run -d \
  --name llamafile \
  --network llm-network \
  --security-opt=no-new-privileges \
  -v /mnt/ssd/podman/llamafile/models:/models:z \
  -p 127.0.0.1:8080:8080 \
  llamafile-runner \
  "cd /models && ./devstral.llamafile"


STEP 5: RUN OPEN WEBUI
======================

podman run -d \
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
# 3. Settings → Admin → Connections
# 4. Add connection:
#    - API Base URL: http://llamafile:8080/v1
#    - API Key: not-needed
# 5. Your model appears in the chat dropdown


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
  -v $XDG_RUNTIME_DIR/podman/podman.sock:/var/run/docker.sock:z \
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
# 6. Model: gpt-3.5-turbo (or what your model reports)


ROOTLESS START/STOP SCRIPTS
===========================

cat > /mnt/ssd/podman/start.sh << 'EOF'
#!/bin/bash
MODEL=${1:-devstral.llamafile}

echo "Starting AI services with model: $MODEL"
echo "Running completely rootless!"

# Ensure user podman socket is running
systemctl --user start podman.socket

# Create network if needed
podman network create llm-network 2>/dev/null

# Start model
podman run -d \
  --name llamafile \
  --network llm-network \
  --security-opt=no-new-privileges \
  -v /mnt/ssd/podman/llamafile/models:/models:z \
  -p 127.0.0.1:8080:8080 \
  llamafile-runner \
  "cd /models && ./$MODEL"

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
  -v $XDG_RUNTIME_DIR/podman/podman.sock:/var/run/docker.sock:z \
  -v /mnt/ssd/podman/openhands-state:/.openhands-state \
  -p 127.0.0.1:3001:3000 \
  --add-host host.docker.internal:host-gateway \
  docker.all-hands.dev/all-hands-ai/openhands:0.17

echo ""
echo "Services available at:"
echo "  Open WebUI: http://localhost:3000"
echo "  OpenHands: http://localhost:3001"
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


ROOTLESS LIMITATIONS IN OPENHANDS
==================================

When OpenHands tries to do something requiring root, it will fail gracefully:

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


HOW TO USE
==========

# Start everything
/mnt/ssd/podman/start.sh

# Start with different model
/mnt/ssd/podman/start.sh mistral.llamafile

# Stop everything
/mnt/ssd/podman/stop.sh

# Check podman socket
systemctl --user status podman.socket


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

# TRY ALSO READING PODMAN AI PET README GUIDE, OR CONSULTING OFFICIAL DOCUMENTATION


ADDING MORE MODELS
==================

# Download more GGUFs
cd /mnt/ssd/podman/llamafile/models
wget [huggingface-url-to-gguf]

# Convert to llamafile
cat > ../config/newmodel.args << 'EOF'
-m
newmodel.gguf
--host
0.0.0.0
--port
8080
--nobrowser
-c
4096
...
EOF

cp ../bin/llamafile newmodel.llamafile
../bin/zipalign -j0 newmodel.llamafile newmodel.gguf ../config/newmodel.args
chmod +x newmodel.llamafile


WHAT YOU GET
============

✓ NO root/sudo required anywhere
✓ Everything runs as your regular user
✓ All data on /mnt/ssd/ with user permissions
✓ Llamafile on port 8080
✓ Open WebUI on port 3000
✓ OpenHands on port 3001

This is as secure as it gets while still being functional!
