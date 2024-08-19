# Prerequisites

- MacOS as the primary operating system
- UTM as the virtual machine software
- CentOS 9 Stream as the OS on the VM
- Internet access to download necessary packages
<br>
<br>

# Steps

## Hypervisor Installation

1. Download and install UTM on MacOS.
2. Create a new virtual machine in UTM with the following settings:
   - Select CentOS 9 Stream ISO as the source.
   - Allocate at least 2GB RAM and 2 CPUs for the VM.
   - Allocate 20GB disk space for the OS installation.
<br>

## OS Installation

1. Boot the VM using the CentOS 9 Stream ISO.
2. Follow the CentOS 9 Stream installation steps until completion.
3. After installation, update CentOS with the following command:
    ```bash
    sudo dnf update -y
    ```
<br>

## Docker Installation

1. Install Docker with the following commands:
    ```bash
    sudo yum install -y yum-utils
    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sudo yum install -y docker-ce docker-ce-cli containerd.io
    ```

2. Start and enable Docker:
    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```
<br>

## Services Installation

### Webserver

1. Install Docker Compose:
    ```bash
    sudo curl -L "https://github.com/docker/compose/releases/download/$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep tag_name | cut -d '"' -f 4)/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    ```

2. Create a `docker-compose.yml` file:
    ```yaml
    services:
      nginx:
        image: nginx:latest
        ports:
          - "10080:80"
        volumes:
          - ./html:/usr/share/nginx/html
        networks:
          - webnet
    ```

3. Create an HTML folder and add an HTML file:
    ```bash
    mkdir html
    echo "<html><body><h1>Hello from Docker</h1></body></html>" > html/index.html
    ```

4. Run Docker Compose:
    ```bash
    sudo docker compose up -d
    ```

5. Access the HTML page via `http://<VM_IP>:10080` to ensure Nginx is working correctly.
<br>

### Reverse Proxy

1. Create a Self-Signed SSL Certificate for Nginx:
    ```bash
    sudo openssl req -newkey rsa:2048 -nodes -keyout /etc/pki/nginx/private/server.key -x509 -days 365 -out /etc/pki/nginx/server.crt
    ```

2. Add NGINX Proxy Manager service to the `docker-compose.yml`:
    ```yaml
    services:
      nginx:
        image: nginx:latest
        ports:
          - "10080:80"
        volumes:
          - ./html:/usr/share/nginx/html
        networks:
          - webnet

      nginx-proxy-manager:
        image: 'jc21/nginx-proxy-manager:latest'
        restart: always
        ports:
          - "80:80"
          - "81:81"
          - "443:443"
        volumes:
          - ./data:/data
          - ./letsencrypt:/etc/letsencrypt
        environment:
          DB_SQLITE_FILE: "/data/database.sqlite"
        networks:
          - webnet

    networks:
      webnet:
    ```

3. Create a local domain in the hosts file:

    - Open a terminal and edit the hosts file:
        ```bash
        sudo nano /etc/hosts
        ```
    - Add the following line at the bottom of the file:
        ```bash
        <VM_IP>   your_domain
        ```
    - Save and exit the editor.

4. Update Nginx configuration in `/etc/nginx/nginx.conf`:

    - Change HTTP and HTTPS ports to 8080 and 8443 to avoid conflicts with Docker ports.
    - Ensure `server_name` matches the local domain created.
    - Verify SSL Certificate paths are correct.

      ```nginx
      user nginx;
      worker_processes auto;
      error_log /var/log/nginx/error.log;
      pid /run/nginx.pid;

      include /usr/share/nginx/modules/*.conf;

      events {
          worker_connections 1024;
      }

      http {
          log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                            '$status $body_bytes_sent "$http_referer" '
                            '"$http_user_agent" "$http_x_forwarded_for"';
          access_log  /var/log/nginx/access.log  main;

          sendfile            on;
          tcp_nopush          on;
          tcp_nodelay         on;
          keepalive_timeout   65;
          types_hash_max_size 4096;

          include             /etc/nginx/mime.types;
          default_type        application/octet-stream;

          server {
              listen       8080;
              listen       [::]:8080;
              server_name  your.domain;
              root         /path/to/your/folder;

              include /etc/nginx/default.d/*.conf;

              error_page 404 /404.html;
              location = /404.html {
              }

              error_page 500 502 503 504 /50x.html;
              location = /50x.html {
              }
          }

          server {
              listen       8443 ssl http2;
              listen       [::]:8443 ssl http2;
              server_name  your.domain;
              root         /path/to/your/folder;

              ssl_certificate "/path/to/your/certificate";
              ssl_certificate_key "/path/to/your/certificate_key";
              ssl_session_cache shared:SSL:1m;
              ssl_session_timeout  10m;
              ssl_ciphers PROFILE=SYSTEM;
              ssl_prefer_server_ciphers on;

              include /etc/nginx/default.d/*.conf;

              error_page 404 /404.html;
              location = /404.html {
              }

              error_page 500 502 503 504 /50x.html;
              location = /50x.html {
              }
          }
      }
      ```

5. Restart Nginx:
    ```bash
    sudo systemctl restart nginx
    ```

6. Run Docker Compose again:
    ```bash
    sudo docker compose up -d
    ```

7. Access NGINX Proxy Manager at `http://<VM_IP>:81` or `http://your_domain:81` to verify Nginx is working properly and for further configuration.
<br>

### Configuration in Nginx Proxy Manager

1. Add a Proxy Host:

    | Config        | Value         |
    |---------------|---------------|
    | Domain Names  | your_domain   |
    | Scheme        | HTTP          |
    | Forward IP    | VM_IP         |
    | Forward Port  | 10080         |

2. Add SSL Certificate > Custom:

    | Config           | Value                |
    |------------------|----------------------|
    | Name             | your_crt_name        |
    | Certificate Key  | upload_your_crt_key  |
    | Certificate      | upload_your_crt      |
<br>

### Perform Hardening

    sudo firewall-cmd --permanent --remove-service=dhcpv6-client
    sudo firewall-cmd --permanent --add-service={http,https,ssh}
    sudo firewall-cmd --reload
<br>
<br>

# Testing

Open in your browser:

    http://your_domain:80 # To access HTML on port 80
    https://your_domain:443 # To access HTML on port 443
    http://your_domain:81 # To access Nginx Proxy Manager GUI

