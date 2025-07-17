# Podman + Ollama + AnythingLLM Setup Guide for Linux Mint, Ubuntu
# Store everything on external SSD at /mnt/ssd/
# Runs from localhost
# Runs rootless in its own container
# No root privileges
# Is still capable of agentic function


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
# RUNNING CONTAINERS - WHERE YOUR AI PET IS KEPT
#=============================================================

# Pull container images. It is important to use the long name with podman, not the short name as with docker, as podman sometimes has issues with docker image pulls.
podman pull docker.io/ollama/ollama:latest
podman pull docker.io/mintplexlabs/anythingllm:latest

# Create network for container communication
podman network create llm-network

# Create Ollama model directory
mkdir -p /mnt/ssd/podman/ollama


#=======================================================================
# STARTING OLLAMA AND ANYTHINGLLM - HOSTING AND TALKING TO YOUR AI PET
#=======================================================================
# First off, ensure your SSD is mounted at /mnt/ssd/
# It will be correctly mounted if there are files inside. If not, run:
mount /mnt/ssd
mount -a

# Next, ensure the internal network is running. This allows Ollama and AnythingLLM to communicate (important)
podman network create llm-network

# Run Ollama
podman run -d \
  --name ollama \
  --network llm-network \
  --security-opt=no-new-privileges \
  -v /mnt/ssd/podman/ollama:/root/.ollama \
  -p 127.0.0.1:11434:11434 \
  docker.io/ollama/ollama:latest

# Run AnythingLLM
podman run -d \
  --name anythingllm \
  --network llm-network \
  --security-opt=no-new-privileges \
  -v anythingllm_storage:/app/server/storage \
  -p 127.0.0.1:3001:3001 \
  -e STORAGE_DIR="/app/server/storage" \
  docker.io/mintplexlabs/anythingllm:latest

# Pull AI model. Substitute examplemodel with the filename from Ollama. For example 'dolphin-llama3:70b'
podman exec ollama ollama pull examplemodel

# Common models: llama3.2, mistral, phi3, deepseek-coder, nomic-embed-text
# Check available models at: https://ollama.com/library

# Pull model without running container (temporary container method)
podman run --rm -v /mnt/ssd/podman/ollama:/root/.ollama ollama/ollama pull examplemodel

# Verify everything is running
podman ps
podman exec ollama ollama list


#=================================================================
# ANYTHINGLLM CONFIGURATION - YOUR COMMUNICATOR WITH YOUR AI PET
#=================================================================
# 1. Open browser: http://localhost:3001
# 2. Create admin account
# 3. Settings → LLM Preference → Ollama
# 4. Ollama base URL: http://ollama:11434
# 5. Select model → Save

# Congratulations! That's it. You've done it. Once in that web window you are able to access your LLMs. 


#==================================================
# SHUTTING DOWN - PUTTING YOUR AI PET AWAY SAFELY
#==================================================
# When you are done interacting with your LLMs run the below two lines of code to stop both Ollama and AnythingLLM and remove their hosted containers:
podman stop ollama anythingllm
podman rm ollama anythingllm

# All the data from the conversations you had with the LLMs during that session is preserved on the SSD inside volumes.
# You can delete these if you wish for security purposes, or back them up somehow, whatever you like. Use podman's purge command.

#==================================================
# HOW TO RESTART AFTER CLOSING
#==================================================
# Whenever you want to restart your containers, simply run the script from 'STARTING OLLAMA AND ANYTHINGLLM' above.

# If you run into permission issues, check troubleshooting.

# You may also need to recreate llm-network, which is the internal means by which Ollama talks to AnythingLLM:
podman network create llm-network


#=============================================================
# MANAGING CONTAINERS - HANDLING YOUR AI PET (MORE IN-DEPTH)
#=============================================================

# CONTAINER LIFECYCLE COMMANDS
#--------------------
# Stop containers
podman stop ollama anythingllm
podman stop -a                      # Stop all running containers

# Start containers
podman start ollama anythingllm
podman start -a                     # Start all stopped containers

# Restart containers
podman restart ollama
podman restart anythingllm

# Remove containers (preserves data in volumes)
podman stop ollama anythingllm
podman rm ollama anythingllm
podman rm -a                        # Remove all stopped containers

# Force remove running container
podman rm -f ollama

# Pause/unpause containers (freeze processes)
podman pause ollama
podman unpause ollama


# OLLAMA MODEL MANAGEMENT - THE MODELS ARE YOUR AI PETS. ENJOY THEM. YOU CAN HAVE LIKE A POKEMON TEAM.
#--------------------
# List all downloaded models
podman exec ollama ollama list

# Show model details
podman exec ollama ollama show examplemodel

# Pull new model. Substitute examplemodel with the filename from Ollama.
podman exec ollama ollama pull examplemodel

# Delete model
podman exec ollama ollama rm examplemodel

# Copy/duplicate a model with new name
podman exec ollama ollama cp examplemodel mynewmodel

# Run model interactively
podman exec -it ollama ollama run examplemodel "Hello, how are you?"

# Stop running model (if stuck)
podman exec ollama pkill ollama


#=============================================================
# DATA LOCATIONS - DATA LOCATIONS
#=============================================================
# Ollama models: /mnt/ssd/podman/ollama/
# AnythingLLM data: /mnt/ssd/podman/storage/volumes/anythingllm_storage/
# Podman images: /mnt/ssd/podman/storage/
# Podman volumes: /mnt/ssd/podman/storage/volumes/


