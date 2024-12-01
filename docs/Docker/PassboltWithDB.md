##
# Passbolt
**[Passbolt](https://www.passbolt.com/)**, is an open source credential platform for modern teams. A versatile, battle-tested solution to manage and collaborate on passwords, accesses, and secrets. All in one.

*In this document*, we will create passbolt with Docker and external database. You can click [link](https://www.passbolt.com/ce/docker) for default contanier based database installation and other methods.

## Requirements

- **Docker**, can install from [here.](https://docs.docker.com/engine/install/)
- Install External Database MariaDB server. Install from  [digital ocean document](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-rocky-linux-8) or [official document](https://mariadb.org/download/?t=repo-config). You can choose your Linux OS type both of them.
- **SMTP:** Need a smtp server for production but you can add from UI after installation. If you don't have smtp server, you can use your gmail or yandex mail as a smtp. For development, can use mailchatcher as a container:

```bash title="Mailctahcer Run with Docker"
docker run -d -p 1080:1080 -p 1025:1025 sj26/mailcatcher
```
After start mailcathcer, enter browser `http://localhost:1080` acces to ui. You can test mailcatcher with codes below.
```bash
# First connect with telnet after run each command
telnet localhost 1025

HELO example.com
MAIL FROM:<sender@example.com>
RCPT TO:<recipient@example.com>
DATA
Subject: Test Mail
This is a test email.
.
QUIT
```
After that, you will see a mail in ui.

## Setup External Database

- Check installation. If status is active, there is no problem :)

```bash
sudo systemctl status mariadb
```

- Start secure configuration
    
    ```bash
    sudo mysql_secure_installation
    ```
    
    1. **Enter current password for root (enter for none):** Press enter as there is no password by default.
    2. **Set root password? [Y/n]:** Select Y and enter a new password.
    3. **Remove anonymous users? [Y/n]:** Select Y
    4. **Disallow root login remotely? [Y/n]**: Enter Y
    5. **Remove the test database and access to it? [Y/n]:** Enter Y
    6. **Reload privilege tables now? [Y/n]:** Enter Y
- Connect MariaDB

```bash
mysql -u root -p
```

- Set remote database.
    - Open `/etc/mysql/mariadb.conf.d/50-server.cnf` file.
    
    ```bash
    sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
    ```
    
    - Find `bind-address` in file and change `127.0.0.1`to `0.0.0.0`.
    
    ```conf
    bind-address            = 0.0.0.0
    ```
    
    - Restart MariaDB
    
    ```bash
    sudo systemctl restart mariadb
    ```
    
- Create  *`passbolt`* database and *`passbolt`* user for access Passbolt in MariaDB console. Can access MariaDB remotely with *passbolt* user.
    
    ```sql
    CREATE DATABASE passbolt;
    GRANT ALL ON passbolt.* to 'passbolt'@'%' IDENTIFIED BY 'password';
    ```
    
- Try remote connection from other server or local computer.
    
    ```bash
    mysql -u passbolt -p -h <your_machine_ip>
    ```
    
##Setup SSL

- Create `traefik` file.

```bash
mkdir traefik
```

- Create `traefik.yaml` . Change `<yourname@domain.tld>` to your email.

```yaml title="traefik.yaml"
global:
  sendAnonymousUsage: false
log:
  level: INFO
  format: common
providers:
  docker:
    endpoint: 'unix:///var/run/docker.sock'
    watch: true
    exposedByDefault: true
    swarmMode: false
  file:
    directory: /etc/traefik/conf/
    watch: true
api:
  dashboard: false
  debug: false
  insecure: false
entryPoints:
  web:
    address: ':80'
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: ':443'
certificatesResolvers:
  letsencrypt:
    acme:
      email: <yourname@domain.tld>
      storage: /shared/acme.json
      caServer: 'https://acme-v02.api.letsencrypt.org/directory'
      keyType: EC256
      httpChallenge:
        entryPoint: web
      tlsChallenge: {}
```

- Create `conf`folder in to the traefik folder.
    
    ```bash
    mkdir conf
    ```
    
    - Create `headers.yaml` in `conf`.
    
    ```yaml title="conf/headers.yaml"
    http:
      middlewares:
        SslHeader:
          headers:
            FrameDeny: true
            AccessControlAllowMethods: 'GET,OPTIONS,PUT'
            AccessControlAllowOriginList:
              - origin-list-or-null
            AccessControlMaxAge: 100
            AddVaryHeader: true
            BrowserXssFilter: true
            ContentTypeNosniff: true
            ForceSTSHeader: true
            STSIncludeSubdomains: true
            STSPreload: true
            ContentSecurityPolicy: default-src 'self' 'unsafe-inline'
            CustomFrameOptionsValue: SAMEORIGIN
            ReferrerPolicy: same-origin
            PermissionsPolicy: vibrate 'self'
            STSSeconds: 315360000
    ```
    
    - Create `tls.yaml` in `conf`.
    
    ```yaml title="conf/tls.yaml"
    tls:
      options:
        default:
          minVersion: VersionTLS12
          sniStrict: true
          curvePreferences:
            - CurveP521
            - CurveP384
          cipherSuites:
            - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
            - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
            - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256l
    ```
    
- All file system like this:

```bash
+ ---traefik
	|--traefik.yaml
	+ --conf
		|--headers.yaml
		|--tls.yaml

```

##Docker Compose Setup

- Docker Compose file:
    - `APP_FULL_BASE_URL:` server ip or domain.
    - `DATASOURCES_DEFAULT_HOST:` database server ip. In our case, server ip.
    - `DATASOURCES_DEFAULT_PASSWORD:` database  user password. (passbolt user’s password)
    - `EMAIL_TRANSPORT_DEFAULT_HOST:`e-mail server domain.(smtp.gmail.com , smtp.yandex.ru). For mailcather, your server IP. For local, use internal IP not `localhost or 127.0.0.1` .
    - `EMAIL_TRANSPORT_DEFAULT_USERNAME` and `EMAIL_TRANSPORT_DEFAULT_PASSWORD` are mail authentication information. Mail addres and mail password / app password. Don't use for mailcatcher.
    - `EMAIL_TRANSPORT_DEFAULT_PORT` mail server port. For mailcatcher, use 1025.
    
    ```yaml title="docker-compose.yml"
    version: '3.5'
    services:
      passbolt:
        image: passbolt/passbolt:latest-ce
        restart: unless-stopped
        environment:
          APP_FULL_BASE_URL: "https://<your_server_domain_or_ip>"
          DATASOURCES_DEFAULT_HOST: "<your_server_domain_or_ip>"
          DATASOURCES_DEFAULT_USERNAME: "passbolt"
          DATASOURCES_DEFAULT_PASSWORD: "<passbolt_db_user_pass>"
          DATASOURCES_DEFAULT_DATABASE: "passbolt"
          EMAIL_TRANSPORT_DEFAULT_HOST: "<mail_server_url><smtp.gmail.com/smtp.yandex.mail.ru>"
          EMAIL_TRANSPORT_DEFAULT_PORT: 587
          EMAIL_TRANSPORT_DEFAULT_USERNAME: "<email@mail.com>"
          EMAIL_TRANSPORT_DEFAULT_PASSWORD: "<email_password>"
        volumes:
          - gpg_volume:/etc/passbolt/gpg
          - jwt_volume:/etc/passbolt/jwt
        command: ["/usr/bin/wait-for.sh", "-t", "0", "<your_server_domain_or_ip>:3306", "--", "/docker-entrypoint.sh"]
        labels:
          traefik.enable: "true"
          traefik.http.routers.passbolt-http.entrypoints: "web"
          traefik.http.routers.passbolt-http.rule: "Host(`<your_server_domain_or_ip>`)"
          traefik.http.routers.passbolt-http.middlewares: "SslHeader@file"
          traefik.http.routers.passbolt-https.middlewares: "SslHeader@file"
          traefik.http.routers.passbolt-https.entrypoints: "websecure"
          traefik.http.routers.passbolt-https.rule: "Host(`<your_server_domain_or_ip>`)"
          traefik.http.routers.passbolt-https.tls: "true"
          traefik.http.routers.passbolt-https.tls.certresolver: "letsencrypt"
      traefik:
        image: traefik:2.6
        restart: always
        ports:
          - 80:80
          - 443:443
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - ./traefik/traefik.yaml:/traefik.yaml:ro
          - ./traefik/conf/:/etc/traefik/conf
          - ./shared/:/shared
    volumes:
      gpg_volume:
      jwt_volume:
    ```
    

## Run run run

- Change your directory to `docker-compose.yaml` directory. And then, up the system:
    
    ```bash
    docker compose up 
    ```
    
- Create first user on other terminal or use `-d` parameter in `docker compose up` command. You can choose *admin* or *user* end of the command.
    
    ```bash
    docker-compose exec passbolt su -m -c "/usr/share/php/passbolt/bin/cake \
                                    passbolt register_user \
                                    -u mail@mail.com \
                                    -f name \
                                    -l surname \
                                    -r admin/user" -s /bin/sh www-data
    ```
    

<aside>
🚨 You can click the link in the output and create passhare for user. Don’t lose your private key. It is very important for recovery. If you lose your private key or passhare, you can’t recovery.

</aside>








