# 🏗️Harbor-registry-trivy
Harbor docker image registry with Trivy Image scanning.
Here are the simple steps to install the latest version of Harbor (v2.14.4) on your headless Debian 13 server.

### Step 1: Install Prerequisites
Harbor requires Docker Compose V2 and OpenSSL. Ensure they are installed on your system.

```bash
sudo apt update
sudo apt install -y docker-compose-plugin openssl
```

*Verify the installation by running `docker compose version`. If it returns an error, ensure the Docker Compose plugin is correctly installed from the official Docker repository.*

### Step 2: Download and Extract Harbor
Download the latest offline installer and extract it into `/opt/harbor`.

```bash
cd /tmp
wget https://github.com/goharbor/harbor/releases/download/v2.14.4/harbor-offline-installer-v2.14.4.tgz
sudo mkdir -p /opt/harbor
sudo tar -xvf harbor-offline-installer-v2.14.4.tgz -C /opt/harbor --strip-components=1
cd /opt/harbor
```

### Step 3: Configure Harbor
Copy the template configuration file and edit it to match your IP, port, and disable HTTPS (since you are using HTTP).

```bash
sudo cp harbor.yml.tmpl harbor.yml
sudo nano harbor.yml
```

Make the following three changes inside the file:

1. **Change the hostname** to your IP:
   ```yaml
   hostname: 10.0.0.140
   ```
2. **Change the HTTP port** to 7081 (find the `http:` section):
   ```yaml
   http:
     port: 7081
   ```
3. **Disable HTTPS** by adding a `#` at the beginning of the `https:` line and the 4 lines below it:
   ```yaml
   # https related config
   #https:
   #  port: 443
   #  certificate: /your/certificate/path
   #  private_key: /your/private/key/path
   ```

*Save and exit (Press `Ctrl+O`, `Enter`, then `Ctrl+X`).*

### Step 4: Run the Installer
Run the installation script. This will load the necessary Docker images and start the containers. This step may take 1–2 minutes.

```bash
sudo ./install.sh --with-trivy
```

### Step 5: Verify and Access
Check if all containers are healthy:
```bash
sudo docker compose ps
```

Once the containers are running, you can access Harbor in your web browser:
*   **URL:** `http://10.0.0.140:7081`
*   **Username:** `admin`
*   **Password:** `Harbor12345`

---

### Optional: Auto-Start on Reboot (Systemd Service)
By default, Harbor does not automatically start if the server reboots. You can fix this by creating a simple systemd service.

1. Create the service file:
```bash
sudo tee /etc/systemd/system/harbor.service > /dev/null <<EOF
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5s
WorkingDirectory=/opt/harbor
ExecStart=/usr/bin/docker compose up -d --remove-orphans
ExecStop=/usr/bin/docker compose stop

[Install]
WantedBy=multi-user.target
EOF
```

2. Enable the service to start on boot:
```bash
sudo systemctl daemon-reload
sudo systemctl enable harbor
```

### Troubleshooting Tips
*   **Firewall:** If you cannot reach the UI, ensure your firewall allows traffic on port 7081 (e.g., `sudo ufw allow 7081/tcp`).
*   **Container Logs:** If the web interface doesn't load, check the logs for errors:
    ```bash
    sudo docker compose logs harbor-core
    ```
*   **Restarting:** If you need to restart the registry manually, use `sudo docker compose restart` from the `/opt/harbor` directory.

### Next Steps for Client Configuration
To push images to this registry from another machine, you must configure the Docker daemon on that machine to trust your insecure registry:

1. Edit `/etc/docker/daemon.json` on the **client machine**:
   ```json
   {
     "insecure-registries": ["10.0.0.140:7081"]
   }
   ```
2. Restart Docker on the client: `sudo systemctl restart docker`
3. Login to the registry: `docker login 10.0.0.140:7081`
