# Self-Hosted n8n on Proxmox VE

## Project Overview

This project documents the deployment of a self-hosted **n8n Community Edition** instance using Proxmox VE, Debian Linux, Docker, and Tailscale.

The objective was to build a lightweight, self-hosted workflow automation server that runs on local hardware and can be accessed securely from remote devices without exposing the service directly to the public internet.

The deployment uses a Debian Linux LXC container hosted on Proxmox VE, with Docker Compose managing the n8n application.

## Architecture

```text
             Remote Device
                   |
                   |
              Tailscale
                   |
                   |
           Private HTTPS Access
                   |
                   |
             Tailscale Serve
                   |
                   |
              Docker Engine
                   |
                   |
              n8n Application
                   |
                   |
             Debian LXC
                   |
                   |
               Proxmox VE
                   |
                   |
              Local Hardware
```

## Technologies Used

- **Proxmox VE:** Virtualisation and container management.
- **Debian 12:** Linux operating system.
- **Docker Engine:** Application containerisation.
- **Docker Compose:** Application deployment and management.
- **n8n Community Edition:** Workflow automation platform.
- **Tailscale:** Private network connectivity.
- **Tailscale Serve:** Secure HTTPS access.

## Hardware Overview

The deployment uses modest, repurposed hardware.

| Component | Specification |
|---|---|
| Host platform | Compact desktop computer |
| Processor | Intel Celeron-class CPU |
| Memory | 4 GB RAM |
| Storage | SSD |
| Hypervisor | Proxmox VE |

The hardware limitations influenced the decision to use an LXC container instead of a full virtual machine.

## 1. Creating the Debian Container

Updated the Proxmox container templates:

```bash
pveam update
```

Downloaded a Debian 12 standard container template.

Created an unprivileged LXC container with:
- Two allocated CPU cores.
- 2 GB allocated RAM.
- 512 MB swap.
- Dedicated container storage.
- Automatic network configuration.
- Automatic startup enabled.

Enabled the container features required by Docker:

```bash
pct set <CONTAINER_ID> --features nesting=1,keyctl=1
```

Started and accessed the container:

```bash
pct start <CONTAINER_ID>
pct enter <CONTAINER_ID>
```

## 2. Installing Docker

Updated Debian:

```bash
apt update && apt upgrade -y
```

Installed the required dependencies:

```bash
apt install -y ca-certificates curl gnupg
```

Added Docker's official package repository and GPG signing key.

Installed Docker Engine, Docker CLI, containerd, Buildx and Docker Compose:

```bash
apt update

apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verified the installation:

```bash
docker --version
docker compose version
systemctl is-active docker
```

## 3. Deploying n8n

Created an application directory:

```bash
mkdir -p /opt/n8n
cd /opt/n8n
```

Generated a dedicated encryption key:

```bash
printf 'N8N_ENCRYPTION_KEY=%s\n' "$(openssl rand -hex 32)" > /opt/n8n/.env

chmod 600 /opt/n8n/.env
```

Created a Docker Compose configuration:

**File:** `compose.yaml`

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=Africa/Johannesburg
      - TZ=Africa/Johannesburg
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

Deployed the application:

```bash
docker compose up -d
```

Verified that the application was running:

```bash
docker ps
```

The deployment uses a persistent Docker volume to retain n8n application data.

## 4. Configuring Tailscale

Installed Tailscale inside the Debian container:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Enabled and started the service:

```bash
systemctl enable tailscaled
systemctl start tailscaled
```

Connected the server to the Tailscale network:

```bash
tailscale up
```

Authenticated through the official Tailscale login process.

Verified connectivity:

```bash
tailscale status
tailscale ip -4
```

Tailscale was also installed on the remote Windows device and connected using the same account.

## 5. Troubleshooting the TUN Device

One of the main challenges was getting Tailscale to operate inside an unprivileged Proxmox LXC container.

Initially, Tailscale failed because the container could not access the Linux TUN device.

The error indicated that `/dev/net/tun` was unavailable.

The solution was to pass through the existing TUN device from the Proxmox host into the container.

The required device permissions and bind mount were configured in the container configuration.

The relevant configuration uses the Linux device major and minor numbers for the TUN device.

After restarting the container, the device became accessible.

Tailscale was restarted and successfully became active.

**What I learned:**

- How Linux TUN devices work.
- Why VPN software needs virtual network interfaces.
- How Proxmox handles device passthrough.
- How to troubleshoot system services using `systemctl` and `journalctl`.
- How unprivileged containers handle device permissions.

## 6. Configuring HTTPS with Tailscale Serve

Enabled HTTPS certificate support through the Tailscale administration console.

Configured Tailscale Serve to forward secure HTTPS requests to the local n8n application:

```bash
tailscale serve --bg http://127.0.0.1:5678
```

Tailscale provided a private HTTPS address for the server.

The Windows device could then access the n8n dashboard through the private Tailscale network.

No public router port forwarding was required.

## Final Deployment Status

| Component | Status |
|---|---|
| Proxmox VE | Installed |
| Debian LXC | Running |
| Docker Engine | Running |
| Docker Compose | Installed |
| n8n Community Edition | Running |
| Tailscale | Connected |
| Tailscale Serve | Running |
| HTTPS access | Working |
| Remote dashboard access | Working |

## Security Considerations

Security was an important part of this deployment.

The following principles were applied:

- Remote access is provided through a private Tailscale network.
- The n8n dashboard is not intentionally exposed to the public internet.
- No router port forwarding was required.
- An application encryption key was generated and stored separately.
- The n8n data volume persists independently of the application container.
- Credentials and environment files must not be committed to source control.
- Infrastructure-specific identifiers and network details are excluded from this repository.

## Future Improvements

Planned improvements include:

- Automated n8n backups.
- Backup and recovery testing.
- Monitoring CPU, memory and storage usage.
- Testing automatic recovery after a Proxmox reboot.
- Exploring additional n8n integrations.
- Building practical workflow automation projects.
- Evaluating hardware upgrades as workloads increase.

## What This Project Taught Me

This deployment provided hands-on experience with:

1. Linux server administration.
2. Proxmox virtualisation and LXC containers.
3. Docker and Docker Compose.
4. Application deployment and persistent storage.
5. Linux networking and TUN devices.
6. VPN configuration and private remote access.
7. HTTPS reverse proxying.
8. Troubleshooting container permissions and system services.

---

**Project Status: Successfully deployed.**

A personal homelab project focused on learning infrastructure management, containerisation, networking, security and workflow automation.
