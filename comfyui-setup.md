# ComfyUI Docker Setup Guide

## Overview
This guide walks through deploying ComfyUI in a Docker container with GPU support, persistent
storage for models and outputs, and automatic directory initialization.

## Directory Structure
comfyui-docker/  
├── docker-compose.yml  
├── Dockerfile  
├── entrypoint.sh  
├── models/ (created automatically)  
├── output/ (created automatically)  
├── input/ (created automatically)  
└── custom_nodes/ (created automatically)  

## Step-by-Step Setup
***Must run capsule code command as this uses multiple terminal sessions***  
<ins>capsule code -u [Machine Name]</ins>

### 1. Create Project Directory
```bash
mkdir -p ~/comfyui-docker
cd ~/comfyui-docker
```
### 2. Create the Dockerfile
Create a file named Dockerfile with the following content:

```bash
FROM python:3.12-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
  git \ 
  wget \ 
  curl \ 
  vim \ 
  nano \ 
  htop \ 
  libgl1-mesa-glx \ 
  libglib2.0-0 \ 
  libsm6 \ 
  libxext6 \ 
  libxrender-dev \ 
  libgomp1 \ 
  && rm -rf /var/lib/apt/lists/*

# Create user with sudo access
RUN useradd -m -s /bin/bash comfyuser && \ 
  echo "comfyuser ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Switch to user
USER comfyuser
WORKDIR /home/comfyuser

# Clone ComfyUI
RUN git clone https://github.com/comfyanonymous/ComfyUI.git

WORKDIR /home/comfyuser/ComfyUI

# Install PyTorch with CUDA support
RUN pip3 install --no-cache-dir torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# Install ComfyUI requirements
RUN pip3 install --no-cache-dir -r requirements.txt

# Save default models, custom_nodes, and input structure before volume mounts hide them
RUN cp -r models models_default && cp -r custom_nodes custom_nodes_default && cp -r input input_default

# Copy entrypoint script
COPY --chown=comfyuser:comfyuser entrypoint.sh /home/comfyuser/entrypoint.sh

RUN chmod +x /home/comfyuser/entrypoint.sh

# Expose ComfyUI port
EXPOSE 8188

# Use entrypoint to initialize directories then start ComfyUI
ENTRYPOINT ["/home/comfyuser/entrypoint.sh"]
```
### 3. Create the docker-compose.yml
Create a file named docker-compose.yml:

```bash
services:
  comfyui:
    build: .
    container_name: comfyui
    ports:
      - "8188:8188"
    volumes:
      # Persist models, outputs, and custom nodes
      - ./models:/home/comfyuser/ComfyUI/models
      - ./output:/home/comfyuser/ComfyUI/output
      - ./input:/home/comfyuser/ComfyUI/input
      - ./custom_nodes:/home/comfyuser/ComfyUI/custom_nodes
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    stdin_open: true
    tty: true

    restart: unless-stopped
```
### 4. Create the Entrypoint Script
Create a file named entrypoint.sh:

```bash
#!/bin/bash

# If models directory is empty, copy the default structure from ComfyUI
if [ -z "$(ls -A /home/comfyuser/ComfyUI/models 2>/dev/null)" ]; then
    echo "Initializing models directory structure..."
    cp -r /home/comfyuser/ComfyUI/models_default/. /home/comfyuser/ComfyUI/models/
fi

# Same for custom_nodes
if [ -z "$(ls -A /home/comfyuser/ComfyUI/custom_nodes 2>/dev/null)" ]; then
    echo "Initializing custom_nodes directory..."
    cp -r /home/comfyuser/ComfyUI/custom_nodes_default/. /home/comfyuser/ComfyUI/custom_nodes/
fi

# Same for input (for 3d folder)
if [ ! -d "/home/comfyuser/ComfyUI/input/3d" ]; then
    echo "Initializing input directory structure..."
    cp -r /home/comfyuser/ComfyUI/input_default/. /home/comfyuser/ComfyUI/input/ 2>/dev/null || mkdir -p /home/comfyuser/ComfyUI/input/3d
fi

# Run ComfyUI
exec python3 main.py --listen 0.0.0.0
```
### 5. Build the Docker Image
```bash
docker compose build
```
This will take several minutes as it:
* Downloads the base Python image
* Installs system dependencies
* Clones ComfyUI from GitHub
* Installs PyTorch with CUDA 12.1 support
* Installs all ComfyUI requirements

### 6. Start the Container
```bash
docker compose up -d

# If the command fails, run:
docker compose ps
# Copy ID from ps command and paste to end of kill command
docker kill [ID]
# Run again
docker compose up -d
```

### 7. Verify It's Running
Check container status:  
```bash
docker compose ps
```
View logs:
```bash
docker compose logs -f
```
Press `Ctrl+C` to exit log viewing.

### 8. Access ComfyUI
#### Option A: Using VS Code Port Forwarding (Recommended for Remote Development)
If you're connected to the server via VS Code Remote SSH:
1. Automatic Port Forwarding (Easiest):
* VS Code should automatically detect port 8188 and show a notification
* Click the notification to open in your browser
* Or look for the pop-up in the bottom-right corner
2. Manual Port Forwarding:
* Open the Ports tab in VS Code (usually at the bottom panel, next to Terminal)
* Click "Forward a Port" button
* Enter 8188 and press Enter
* Right-click on the forwarded port and select "Open in Browser"
3. Alternatively, you can also:
* Press F1 or Ctrl+Shift+P (Windows/Linux) / Cmd+Shift+P (Mac)
* Type "Forward a Port"
* Press the “World” Icon

## Common Operations

### Stop the Container
```bash
docker compose down
```

### Restart the Container
```bash
docker compose restart
```

### View Logs
```bash
docker compose logs -f comfyui
```

### Access Container Shell
```bash
docker exec -it comfyui bash
```

### Rebuild After Changes
```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```
