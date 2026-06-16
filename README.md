Auto-updating llama.cpp vulkan systemd service
----------------------------------------------

## First-Time Setup (Quick Start)

```shell
# Install dependencies (jq is the only non-standard one)
apt update && sudo apt install -y curl jq

# Create directories
mkdir -p /models /opt/llama-server /var/cache/llama-updater /var/lib/llama-updater

# Copy your models to /models, e.g.
cp Qwen3.6-27B-UD-Q6_K_XL.gguf /models/

# Create the service user
useradd -r -s /usr/sbin/nologin -d /opt/llama-server llama

# Copy the script and make it executable
cp update-llama-server /usr/local/bin/
chmod +x /usr/local/bin/update-llama-server

# Copy the systemd service (edit ExecStart first!)
cp llama-server.service /etc/systemd/system/

# Copy the llama updater service
cp llama-updater.timer /etc/systemd/system/
cp llama-updater.service /etc/systemd/system/

systemctl daemon-reload

# Run the update script once to download the initial version
/usr/local/bin/update-llama-server

# Start the server
systemctl enable --now llama-server

# Start the updater
systemctl enable --now llama-updater.timer

# Check status of both services
systemctl status llama-updater.timer
systemctl status llama-server
```