#=============================================================
# TROUBLESHOOTING - YOUR AI PET HAS PROBLEMS?
#=============================================================
# Systemd reload after ANY config changes (fstab, containers.conf)
sudo systemctl daemon-reload

# List active containers
podman ps

# Stops everything and cleans up after itself
podman stop ollama anythingllm
podman rm ollama anythingllm

# Resets and deletes your containers so you can start fresh.
podman system reset

# Permission denied after reboot
sudo chown -R $USER:$USER /mnt/ssd/podman
podman ps  # Should work now

# This one may help if you run into any strange issues pulling images.
podman pull docker.io/postgres:14

# Container won't start - check logs
podman logs ollama
podman logs anythingllm

# Test Ollama is accessible from AnythingLLM
podman exec anythingllm curl http://ollama:11434/api/tags

# Check disk space
df -h /mnt/ssd
du -sh /mnt/ssd/podman/ollama/models/

# Update containers to latest version
podman pull docker.io/ollama/ollama:latest
podman pull docker.io/mintplexlabs/anythingllm:latest
# Then remove and recreate containers


#=============================================================
# AUTO-START WITH SYSTEMD (Optional)
#=============================================================

# Generate systemd service files
podman generate systemd --new --name ollama > ~/.config/systemd/user/ollama.service
podman generate systemd --new --name anythingllm > ~/.config/systemd/user/anythingllm.service

# Enable services to start on boot
systemctl --user enable ollama.service
systemctl --user enable anythingllm.service

# Enable lingering (keeps services running when logged out)
sudo loginctl enable-linger $USER

# Manage services
systemctl --user start ollama
systemctl --user stop ollama
systemctl --user status ollama
systemctl --user restart ollama


#=============================================================
# MODEL SIZE GUIDELINES
#=============================================================

# Model sizes and RAM requirements:
# - 1B models:  ~1-2GB RAM
# - 3B models:  ~3-4GB RAM  
# - 7B models:  ~7-8GB RAM
# - 13B models: ~13-16GB RAM
# - 30B models: ~30-32GB RAM
# - 70B models: ~70GB+ RAM

# Note: Actual RAM usage increases with context length
# Add 2-8GB for active conversations depending on context size


#=============================================================
# USEFUL COMMANDS THAT AFFECT YOUR AI PET'S CAGE
#=============================================================

# SYSTEM COMMANDS
#--------------------
# Mount/unmount SSD (no sudo needed with 'user' in fstab)
mount /mnt/ssd
umount /mnt/ssd

# Another mount option to try, especially if you've messed with fstab
mount /mnt/ssd
mount -a

# Check what's listening on ports
ss -tlnp | grep -E "3001|11434"
netstat -tulpn | grep -E "3001|11434"    # Alternative

# System info
podman info
podman version
podman system df                    # Show disk usage

# Clean up system
podman system prune                 # Remove unused data
podman system prune -a              # Remove all unused data (careful!)
podman system prune --volumes       # Also remove unused volumes



#EVERYTHING FROM HERE BELOW IS QUITE BORING. USE ONLY IF REQUIRED. THESE ARE VARIOUS MANAGEMENT COMMANDS FOR SPECIFIC TROUBLESHOOTING.


# CONTAINER INSPECTION. 'podman ps' is actually quite useful though.
#--------------------
# View logs
podman logs ollama
podman logs anythingllm --tail 50
podman logs -f ollama               # Follow logs in real-time

# Inspect container details
podman inspect ollama
podman inspect ollama | grep -i memory    # Check memory limits

# View resource usage
podman stats                        # Real-time stats for all containers
podman stats --no-stream           # One-time snapshot

# List all containers (including stopped)
podman ps -a

# List with custom format
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Show container processes
podman top ollama

# Show port mappings
podman port ollama
podman port anythingllm



# NETWORK MANAGEMENT
#--------------------
# List networks
podman network ls

# Inspect network
podman network inspect llm-network

# See which containers use a network
podman network inspect llm-network | grep -A5 Containers

# Create new network
podman network create mynetwork
podman network create --internal isolated-net   # No external access

# Remove network (must remove containers first)
podman network rm llm-network

# Connect running container to network
podman network connect llm-network mycontainer

# Disconnect from network
podman network disconnect llm-network mycontainer



# IMAGE MANAGEMENT
#--------------------
# List images
podman images

# Remove unused images
podman image prune

# Remove specific image
podman rmi docker.io/ollama/ollama:old-version

# Show image history
podman history ollama/ollama

# Save image to file (for backup)
podman save -o ollama-backup.tar ollama/ollama

# Load image from file
podman load -i ollama-backup.tar


# VOLUME MANAGEMENT
#--------------------
# List volumes
podman volume ls

# Inspect volume
podman volume inspect anythingllm_storage

# Show volume size
podman volume inspect anythingllm_storage | grep -i size

# Backup volume (while container stopped)
podman run --rm -v anythingllm_storage:/source:ro -v $(pwd):/backup alpine tar czf /backup/anythingllm-backup.tar.gz -C /source .

# Create new volume
podman volume create my_new_volume

# Remove unused volumes
podman volume prune
