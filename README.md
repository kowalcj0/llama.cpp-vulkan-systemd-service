Auto-updating llama.cpp vulkan systemd service
----------------------------------------------

A set of systemd units and a Bash update script that deploy and maintain a self-updating [llama.cpp](https://github.com/ggml-org/llama.cpp) OpenAI-compatible server using Vulkan GPU acceleration on Linux. The update script polls the `ggml-org/llama.cpp` GitHub releases, downloads the latest Vulkan x64 binary, and atomically swaps a `current` symlink so the server always runs the newest release with minimal downtime. If the new binary fails to start, the script rolls back to the previous version automatically. A daily systemd timer keeps things current, while the service unit provides security hardening, GPU device access, and automatic restart on failure.

## First-Time Setup (Quick Start)

```shell
# Install dependencies (jq is the only non-standard one)
apt update && sudo apt install -y curl jq

# Create directories
mkdir -p /models /opt/llama-server /var/cache/llama-updater /var/lib/llama-updater

# Copy your models to /models, e.g.
cp Qwen3.6-27B-UD-Q6_K_XL.gguf /models/

# If you're running this service in an uprivileged LXC container with GPU passthrough,
# then skip the user creation step below
# Create the service user (render group needed for GPU/Vulkan access)
useradd -r -s /usr/sbin/nologin -d /opt/llama-server llama
usermod -aG render llama
chown llama:llama /opt/llama-server

# Copy the script and make it executable
cp update-llama-server /usr/local/bin/
chmod +x /usr/local/bin/update-llama-server

# Edit llama-server.service to match your setup:
#   --model  → path to your .gguf model file
#   --host   → bind address (0.0.0.0 = all interfaces, 127.0.0.1 = local only)
#   --port   → listen port (default 9000)
#   --ctx-size → context window size (262144 = 256k tokens, adjust for your VRAM)
#   any other flags as needed
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


## Extras

### amdgpu_top

`amdgpu_top` is a great tool for monitoring AMD GPUs.
Unfortunately, it's not (yet) available on Debian 13 (Trixie) provided by Proxmox.

To install it manually, run:
```shell
wget https://github.com/Umio-Yasuno/amdgpu_top/releases/download/v0.11.5/amdgpu-top_without_gui_0.11.5-1_amd64.deb
apt install ./amdgpu-top_without_gui_0.11.5-1_amd64.deb
```


### Unprivileged LXC container

An example configuration of a LXC container with GPU passthrough:
```ini
arch: amd64
cores: 40
dev0: /dev/dri/renderD128
features: nesting=1
hostname: llamacpp
memory: 163840
mp0: /root/models,mp=/models
net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=BC:24:11:44:0B:2B,ip=dhcp,type=veth
ostype: debian
rootfs: local-zfs:subvol-136-disk-0,size=150G
swap: 512
unprivileged: 1
lxc.cgroup2.cpuset.cpus: 40-79
```

