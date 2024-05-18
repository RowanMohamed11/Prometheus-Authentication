# Authentication/Encryption in Prometheus
This project demonstrates how to set up Node Exporter with TLS encryption and basic authentication, and configure Prometheus to scrape metrics from the Node Exporter over HTTPS.

## Prerequisites

- Node Exporter installed on the target server.
- Prometheus installed on a separate server.
- `openssl` and `apache2-utils` (or equivalent) installed on the respective servers.

## Step-by-Step Setup

### 1. Generate Self-Signed Certificates

On the Node Exporter server, run the following command to generate self-signed certificates using OpenSSL:

```sh
sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=US/ST=California/L=Oakland/O=MyOrg/CN=localhost" -addext "subjectAltName = DNS:localhost"
```

### 2. Create Directory and Move Certificates
Create a directory for Node Exporter and move the generated certificates:
```
sudo mkdir /etc/node_exporter
sudo mv node_exporter.* /etc/node_exporter/
```

### 3. Create Configuration File
Create a config.yml file in /etc/node_exporter with the following content:
```
tls_server_config:
  cert_file: node_exporter.crt
  key_file: node_exporter.key
```
### 4. Update Permissions
Update the permissions for the Node Exporter directory:
```
sudo chown -R node_exporter:node_exporter /etc/node_exporter
```

### 5. Update Systemd Service
Update the Node Exporter systemd service file to include the TLS configuration:
```
sudo vi /etc/systemd/system/node_exporter.service
```
Add the following content:
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/config.yml"

[Install]
WantedBy=multi-user.target
```
### 6. Reload Daemon and Restart Node Exporter
Reload the systemd daemon and restart the Node Exporter service:
```
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
```

### 7. Copy Certificate to Prometheus Server
Copy the node_exporter.crt file from the Node Exporter server to the Prometheus server:
```
scp ubuntu@node-exporter.example.com:/etc/node_exporter/node_exporter.crt /etc/prometheus/
```
### 8. Update Permissions on Prometheus Server
Update the permissions of the copied certificate:

```
sudo chown prometheus:prometheus /etc/prometheus/node_exporter.crt
```
### 9. Update Prometheus Configuration
Update the Prometheus configuration to scrape metrics over HTTPS. Edit prometheus.yml to include the following configuration:

```
scrape_configs:
  - job_name: 'node_exporter'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/node_exporter.crt
      insecure_skip_verify: true  # Only for self-signed certificates
    static_configs:
      - targets: ['node-exporter.example.com:9100']
```
### 10. Restart Prometheus
Restart the Prometheus service to apply the changes:
```
sudo systemctl restart prometheus
```
### 11. Set Up Basic Authentication
Generate a password hash using apache2-utils:
```
sudo apt-get update && sudo apt install apache2-utils -y
htpasswd -nBC 12 "" | tr -d ':\n'
```
Update the config.yml file on the Node Exporter server to include basic authentication:
```
basic_auth_users:
  prometheus: <hashed_password>
```
### 12. Restart Node Exporter
Restart the Node Exporter service:
```
sudo systemctl restart node_exporter
```
### 13. Update Prometheus Configuration with Authentication
Update the Prometheus configuration to include basic authentication:
```
scrape_configs:
  - job_name: 'node_exporter'
    scheme: https
    basic_auth:
      username: prometheus
      password: your_password
    tls_config:
      ca_file: /etc/prometheus/node_exporter.crt
      insecure_skip_verify: true
    static_configs:
      - targets: ['node-exporter.example.com:9100']
```
### 14. Restart Prometheus
Restart the Prometheus service:
```
sudo systemctl restart prometheus
```
## Verification
Check the status of the Node Exporter and Prometheus services to ensure they are running correctly and metrics are being scraped over HTTPS.
```
sudo systemctl status node_exporter
sudo systemctl status prometheus
```
You should now have Node Exporter running with TLS and basic authentication, and Prometheus scraping metrics over HTTPS.
### You can check prometheus in a browser using the server’s IP address:
```
http://<PROMETHEUS_SERVER_IP>:9090
``` 
## Conclusion
By following these steps, you have secured the communication between Node Exporter and Prometheus using TLS and basic authentication. This setup enhances the security of your monitoring infrastructure.







