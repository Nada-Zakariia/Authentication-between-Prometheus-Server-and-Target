# Authentication-between-Prometheus-Server-and-target.
Implementing Encryption and Authentication using self-signed Certificate.
## 1- Create certificate for Node Exporter
### - Create node_exporter directory in /etc
  ```sh
  sudo mkdir /etc/node_exporter
  ```
### - Generate a 2048-bit RSA private key
  ```sh       
  sudo openssl genrsa -out /etc/node_exporter/node_exporter.key 2048
  ```
### - Generate a certificate
  ```sh
  sudo openssl req -new -key /etc/node_exporter/node_exporter.key  -out /etc/node_exporter/node_exporter.csr
  ```
### - Self-sign the certificate with the private key
  ```sh        
  sudo openssl x509 -req -days 365 -in /etc/node_exporter/node_exporter.csr -signkey /etc/node_exporter/node_exporter.key -out /etc/node_exporter/node_exporter.crt
  ```
### - Create targetconfig.yml file for crt and key
  ```yaml        
          tls_server_config:
            cert_file: node_exporter.crt
            key_file: node_exporter.key
  ```
### - Set the ownership of the node_exporter directory 
   ```sh         
   sudo chown -R node_exporter:node_exporter /etc/node_exporter
   ```
### - Update the systemd service of node_exporter (/etc/systemd/system/node_exporter.service)
  ```ini
  [Unit]
  Description=Prometheus Node Exporter
  Wants=network-online.target
  After=network-online.target
  [Service]
  User=node_exporter
  Group=node_exporter
  Type=simple
  ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/targetconfig.yml"
  [Install]
  WantedBy=multi-user.target
  ```

### - Reload the daemon and restart the node_exporter
  ```sh
  sudo systemctl daemon-reload
  sudo systemctl restart node_exporter
  ```
### - Verify the metrics locally  
  ```sh
  curl https://localhost:9100/metrics 
  curl -k https://localhost:9100/metrics
  ```
### - Copy the node_exporter.crt to prometheus server
  ```sh
  scp /etc/node_exporter/node_exporter.crt root@192.168.1.11:/etc/prometheus
  ```
### - Change the ownership of node_expoter.crt file to prometheus 
  ```sh
  sudo chown prometheus:prometheus /etc/promtheus/node_exporter.crt
  ```
### - Update the prometheus configuration file with the certificate of target
  ```yml
  - job_name: 'Linux Server'
    scheme: https
    tls_config:
       ca_file: /etc/prometheus/node_exporter.crt
       insecure_skip_verify: true
    static_configs:
       - targets: ['192.168.1.12:9100']
  ```
----------------------------------------------------------------
## 2- Authentication
### - Generate hashed passwd using apache2-utils package
  ```sh
  sudo apt-get update && sudo apt install apache2-utils -y
  htpasswd -nBC 12 "" | tr -d ':\n'
  ```
### - Update targetconfig.yml with some lines
 ```yaml
 basic_auth_users:
   prometheus: $2y$12$jAcN21HKQtAekuZnyklezuvIuQXShduzId4rXRaa73aZH0fev2kzq
 ``` 
### - Restart node_exporter service
 ```sh
 sudo systemctl restart node_exporter
 ```
### - Update prometheus.yml with username and password

 ```yaml
   - job_name: 'Linux Server'
     scheme: https
     basic_auth:
       username: prometheus
       password: node
     tls_config:
       ca_file: /etc/prometheus/node_exporter.crt
       insecure_skip_verify: true
 ```
### - Restart prometheus service
 ```sh
 sudo systemctl restart prometheus
 ```


 
