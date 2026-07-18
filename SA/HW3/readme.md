# HW3 筆記與佈署指南

## 基本設定

### 1. Set up nopasswd user

這個應該在作業一就做過了：

```bash
sudo visudo -f /etc/sudoers.d/judge

# 新增以下內容
judge ALL=(ALL:ALL) NOPASSWD: ALL

```

### 2. 設定 `/etc/hosts`

```bash
192.168.255.123 ta.315551018.cs.nycu
192.168.254.145 315551018.cs.nycu
192.168.254.145 hello.315551018.cs.nycu
192.168.254.145 acme.315551018.cs.nycu
192.168.254.145 auth.315551018.cs.nycu
192.168.254.145 matrix.315551018.cs.nycu
192.168.254.145 mas.315551018.cs.nycu
192.168.254.145 i.315551018.cs.nycu
192.168.254.145 postgres.315551018.cs.nycu
```

### 3. 生成 CA 憑證

建立 `root.cnf` 設定檔：

```ini
[ req ]
default_bits       = 4096
distinguished_name = req_distinguished_name
prompt             = no
x509_extensions    = v3_ca

[ req_distinguished_name ]
# ⚠️ 注意：這裡的順序決定了憑證內部的 Subject 順序
C  = TW
O  = National Yang Ming Chiao Tung University
CN = SA315551018 Root CA

[ v3_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign
```

建立 `interca.cnf` 設定檔（限制 Chain Length）：

```ini
cat <<EOF > interca.cnf
[ req ]
default_bits       = 4096
distinguished_name = req_distinguished_name
prompt             = no

[ req_distinguished_name ]
C  = TW
O  = National Yang Ming Chiao Tung University
CN = SA315551018

[ v3_intermediate_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
# pathlen:0 限制這張憑證往下只能簽發一般憑證，不能再簽發 CA
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign
EOF

```

簽發憑證：

```bash
# Root CA 憑證
openssl genrsa -out sarootca.key 4096
openssl req -x509 -new -nodes -key /home/judge/hw3/sarootca.key -sha256 -days 3650 -out sarootca.crt -config rootca.cnf

# 生成 Intermediate CA Key 與 CSR (Certificate Signing Request)
openssl genrsa -out sa.key 4096
openssl req -new -key sa.key -out sa.csr -config interca.cnf
openssl x509 -req -in sa.csr -CA sarootca.crt -CAkey sarootca.key -CAcreateserial -out sa.crt -days 3650 -sha256 -extfile interca.cnf -extensions v3_intermediate_ca
```

將 Root CA 加入系統信任清單：

```bash
sudo cp sarootca.crt /usr/local/share/ca-certificates/sarootca.crt
sudo update-ca-certificates
```

> 如果要使用瀏覽器的話記得自己把兩個憑證加到信任清單

---

## ACME + Traefik

先設定 `ACME server` 使用我們自己簽的 `intermediate CA`：

```bash
sudo mkdir -p /home/judge/hw3/deploy/step
# 調整權限給容器內的 step 使用者
sudo chown -R 1000:1000 /home/judge/hw3/deploy/step

sudo docker run --rm -it \
  -v /home/judge/hw3/deploy/step:/home/step \
  smallstep/step-ca step ca init \
  --name="SA315551018" \
  --dns="acme.315551018.cs.nycu" \
  --address=":9000" \
  --provisioner="acme" \
  --acme

# 執行上面指令的過程中會產生密碼，請將其替換掉下面的 "passwd"
echo "passwd" | sudo tee /home/judge/hw3/deploy/step/password.txt

# 1. 替換 Root CA
sudo cp /home/judge/hw3/sarootca.crt /home/judge/hw3/deploy/step/certs/root_ca.crt

# 2. 替換中介憑證 (Intermediate CA)
sudo cp /home/judge/hw3/sa.crt /home/judge/hw3/deploy/step/certs/intermediate_ca.crt

# 3. 替換中介憑證私鑰
sudo cp /home/judge/hw3/sa.key /home/judge/hw3/deploy/step/secrets/intermediate_ca_key

```

編輯 `ACME server` 設定檔：

```bash
sudo vim /home/judge/hw3/deploy/step/config/ca.json
```

修改以下區塊：

```json
{
  "authority": {
    "policy": {
      "x509": {
        "allow": {
          "dns": ["*.315551018.cs.nycu", "315551018.cs.nycu"]
        }
      }
    },
    "provisioners": [
      {
        "type": "ACME",
        "name": "acme-1",
        "challenges": [
          "http-01"
        ],
        "claims": {
          "maxTLSCertDuration": "30m",
          "defaultTLSCertDuration": "15m"
        }
      }
    ]
  }
}
```

用acme server自己幫自己簽一張憑證

```bash
sudo docker exec -it acme step ca certificate acme.315551018.cs.nycu /home/step/acme.crt /home/step/acme.key

# 這個的時間會非常短提交前可以用這兩個重置一下
sudo docker exec -it acme step ca certificate acme.315551018.cs.nycu /home/step/acme.crt /home/step/acme.key --force
sudo docker compose restart web
```

設定 Traefik 靜態設定檔：

```bash
vim /home/judge/hw3/deploy/traefik.yml
```

```yaml
api:
  insecure: false

serversTransport:
  insecureSkipVerify: true

providers:
  docker:
    exposedByDefault: false
  file:
    directory: /etc/traefik/dynamic

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    http3: {}

certificatesResolvers:
  stepca:
    acme:
      email: sa@315551018.cs.nycu
      storage: /letsencrypt/acme.json
      # 這裡的名稱（acme-1）要跟上面 ca.json 的 provisioner name 一致
      caServer: https://acme.315551018.cs.nycu:9000/acme/acme-1/directory
      certificatesDuration: 1
      httpChallenge:
        entryPoint: web

```

```bash
vim ~/.bashrc
#要放在檔案上面
export SA_ACME_SERVER_URL="https://acme.315551018.cs.nycu/acme/acme-1/directory"

source ~/.bashrc
```

設定 Traefik 動態轉發與 Middleware：

```bash
vim /home/judge/hw3/deploy/traefik-dynamic/middlewares.yml
```

```yaml
http:
  middlewares:
    sec-headers:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        forceSTSHeader: true
        customResponseHeaders:
          Server: ""
          X-Powered-By: ""
          Via: ""
```

設定 Traefik acme router的 Cert

```bash
vim /home/judge/hw3/deploy/tls.yml
```

```yaml
tls:
  certificates:
    - certFile: "/certs/acme.crt"
      keyFile: "/certs/acme.key"
```

建立基礎的 `docker-compose.yml`：

```bash
vim /home/judge/hw3/deploy/docker-compose.yml
```

```yaml
services:
  web:
    image: traefik:v3.6.1
    container_name: web
    restart: always
    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "443:443/udp"
    environment:
      - LEGO_CA_CERTIFICATES=/certs/sarootca.crt
    # 加這個設定是為了讓 acme server 在簽發憑證時找得到對象
    networks:
      default: 
        aliases:
          - 315551018.cs.nycu
          - hello.315551018.cs.nycu
          - auth.315551018.cs.nycu
          - matrix.315551018.cs.nycu
          - mas.315551018.cs.nycu
          - i.315551018.cs.nycu
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./traefik.yml:/etc/traefik/traefik.yml:ro"
      - "./traefik-acme:/letsencrypt"
      - "./traefik-dynamic:/etc/traefik/dynamic"
      - "/home/judge/hw3/sarootca.crt:/certs/sarootca.crt:ro"
      # 前面用acme server自己幫自己簽的
      - "./step/acme.crt:/certs/acme.crt:ro"
      - "./step/acme.key:/certs/acme.key:ro"
    depends_on:
      - acme

  acme:
    image: smallstep/step-ca
    container_name: acme
    restart: always
    environment:
      - STEPPATH=/home/step
    command: /usr/local/bin/step-ca --password-file /home/step/password.txt /home/step/config/ca.json
    networks:
      default:
        aliases:
          - acme.315551018.cs.nycu
    extra_hosts:
      - "ta.315551018.cs.nycu:192.168.255.123"
    volumes:
      - ./step:/home/step
    labels:
      - "traefik.enable=true" # 告訴 Traefik 要代理這個容器
      - "traefik.http.routers.acme.rule=Host(`acme.315551018.cs.nycu`)" # 設定路由規則
      - "traefik.http.routers.acme.entrypoints=websecure" # 綁定在 443 port
      - "traefik.http.routers.acme.tls=true" # 告訴 Traefik 這個路由要啟用 TLS
      - "traefik.http.services.acme.loadbalancer.server.port=9000" # 這裡請填寫 step-ca 實際監聽的 Port 
      - "traefik.http.services.acme.loadbalancer.server.scheme=https"


  hello:
    image: hashicorp/http-echo
    restart: always
    command: -text="I love NYCU NASA 2025"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.hello.rule=Host(`hello.315551018.cs.nycu`)"
      - "traefik.http.routers.hello.entrypoints=websecure"
      - "traefik.http.routers.hello.tls.certresolver=stepca"
      # 套用剛剛建立的安全標頭 middleware
      - "traefik.http.routers.hello.middlewares=sec-headers@file"

volumes:
  acme-data:
```

---

## Postgres DB

詳細原理可參考 [這篇貼文](https://sliplane.io/blog/setup-tls-for-postgresql-in-docker)。

先簽發一個給 `postgres db` 使用的憑證：

```bash
# 建立獨立資料夾管理憑證
cd /home/judge/hw3
mkdir judge_client
cd judge_client

openssl req -newkey rsa:2048 -nodes -keyout postgres.key -out postgres.csr -subj "/CN=postgres.315551018.cs.nycu"
openssl x509 \
  -req \
  -in postgres.csr \
  -CA sa.crt \
  -CAkey sa.key \
  -CAcreateserial \
  -out postgres.crt \
  -days 365 \
  -sha256 \
  -extfile <(printf "basicConstraints=critical,CA:FALSE\nkeyUsage=critical,digitalSignature,keyEncipherment\nextendedKeyUsage=serverAuth\nsubjectAltName=DNS:postgres.315551018.cs.nycu")
```

打包憑證信任鏈：

```bash
cd /home/judge/hw3

# 1. 打包「伺服器憑證鏈」(伺服器專屬憑證 + 中介 CA)
cat postgres.crt sa.crt > postgres_chain.crt

# 2. 打包「完整信任根」(中介 CA + Root CA)
cat sa.crt sarootca.crt > ca_bundle.crt
```

建立資料庫啟動初始化腳本（包含後面 Keycloak, Synapse, MAS, UURL 所需的 DB 設定）：

```bash
vim /home/judge/hw3/deploy/pg-entrypoint.sh
```

```bash
#!/bin/sh
set -e

echo "=== [1] 正在設定 PostgreSQL TLS 憑證與權限 ==="
mkdir -p /var/lib/postgresql/certs
cp /certs/postgres.crt /var/lib/postgresql/certs/server.crt
cp /certs/postgres.key /var/lib/postgresql/certs/server.key
cp /certs/sarootca.crt /var/lib/postgresql/certs/root.crt

chown -R postgres:postgres /var/lib/postgresql/certs
chmod 600 /var/lib/postgresql/certs/server.key

echo "=== [2] 產生資料庫初始化腳本 ==="
mkdir -p /docker-entrypoint-initdb.d

cat <<EOF > /docker-entrypoint-initdb.d/01-init.sql
CREATE DATABASE uurl;
CREATE DATABASE keycloak;
CREATE DATABASE synapse;
CREATE DATABASE mas;

-- 1. 建立助教的 judge 帳號
CREATE USER judge WITH PASSWORD 'judge';
GRANT ALL PRIVILEGES ON DATABASE uurl TO judge;
GRANT ALL PRIVILEGES ON DATABASE keycloak TO judge;

-- 2. 建立 Keycloak 專用的內部帳號
CREATE USER keycloak_user WITH PASSWORD 'keycloak_password';
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak_user;

-- 3. 建立 Synapse 專屬帳號
CREATE USER synapse_user WITH PASSWORD 'synapse_password';
GRANT ALL PRIVILEGES ON DATABASE synapse TO synapse_user;

-- 4. 建立 MAS 專屬帳號
CREATE USER mas_user WITH PASSWORD 'mas_password';
GRANT ALL PRIVILEGES ON DATABASE mas TO mas_user;

-- 5. 建立 URL Shortener 專屬帳號
CREATE USER uurl_user WITH PASSWORD 'uurl_password';
GRANT ALL PRIVILEGES ON DATABASE uurl TO uurl_user;

-- =====================================
-- 資料庫結構與權限設定
-- =====================================

\c uurl
GRANT ALL ON SCHEMA public TO judge;
GRANT ALL ON SCHEMA public TO uurl_user;

-- 建立 URL 短網址資料表
CREATE TABLE urls (
    id SERIAL PRIMARY KEY,
    creator VARCHAR(255),
    original_url TEXT NOT NULL,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 將表格與自增主鍵(Sequence)的權限授予 uurl_user 和 judge
GRANT ALL PRIVILEGES ON TABLE urls TO uurl_user, judge;
GRANT USAGE, SELECT ON SEQUENCE urls_id_seq TO uurl_user, judge;

\c keycloak
GRANT ALL ON SCHEMA public TO keycloak_user;

\c synapse
GRANT ALL ON SCHEMA public TO synapse_user;

\c mas
GRANT ALL ON SCHEMA public TO mas_user;
EOF

cat <<'EOF' > /docker-entrypoint-initdb.d/02-hba.sh
#!/bin/sh
echo "Injecting strict rules into pg_hba.conf..."
# 行 1：讓 judge 必須通過 mTLS 驗證
# 行 2-5：讓內部服務可以透過密碼連線到對應的資料庫
sed -i '1s/^/hostssl all judge all cert clientcert=verify-full\n\
host keycloak keycloak_user all scram-sha-256\n\
host synapse synapse_user all scram-sha-256\n\
host mas mas_user all scram-sha-256\n\
host uurl uurl_user all scram-sha-256\n/' "$PGDATA/pg_hba.conf"
EOF
chmod +x /docker-entrypoint-initdb.d/02-hba.sh

echo "=== [3] 啟動 PostgreSQL 伺服器 ==="
exec docker-entrypoint.sh postgres \
  -c ssl=on \
  -c ssl_cert_file=/var/lib/postgresql/certs/server.crt \
  -c ssl_key_file=/var/lib/postgresql/certs/server.key \
  -c ssl_ca_file=/var/lib/postgresql/certs/root.crt
```

確保指令碼具備執行權限：

```bash
chmod +x /home/judge/hw3/deploy/pg-entrypoint.sh
```

更新 `docker-compose.yml`，將 `postgres` 服務加入：

```yaml
  postgres:
    image: postgres:17-alpine
    container_name: postgres
    restart: always
    ports:
      - "5432:5432"
    environment:
      # 設定 root 密碼，避免啟動報錯
      POSTGRES_USER: root
      POSTGRES_PASSWORD: supersecretroot
    # 覆蓋預設 entrypoint，改用我們剛剛寫的腳本
    entrypoint: ["/pg-entrypoint.sh"]
    volumes:
      # 掛載我們的腳本
      - ./pg-entrypoint.sh:/pg-entrypoint.sh:ro
      # 掛載一開始做好的 CA 與中介憑證 (唯讀即可，腳本會自己拷貝)
      - /home/judge/hw3/ca_bundle.crt:/certs/sarootca.crt:ro
      - /home/judge/hw3/postgres_chain.crt:/certs/postgres.crt:ro
      - /home/judge/hw3/postgres.key:/certs/postgres.key:ro
      # 具名資料卷，確保資料持久化
      - postgres-data:/var/lib/postgresql/data
    networks:
      default:
        aliases:
          - postgres.315551018.cs.nycu
```

測試連線：

```bash
cd /home/judge/hw3/judge_client

psql "host=postgres.315551018.cs.nycu port=5432 dbname=uurl user=judge sslmode=verify-full sslrootcert='../ca_bundle.crt' sslcert=judge_client.crt sslkey=judge_client.key"
```

---

## Keycloak

在 `docker-compose.yml` 中加上 `auth` (Keycloak) 服務：

```yaml
  auth:
    image: keycloak/keycloak:26.6
    container_name: auth
    restart: always
    command: start
    environment:
      - KC_HOSTNAME=auth.315551018.cs.nycu
      - KC_HOSTNAME_STRICT=false
      # 這是 Keycloak 網頁後台的最高管理員帳密
      - KC_BOOTSTRAP_ADMIN_USERNAME=admin
      - KC_BOOTSTRAP_ADMIN_PASSWORD=admin
      # 連線到底層 Postgres 資料庫的帳密設定 (與 pg-entrypoint.sh 對應)
      - KC_DB=postgres
      - KC_DB_URL=jdbc:postgresql://postgres:5432/keycloak
      - KC_DB_USERNAME=keycloak_user
      - KC_DB_PASSWORD=keycloak_password
    volumes:
      - auth-data:/data
    depends_on:
      - postgres
    networks:
      default:
        aliases:
          - auth.315551018.cs.nycu
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.auth.rule=Host(`auth.315551018.cs.nycu`)"
      - "traefik.http.routers.auth.entrypoints=websecure"
      - "traefik.http.routers.auth.tls.certresolver=stepca"
      # Keycloak 內建的網頁服務連接埠是 8080
      - "traefik.http.services.auth.loadbalancer.server.port=8080"
      # 安全考量：強制加上我們之前設定好的 HSTS 等安全防禦標頭
      - "traefik.http.routers.auth.middlewares=sec-headers@file"


volumes:
  auth-data:
```

建立完成後，到 Keycloak 後台建立一個 client (`sa-client`)。開啟 `Service Accounts Enabled`（Machine-to-Machine 功能），並複製 `Credentials` 頁籤中的 Client Secret。

建立測試腳本 `test-oidc.sh`：

```bash
#!/bin/bash

# ==========================================
# 請在這裡填入你的變數
# ==========================================
PROVIDER_URL="https://auth.315551018.cs.nycu/realms/master"
CLIENT_ID="sa-client"
# ⚠️ 請把下面這串換成你在 Credentials 頁籤複製的密碼
CLIENT_SECRET="QqYTNkSjL5YYU9bYOX6HACfVlfBuDbnT"

# ==========================================
echo -e "\n=== [第一關] 測試 OIDC Discovery Document ==="
echo "目標: 驗證 issuer 以及各個 endpoint 是否存在且為 HTTPS"

# 下載並解析 Discovery Document
DISCOVERY_JSON=$(curl -s -k "$PROVIDER_URL/.well-known/openid-configuration")

# 提取必要欄位 (使用 jq 解析 JSON)
ISSUER=$(echo "$DISCOVERY_JSON" | jq -r '.issuer')
TOKEN_EP=$(echo "$DISCOVERY_JSON" | jq -r '.token_endpoint')
USERINFO_EP=$(echo "$DISCOVERY_JSON" | jq -r '.userinfo_endpoint')
INTROSPECT_EP=$(echo "$DISCOVERY_JSON" | jq -r '.introspection_endpoint')
REVOCATION_EP=$(echo "$DISCOVERY_JSON" | jq -r '.revocation_endpoint')

echo "- Issuer: $ISSUER"
echo "- Token Endpoint: $TOKEN_EP"
echo "- UserInfo Endpoint: $USERINFO_EP"
echo "- Introspection Endpoint: $INTROSPECT_EP"
echo "- Revocation Endpoint: $REVOCATION_EP"

if [[ "$ISSUER" == "$PROVIDER_URL" && "$TOKEN_EP" == https* && "$REVOCATION_EP" == https* ]]; then
    echo -e "✅ 第一關通過！端點全為 HTTPS 且 Issuer 吻合。"
else
    echo -e "❌ 第一關失敗！請檢查上面的輸出是否為空值或 HTTP。"
    exit 1
fi

# ==========================================
echo -e "\n=== [第二關] 測試 Machine-to-Machine 取得 Token ==="
echo "目標: 模擬助教腳本，使用 Client Credentials Grant 獲取 Token"

# 對 Token Endpoint 發送 POST 請求
TOKEN_RESPONSE=$(curl -s -k -X POST "$TOKEN_EP" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET")

# 嘗試提取 access_token
ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r '.access_token')

if [ "$ACCESS_TOKEN" != "null" ] && [ -n "$ACCESS_TOKEN" ]; then
    echo -e "✅ 第二關通過！成功取得 Access Token！"
    echo "Token 預覽 (前30字元): ${ACCESS_TOKEN:0:30}..."
else
    echo -e "❌ 第二關失敗！無法取得 Token。Keycloak 回應如下："
    echo "$TOKEN_RESPONSE" | jq .
fi
echo -e "\n測試結束。\n"

```

```bash
vim ~/.bashrc
# 記得放在檔案上面

# 上面建立的那個client
export SA_OIDC_CLIENT_ID="sa-client"
export SA_OIDC_CLIENT_SECRET="gItQ7kfz29EdORpp6DlSlzAbF9Xw7KSa"

export SA_OIDC_PROVIDER="https://auth.315551018.cs.nycu/realms/master/.well-known/openid-configuration"

source ~/.bashrc
```

---

## Matrix Homeserver Setup

先在 Keycloak 後台建立一個 `mas-client`，設定選取 `Standard Flow` 與 `Service Account Roles`，並將 Redirect URL 設為 `https://mas.315551018.cs.nycu/`。

### MAS (Matrix Authentication Service) 設定

建一個加密用的`pem key`

```bash
openssl genrsa -out mas-signing.pem 2048
```

```bash
sudo vim /home/judge/hw3/deploy/mas-config.yaml
```

```yaml
http:
  # 補上這個強制欄位，告訴 MAS 它的對外真實網址
  public_base: "https://mas.315551018.cs.nycu/"
  listeners:
    - binds:
        - host: 0.0.0.0
          port: 8080
      resources:
        - name: discovery
        - name: human
        - name: oauth
        - name: compat        
        - name: graphql
          # 如果要在本機從 curl 測試的話要加這個
          # undocumented_oauth2_access: true
        - name: assets
        - name: adminapi

secrets:
  # Encryption secret (used for encrypting cookies and database fields)
  # This must be a 32-byte long hex-encoded key (可以使用 openssl rand -hex 32 生成)
  encryption: 5eb31bf6243052f9b74d42152bad6abf253a14ed9884f05bb9260616608d142e
  keys:
    - file: /mas-keys/mas-signing.pem

# 連線到我們在 PostgreSQL 建立的專屬資料庫
database:
  uri: "postgresql://mas_user:mas_password@postgres:5432/mas"

# 對接 Synapse 的設定
matrix:
  homeserver: "315551018.cs.nycu"
  endpoint: "http://matrix:8008"
  secret: "synapse_mas_shared_secret" # 這個密碼之後也要填到 Synapse 設定檔裡

# 對接 Keycloak (OIDC Provider) 的設定
upstream_oauth2:
  providers:
    # 必須是一個合法的 ULID
    - id: 01KVFQQYYDQNZR6A9ZPXMXA2BV
      issuer: "https://auth.315551018.cs.nycu/realms/master"
      client_id: "mas-client"
      token_endpoint_auth_method: client_secret_basic
      client_secret: "<請把你的_MAS_Client_Secret_貼在這裡>"
      claims_imports:
        localpart:
          action: require
          template: "{{ user.preferred_username }}"

experimental:
  access_token_ttl: 86400
  compat_token_ttl: 86400
  inactive_session_expiration:
    ttl: 86400
    expire_oauth_session: false
    expire_user_session: false


# 助教評分用的 Admin API 客戶端
clients:
  - client_id: "01KVFQFRSP69DXFJWEZP8P156V" # 這是作業要求的 SA_MAS_CLIENT_ID
    client_auth_method: client_secret_basic
    client_secret: "sa-mas-secret" # 這是作業要求的 SA_MAS_CLIENT_SECRET

# 宣告系統的權限策略
policy:
  data:
    # 將助教的 Client ID 加進管理員白名單
    admin_clients:
      - "01KVFQFRSP69DXFJWEZP8P156V"
```

建立包含完整系統與自簽 Root CA 的憑證組合包：

```bash
cat /etc/ssl/certs/ca-certificates.crt /home/judge/hw3/sarootca.crt > /home/judge/hw3/mas_ca_bundle.crt
```

在 `docker-compose.yml` 加上 `mas` 服務：

```yaml
  mas:
    image: ghcr.io/element-hq/matrix-authentication-service:1.19.0
    container_name: mas
    restart: always
    volumes:
      - ./mas-config.yaml:/mas.yaml:ro
      - /home/judge/hw3/mas_ca_bundle.crt:/etc/ssl/certs/ca-certificates.crt:ro
      - /home/judge/hw3/deploy/mas-signing.pem:/mas-keys/mas-signing.pem

      - mas-data:/data
    environment:
      - MAS_CONFIG=/mas.yaml
    depends_on:
      - postgres
    networks:
      default:
        aliases:
          - mas.315551018.cs.nycu
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 10s
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mas.rule=Host(`mas.315551018.cs.nycu`)"
      - "traefik.http.routers.mas.entrypoints=websecure"
      - "traefik.http.routers.mas.tls.certresolver=stepca"
      - "traefik.http.services.mas.loadbalancer.server.port=8080"
      - "traefik.http.routers.mas.middlewares=sec-headers@file"

volumes:
  mas-data:
```

建立測試腳本 `test-mas.sh`：

```bash
#!/bin/bash

MAS_URL="https://mas.315551018.cs.nycu"
CLIENT_ID="01KVFQFRSP69DXFJWEZP8P156V"
CLIENT_SECRET="sa-mas-secret"

echo "=== [第一關] 測試 MAS 服務是否上線 ==="
DISCOVERY_STATUS=$(curl -s -o /dev/null -w "%{http_code}" -k "$MAS_URL/.well-known/openid-configuration")

if [ "$DISCOVERY_STATUS" == "200" ]; then
    echo "✅ 第一關通過！MAS 已成功啟動並可透過 Traefik 存取。"
else
    echo "❌ 第一關失敗！HTTP 狀態碼為 $DISCOVERY_STATUS，請檢查 Traefik 設定或 MAS Log。"
fi

echo -e "\n=== [第二關] 測試 Admin API (Client Credentials) ==="
TOKEN_RESPONSE=$(curl -s -k -X POST "$MAS_URL/oauth2/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET")

ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r '.access_token')

if [ "$ACCESS_TOKEN" != "null" ] && [ -n "$ACCESS_TOKEN" ]; then
    echo "✅ 第二關通過！成功取得 MAS Admin API 的 Token！"
else
    echo "❌ 第二關失敗！無法取得 Token。回應如下："
    echo "$TOKEN_RESPONSE" | jq .
fi

```

### Homeserver (Synapse) 設定

```bash
sudo vim /home/judge/hw3/deploy/homeserver.yaml
```

```yaml
report_stats: false

# 伺服器網域 (這是助教必定會檢查的 Server Name)
server_name: "315551018.cs.nycu"

# 對外發布的公開網址
public_baseurl: "https://matrix.315551018.cs.nycu"

# 監聽設定：只監聽內部 8008 埠，交由 Traefik 反向代理
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true # 允許信任從 Traefik 傳遞過來的真實 IP
    resources:
      - names: [client, federation]
        compress: false

rc_message:
  per_second: 100
  burst_count: 1000


rc_invites:
  per_room:
    per_second: 100
    burst_count: 1000
  per_user:
    per_second: 100
    burst_count: 1000

# 資料庫連線設定 (指向我們一開始建立的 Synapse 專屬帳號)
database:
  name: psycopg2
  allow_unsafe_locale: true
  args:
    user: synapse_user
    password: synapse_password
    database: synapse
    host: postgres
    port: 5432
    cp_min: 5
    cp_max: 10

# 將登入與註冊全權委託給 MAS 處理
experimental_features:
  extensible_intent_integration: true
  msc3861:
    enabled: true
    # 這是 MAS 在 Docker 內網的位址
    issuer: "https://mas.315551018.cs.nycu/"
    # 這是助教用來管理 MAS 的 Admin API 客戶端 ID
    client_id: "01KVFQFRSP69DXFJWEZP8P156V"
    client_auth_method: "client_secret_post"
    # 這個密碼必須跟你在 mas-config.yaml 裡寫的 matrix.secret 完全一樣！
    client_secret: "sa-mas-secret"
    # admin token 是在 mas-config.yaml 的 matrix 區塊裡面那個
    admin_token: "synapse_mas_shared_secret"

# 一些必要的安全或效能雜項
media_store_path: "/data/media_store"
registration_shared_secret: "a_random_secret_for_registration_123"
macaroon_secret_key: "a_random_macaroon_secret_key_456"
form_secret: "a_random_form_secret_789"
signing_key_path: "/data/signing.key"
trusted_key_servers:
  - server_name: "matrix.org"

```

在 `docker-compose.yml` 加上 `matrix` 服務：

```yaml
  matrix:
    image: matrixdotorg/synapse:latest
    container_name: matrix
    restart: always
    volumes:
      - ./homeserver.yaml:/data/homeserver.yaml:ro
      # 給它一個目錄存放 Log、媒體檔案和自動產生的密鑰
      - ./synapse-data:/data
      # ⚠️ 關鍵！把 Step-CA 憑證掛載進去，讓 Synapse 信任 MAS 的憑證
      - /home/judge/hw3/sarootca.crt:/certs/sarootca.crt:ro
    environment:
      # 告訴 Synapse 去哪裡找信任的根憑證 (Python 的 requests 庫支援這個變數)
      - REQUESTS_CA_BUNDLE=/certs/sarootca.crt
    depends_on:
      - postgres
      - mas
    networks:
      default:
        aliases:
          - matrix.315551018.cs.nycu
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8008/_matrix/client/versions"]
      interval: 5s
      timeout: 2s
      retries: 20
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.matrix.rule=Host(`matrix.315551018.cs.nycu`)"
      - "traefik.http.routers.matrix.entrypoints=websecure"
      - "traefik.http.routers.matrix.tls.certresolver=stepca"
      - "traefik.http.services.matrix.loadbalancer.server.port=8008"
      - "traefik.http.routers.matrix.middlewares=sec-headers@file"

```

初始化資料夾並修正權限：

```bash
mkdir -p /home/judge/hw3/deploy/synapse-data
sudo chown -R 991:991 /home/judge/hw3/deploy/synapse-data # Synapse 容器內部預設的 user ID 是 991

```

```bash
vim ~/.bashrc

export SA_MAS_CLIENT_ID="01KVFQFRSP69DXFJWEZP8P156V"
export SA_MAS_CLIENT_SECRET="sa-mas-secret"

source ~/.bashrc

curl -k -u "$SA_MAS_CLIENT_ID:$SA_MAS_CLIENT_SECRET"      -d "grant_type=client_credentials&scope=urn:mas:admin"      "https://mas.315551018.cs.nycu/oauth2/token"
```

---

## UURL Shortener 服務

### 修正 Traefik 標頭以攔截認證流量

為了讓 Matrix 能正常從網頁端進行 OIDC 登入，必須修改 `docker-compose.yml` 中 `matrix` 的標頭標籤，利用優先權攔截登入請求：

```yaml
    labels:
      - "traefik.enable=true"
      
      # 🌟 1. 建立唯一的內部接收器
      - "traefik.http.services.mas-svc.loadbalancer.server.port=8080"

      # 🌟 2. 處理原本 MAS 網域的正常流量
      - "traefik.http.routers.mas-main.rule=Host(`mas.315551018.cs.nycu`)"
      - "traefik.http.routers.mas-main.entrypoints=websecure"
      - "traefik.http.routers.mas-main.tls.certresolver=stepca"
      - "traefik.http.routers.mas-main.service=mas-svc"

      # 🌟 3. 攔截 Matrix 網域的認證 API
      - "traefik.http.routers.mas-auth.rule=Host(`matrix.315551018.cs.nycu`) && (PathPrefix(`/_matrix/client/v3/login`) || PathPrefix(`/_matrix/client/r0/login`) || PathPrefix(`/_matrix/client/v3/logout`) || PathPrefix(`/_matrix/client/v3/refresh`))"
      - "traefik.http.routers.mas-auth.priority=1000"
      - "traefik.http.routers.mas-auth.entrypoints=websecure"
      - "traefik.http.routers.mas-auth.tls.certresolver=stepca"
      - "traefik.http.routers.mas-auth.service=mas-svc"

```

> ⚠️ **註**：先去 Keycloak 建立一個 User 給 shortener bot 使用，然後開啟 `https://app.element.io` 登入該帳號拿取你的 `access token`。

### 環境準備（安裝 Node.js 與 pnpm）

```bash
# 下載並安裝 nvm：
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# 載入 nvm 環境變數：
\. "$HOME/.nvm/nvm.sh"

# 下載並安裝 Node.js 24：
nvm install 24
node -v # 應該要印出 v24.x.x

# 下載並安裝 pnpm：
corepack enable pnpm
pnpm -v

```

### 初始化專案與安裝套件

```bash
mkdir -p /home/judge/hw3/shortener
cd /home/judge/hw3/shortener

# 初始化專案
pnpm init

# 安裝正式環境所需的套件
pnpm add express pg prom-client express-basic-auth jsonwebtoken express-jwt jwks-rsa dotenv matrix-bot-sdk

# 安裝開發環境所需的型別與工具
pnpm add -D typescript @types/express @types/pg @types/jsonwebtoken ts-node nodemon

# 初始化 TypeScript 設定檔
pnpm exec tsc --init

```

### 撰寫主程式 `src/index.ts`

```bash
mkdir -p src
vim src/index.ts
```

```typescript
import express, { type Request, type Response, type NextFunction } from 'express';
import { Pool } from 'pg';
import crypto from 'crypto';
import os from 'os';
import promClient from 'prom-client';
import basicAuth from 'express-basic-auth';
import { expressjwt, type GetVerificationKey } from 'express-jwt';
import jwksRsa from 'jwks-rsa';
import { MatrixClient, SimpleFsStorageProvider, AutojoinRoomsMixin } from 'matrix-bot-sdk';

// ==========================================
// 0. 基礎設定與環境變數
// ==========================================
const app = express();
app.use(express.json()); // 解析 POST body 的 JSON

const pool = new Pool({
    host: process.env.PGHOST || 'postgres',
    database: 'uurl',
    user: process.env.PGUSER || 'uurl',
    password: process.env.PGPASSWORD || 'uurl_password',
    port: 5432,
});

const MATRIX_ACCESS_TOKEN = process.env.MATRIX_TOKEN || '';
const HOMESERVER_URL = "https://matrix.315551018.cs.nycu";

// ==========================================
// 1. 監控指標：Prometheus Setup
// ==========================================
const containerId = os.hostname().substring(0, 12);
const register = new promClient.Registry();

register.setDefaultLabels({ container: containerId });

const redirectsTotal = new promClient.Counter({
    name: 'sa_redirects_total',
    help: 'number of successful redirects',
    registers: [register],
});

const redirectsNotFoundTotal = new promClient.Counter({
    name: 'sa_redirects_not_found_total',
    help: 'number of redirects failed with user given wrong short code',
    registers: [register],
});

const createdUrlsTotal = new promClient.Counter({
    name: 'sa_created_urls_total',
    help: 'number of short URLs created by your service',
    registers: [register],
});

// ==========================================
// 全域 Request/Response Logger [新增 LOG]
// 移到前面，讓所有請求（包含 Auth 失敗）都能先被記錄
// ==========================================
app.use((req, res, next) => {
    const ip = req.headers['x-forwarded-for'] || req.socket.remoteAddress;
    console.log(`[REQ] ${req.method} ${req.url} | IP: ${ip}`);
    
    res.on('finish', () => {
        console.log(`[RES] ${req.method} ${req.url} | Status: ${res.statusCode}`);
    });

    next();
});

// ==========================================
// 2. 身份驗證 Middlewares
// ==========================================
const metricsAuth = basicAuth({
    users: { 'sa': '315551018' },
    challenge: true,
    unauthorizedResponse: (req: Request) => {
        console.warn(`[AUTH_WARN] Metrics Basic Auth 失敗 | IP: ${req.ip}`); 
        return 'Unauthorized';
    },
});

const checkOidcJwt = expressjwt({
    secret: jwksRsa.expressJwtSecret({
        cache: true,
        rateLimit: true,
        jwksRequestsPerMinute: 5,
        jwksUri: 'http://auth:8080/realms/master/protocol/openid-connect/certs',
    }) as GetVerificationKey,
    issuer: 'https://auth.315551018.cs.nycu/realms/master',
    algorithms: ['RS256'],
    requestProperty: 'auth',
});

// ==========================================
// 3. API 路由實作 (Express)
// ==========================================

app.get('/healthz', async (req, res) => {
    try {
        await pool.query('SELECT 1');
        res.status(200).send('OK');
    } catch (err) {
        console.error('[DB_ERROR] Healthcheck 失敗:', err); // [新增 LOG]
        res.status(503).send('DB unavailable');
    }
});

app.post('/api/shorten', checkOidcJwt, async (req: Request, res: Response) => {
    const { url } = req.body;
    const creator = (req as any).auth?.sub || 'unknown';

    console.log(`[API_SHORTEN] 收到請求 | Creator (JWT sub): ${creator} | Target URL: ${url}`); // [新增 LOG]

    if (!url) {
        console.warn(`[API_SHORTEN] 缺少 URL 參數`); // [新增 LOG]
        return res.status(400).json({ error: 'URL is required' });
    }

    const shortCode = crypto.randomBytes(4).toString('base64url').substring(0, 6);

    try {
        await pool.query(
            'INSERT INTO urls (creator, original_url, short_code) VALUES ($1, $2, $3)',
            [creator, url, shortCode]
        );

        createdUrlsTotal.inc();
        console.log(`[API_SHORTEN] 成功建立短網址 | Code: ${shortCode}`); // [新增 LOG]
        return res.status(201).json({ short_code: shortCode });
    } catch (err) {
        console.error('[DB_ERROR] DB Insert Error in /api/shorten:', err);
        return res.status(500).json({ error: 'Internal Server Error' });
    }
});

app.get('/-/:short_code', checkOidcJwt, async (req: Request, res: Response) => {
    const { short_code } = req.params;
    const visitor = (req as any).auth?.sub || 'unknown';

    console.log(`[API_REDIRECT] 收到轉址請求 | Visitor: ${visitor} | Code: ${short_code}`); // [新增 LOG]

    try {
        const result = await pool.query(
            'SELECT original_url FROM urls WHERE short_code = $1',
            [short_code]
        );

        if (result.rows.length > 0) {
            redirectsTotal.inc();
            const target = result.rows[0].original_url;
            console.log(`[API_REDIRECT] 找到網址，準備轉址至: ${target}`); // [新增 LOG]
            return res.redirect(303, target);
        } else {
            redirectsNotFoundTotal.inc();
            console.warn(`[API_REDIRECT] 找不到對應的短網址 | Code: ${short_code}`); // [新增 LOG]
            return res.status(404).send("Can't find original URL by given short code.");
        }
    } catch (err) {
        console.error('[DB_ERROR] DB Query Error in /-:short_code:', err);
        return res.status(500).send('Internal Server Error');
    }
});

app.get('/metrics', metricsAuth, async (req: Request, res: Response) => {
    console.log(`[API_METRICS] 成功通過 Basic Auth，正在輸出 Metrics`); // [新增 LOG]
    try {
        res.set('Content-Type', register.contentType);
        res.end(await register.metrics());
    } catch (ex) {
        console.error('[METRICS_ERROR] 匯出 Metrics 失敗:', ex); // [新增 LOG]
        res.status(500).end(String(ex));
    }
});

// ==========================================
// 全域錯誤處理 (攔截 JWT 驗證錯誤) [新增 LOG]
// ==========================================
app.use((err: any, req: Request, res: Response, next: NextFunction) => {
    if (err.name === 'UnauthorizedError') {
        console.error(`[AUTH_ERROR] JWT 驗證失敗 | URL: ${req.url} | 原因: ${err.message}`);
        res.status(401).json({ error: 'Invalid token', details: err.message });
    } else {
        next(err);
    }
});


// ==========================================
// 4. Matrix Bot 實作
// ==========================================
const storage = new SimpleFsStorageProvider("bot-sync.json");
const matrixClient = new MatrixClient(HOMESERVER_URL, MATRIX_ACCESS_TOKEN, storage);

matrixClient.on("room.invite", async (roomId) => {
    try {
        console.log(`[MATRIX_EVENT] 收到房間邀請 | RoomID: ${roomId}`); // [新增 LOG]
        await matrixClient.joinRoom(roomId);
        console.log(`[MATRIX_EVENT] 成功加入房間 | RoomID: ${roomId}`); // [新增 LOG]
    } catch (e) {
        console.error(`[MATRIX_ERROR] 無法加入房間 ${roomId}:`, e);
    }
});

matrixClient.on("room.message", async (roomId, event) => {
    const botUserId = await matrixClient.getUserId();
    if (event.sender === botUserId) return; // 忽略自己發送的訊息

    const content = event.content;
    if (!content || content.msgtype !== "m.text") return;

    const body = content.body.trim();
    if (!body.startsWith("!sa")) return;

    console.log(`[MATRIX_CMD] 收到指令 | Sender: ${event.sender} | Room: ${roomId} | Body: ${body}`); // [新增 LOG]

    const args = body.split(/\s+/);
    const command = args[1];

    if (command === "shorten") {
        const urlToShorten = args[2];
        let isValidUrl = false;

        try {
            if (urlToShorten) {
                new URL(urlToShorten);
                isValidUrl = true;
            }
        } catch (e) {
            isValidUrl = false;
        }

        if (!isValidUrl) {
            console.warn(`[MATRIX_CMD] Shorten 失敗，無效的 URL | Input: ${urlToShorten}`); // [新增 LOG]
            await matrixClient.sendText(roomId, "Usage: !sa shorten <url>");
            return;
        }

        const shortCode = crypto.randomBytes(4).toString('base64url').substring(0, 6);
        const creator = event.sender;

        try {
            await pool.query(
                'INSERT INTO urls (creator, original_url, short_code) VALUES ($1, $2, $3)',
                [creator, urlToShorten, shortCode]
            );

            createdUrlsTotal.inc();
            const shortUrl = `https://i.315551018.cs.nycu/-/${shortCode}`;
            console.log(`[MATRIX_CMD] 成功建立短網址 | Sender: ${creator} | Code: ${shortCode}`); // [新增 LOG]
            await matrixClient.sendText(roomId, shortUrl);
        } catch (err) {
            console.error("[DB_ERROR] Matrix Bot Insert Error:", err);
            await matrixClient.replyNotice(roomId, event, "Internal Server Error");
        }
    }
    else if (command === "get") {
        const shortCode = args[2];

        if (!shortCode) {
            await matrixClient.sendText(roomId, "Usage: !sa get <short_code>");
            return;
        }

        try {
            const result = await pool.query(
                'SELECT original_url FROM urls WHERE short_code = $1',
                [shortCode]
            );

            if (result.rows.length > 0) {
                console.log(`[MATRIX_CMD] Get 成功 | Code: ${shortCode} -> ${result.rows[0].original_url}`); // [新增 LOG]
                await matrixClient.sendText(roomId, result.rows[0].original_url);
            } else {
                console.warn(`[MATRIX_CMD] Get 失敗，找不到紀錄 | Code: ${shortCode}`); // [新增 LOG]
                await matrixClient.sendText(roomId, "Usage: !sa get <short_code>"); // 原本的邏輯，你也可以考慮回傳 "Not found"
            }
        } catch (err) {
            console.error("[DB_ERROR] Bot DB Query Error:", err);
        }
    }
    else if (command === "list") {
        const creator = event.sender;

        try {
            const result = await pool.query(
                'SELECT short_code, original_url FROM urls WHERE creator = $1',
                [creator]
            );

            if (result.rows.length > 0) {
                console.log(`[MATRIX_CMD] List 成功 | Sender: ${creator} | 數量: ${result.rows.length}`); // [新增 LOG]
                const listLines = result.rows.map(row => `${row.short_code} => ${row.original_url}`);
                await matrixClient.sendText(roomId, listLines.join("\n"));
            } else {
                console.log(`[MATRIX_CMD] List 成功，但無資料 | Sender: ${creator}`); // [新增 LOG]
                await matrixClient.sendText(roomId, "No URLs found.");
            }
        } catch (err) {
            console.error("[DB_ERROR] Bot DB List Error:", err);
        }
    }
    else {
        console.warn(`[MATRIX_CMD] 未知指令 | Sender: ${event.sender} | Command: ${command}`); // [新增 LOG]
        await matrixClient.sendText(roomId, "Usage: !sa {shorten|get|list}");
    }
});

// ==========================================
// 啟動 Matrix Bot（自動重試）
// ==========================================
async function startMatrixBot(retryInterval = 5000): Promise<void> {
    while (true) {
        try {
            console.log("🔄 [MATRIX_SYS] Connecting to Matrix...");
            await matrixClient.start();
            console.log("🚀 [MATRIX_SYS] Matrix Bot 已成功啟動並連線至 Synapse！");
            return;
        } catch (err) {
            console.error("❌ [MATRIX_ERROR] Matrix Bot 啟動失敗：", err);
            console.log(`⏳ [MATRIX_SYS] ${retryInterval / 1000} 秒後重新嘗試連線...`);
            await new Promise(resolve => setTimeout(resolve, retryInterval));
        }
    }
}

// ==========================================
// 5. 啟動 Express API 伺服器
// ==========================================
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
    console.log(`🌐 [SYS] URL Shortener API is running on port ${PORT}`);
});

const shutdown = () => {
    console.log('🛑 [SYS] 收到關閉訊號，準備優雅下線...');
    matrixClient.stop();
    pool.end(() => {
        console.log('🛑 [DB] PostgreSQL 連線已關閉');
        process.exit(0); 
    });
};
process.on('SIGTERM', shutdown); 
process.on('SIGINT', shutdown);

// ==========================================
// 啟動 Matrix Bot（背景持續重試）
// ==========================================
startMatrixBot().catch(err => {
    console.error("[MATRIX_ERROR] Unexpected Matrix Bot error:", err);
});


```

### 建立 `Dockerfile`

```bash
vim /home/judge/hw3/shortener/Dockerfile
```

```dockerfile
FROM node:22-alpine
WORKDIR /app

# 在容器內全域安裝 pnpm
RUN npm install -g pnpm

# 複製依賴檔案
COPY package.json pnpm-lock.yaml* ./

# 使用 pnpm 安裝依賴
RUN pnpm install

# 複製其餘原始碼
COPY . .

# 編譯 TypeScript
RUN pnpm exec tsc

EXPOSE 3000
CMD ["node", "dist/index.js"]

```

### 將 Shortener 加入 `docker-compose.yml`

```yaml
  uurl:
    build: ./shortener
    container_name: shortener
    restart: always
    environment:
      - PGUSER=uurl_user
      - PGPASSWORD=uurl_password
      - PGHOST=postgres
      # 🌟 把剛剛獲取的 Element Matrix Token 貼在這裡！
      - MATRIX_TOKEN=syt_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
      # 關掉 Node.js 的 TLS 自簽憑證拒絕限制
      - NODE_TLS_REJECT_UNAUTHORIZED=0
    depends_on:
      matrix:
        condition: service_healthy
      mas:
        condition: service_healthy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.shortener.rule=Host(`i.315551018.cs.nycu`)"
      - "traefik.http.routers.shortener.entrypoints=websecure"
      - "traefik.http.routers.shortener.tls.certresolver=stepca"
    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "require('http').get('http://localhost:3000/healthz',r=>process.exit(r.statusCode===200?0:1))"
        ]
      interval: 5s
      timeout: 5s
      retries: 10
```

可以用下面這個來測試一下 shortener 的結果
```bash
TOKEN=$(curl -sk \
-d "grant_type=client_credentials" \
-d "client_id=sa-client" \
-d "client_secret=gItQ7kfz29EdORpp6DlSlzAbF9Xw7KSa" \
https://auth.315551018.cs.nycu/realms/master/protocol/openid-connect/token \
| jq -r .access_token)
```

---

## Logging

### 簽憑證
```bash
openssl genrsa -out vector-client.key 2048
openssl req -new -key vector-client.key -out vector-client.csr -subj "/CN=vector-client"
#這步很重要！mTLS 認證通常會嚴格檢查憑證用途。我們必須加上 clientAuth 標籤，助教的伺服器才不會踢掉你。
echo "extendedKeyUsage = clientAuth" > client.ext

openssl x509 -req -in vector-client.csr \
  -CA sa.crt -CAkey sa.key -CAcreateserial \
  -out vector-client.crt -days 365 -extfile client.ext

cat vector-client.crt sa.crt > vector-client-fullchain.crt

cat vector-client.crt sa.crt sarootca.crt > vector-client-fullchain.crt
```

### 改一下traefik的log讓他保留 user-agent
```yaml
# traefik.yml
accessLog:
  filePath: "/logs/traefik-access.log"
  format: json
  fields:
    headers:
      defaultMode: drop
      names:
        User-Agent: keep
        
# docker-compose.yml
# 👇 加上這行：建立一個共用的 logs 資料夾 (對應助教要求的實體路徑)
      - "/home/judge/hw3/logs:/logs"
```

### 設定vector yaml

```yaml
#docker-compose.yml
vector:
    image: timberio/vector:0.38.0-alpine
    container_name: vector
    restart: always
    volumes:
      # 1. 掛載 Vector 的核心邏輯設定檔
      - "./vector.yaml:/etc/vector/vector.yaml:ro"
      
      # 2. 掛載 Log 資料夾 (讓 Vector 能讀取 Traefik 的 log，並寫出最終的 access.log)
      - "/home/judge/hw3/logs:/logs"
      
      # 3. 掛載憑證資料夾 (為了 mTLS 雙向認證，裡面要有你的 Root CA 和 Client 憑證鍊)
      - "/home/judge/hw3/sarootca.crt:/certs/sarootca.crt:ro"
      - "/home/judge/hw3/vector-client-fullchain.crt:/certs/vector-client-fullchain.crt:ro"
      - "/home/judge/hw3/vector-client.key:/certs/vector-client.key:ro"
    depends_on:
      - traefik
```


### vector的設定檔
```yaml
sources:
  traefik_logs:
    type: file
    include:
      - "/logs/traefik-access.log"
    read_from: end

transforms:
  parse_json:
    type: remap
    inputs:
      - traefik_logs
    source: |-
      # Parse Traefik JSON log if it's not already structured
      parsed, err = parse_json(string!(.message))
      if err == null {
        . = merge!(., parsed)
      }

  filter_probe:
    type: filter
    inputs: ["parse_json"]
    condition: |-
      ua = string(."request_User-Agent") ?? ""
      ua != "sa-probe/315551018"

  # Basic log for /home/judge/hw3/logs/access.log
  format_basic_log:
    type: remap
    inputs: ["filter_probe"]
    source: |-
      uri = string(.RequestPath) ?? "-"
      ua = string(."request_User-Agent") ?? "-"
      host = string(.RequestHost) ?? "-"
      # 將它們串在同一行，中間用空白隔開
      .message = uri + " " + ua + " " + host 


# 準備要送給 OTEL 的欄位（最簡單的方式）
  prepare_otel:
    type: remap
    inputs:
      - filter_probe
    source: |-
      uri = string(.RequestPath) ?? string(.request_path) ?? "/"
      host = string(.RequestHost) ?? string(.request_host) ?? "-"
      method = string(.RequestMethod) ?? string(.request_method) ?? "GET"
      ua = string(."request_User-Agent") ?? string(.request_User_Agent) ?? "-"
      status = to_int(.DownstreamStatus) ?? to_int(.downstream_status) ?? 200

      # 加上 timestamp
      now_ns = to_string(to_unix_timestamp(now(), "nanoseconds"))

      . = {
        "resourceLogs": [
          {
            "resource": {
              "attributes": [
                { "key": "service.name", "value": { "stringValue": "traefik" } }
              ]
            },
            "scopeLogs": [
              {
                "scope": {
                  "name": "vector",
                  "version": "0.0.1"
                },
                "logRecords": [
                  {
                    "timeUnixNano": now_ns,
                    "observedTimeUnixNano": now_ns,
                    "severityNumber": 9,           # INFO
                    "severityText": "INFO",
                    "body": {
                      "stringValue": "traefik access log: " + method + " " + uri
                    },
                    "attributes": [
                      { "key": "sa.uri",         "value": { "stringValue": uri } },
                      { "key": "sa.host",        "value": { "stringValue": host } },
                      { "key": "sa.method",      "value": { "stringValue": method } },
                      { "key": "sa.user_agent",  "value": { "stringValue": ua } },
                      { "key": "sa.status",      "value": { "stringValue": status } }
                    ]
                  }
                ]
              }
            ]
          }
        ]
      }

sinks:
  local_file_out:
    type: file
    inputs:
      - format_basic_log
    path: "/logs/access.log"
    encoding:
      codec: text


  otel_out:
    type: opentelemetry
    inputs:
      - prepare_otel
    protocol:
      type: http
      # uri: "http://192.168.254.145:8081/ingest/otlp/v1/logs"
      uri: "https://ta.315551018.cs.nycu:10001/v1/logs"
      method: post
      encoding:
        codec: otlp
      tls:
        ca_file: "/certs/ca_bundle.crt"           # Your root CA
        crt_file: "/certs/vector-client-fullchain.crt"
        key_file: "/certs/vector-client.key"
        verify_certificate: true   # usually keep enabled
        verify_hostname: true

    batch:
      max_events: 1
      timeout_secs: 1

    request:
      retry_attempts: 999      # 幾乎一直重試
      retry_max_duration_secs: 300
      retry_initial_backoff_secs: 0.5   # 非常重要！縮短初始等待
      retry_jitter: true

    buffer:
      type: memory
      max_events: 2000
      when_full: block

# ======================
# Vector API（用來 debug）
# ======================
api:
  enabled: true
  address: "0.0.0.0:8686"
```

## 最終完整 `docker-compose.yml`

```yaml

services:
  web:
    image: traefik:v3.6.1
    container_name: traefik
    restart: always
    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "443:443/udp"
    environment:
      - LEGO_CA_CERTIFICATES=/certs/sarootca.crt
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./traefik.yml:/etc/traefik/traefik.yml:ro"
      - "./traefik-acme:/letsencrypt"
      - "./traefik-dynamic:/etc/traefik/dynamic"
      - "/home/judge/hw3/sarootca.crt:/certs/sarootca.crt:ro"

      # 前面用acme server自己幫自己簽的
      - "./step/acme.crt:/certs/acme.crt:ro"
      - "./step/acme.key:/certs/acme.key:ro"

      - "/home/judge/hw3/logs:/logs"
    networks:
      default:
        aliases:
          - 315551018.cs.nycu
          - hello.315551018.cs.nycu
          - auth.315551018.cs.nycu
          - matrix.315551018.cs.nycu
          - mas.315551018.cs.nycu
          - i.315551018.cs.nycu
    depends_on:
      - acme
      - auth
      - matrix
      - mas
      - uurl 
  
  vector:
    image: timberio/vector:nightly-2026-06-20-debian
    container_name: vector
    restart: always
    volumes:
      # 1. 掛載 Vector 的核心邏輯設定檔
      - "./vector.yaml:/etc/vector/vector.yaml:ro"
      
      # 2. 掛載 Log 資料夾 (讓 Vector 能讀取 Traefik 的 log，並寫出最終的 access.log)
      - "/home/judge/hw3/logs:/logs"
      
      # 3. 掛載憑證資料夾 (為了 mTLS 雙向認證，裡面要有你的 Root CA 和 Client 憑證鍊)
      - "/home/judge/hw3/ca_bundle.crt:/certs/ca_bundle.crt:ro"
      - "/home/judge/hw3/vector-client-fullchain.crt:/certs/vector-client-fullchain.crt:ro"
      - "/home/judge/hw3/vector-client.key:/certs/vector-client.key:ro"
    depends_on:
      - web
    extra_hosts:
      - "ta.315551018.cs.nycu:192.168.255.123"
  acme:
    image: smallstep/step-ca
    container_name: acme
    restart: always
    environment:
      - STEPPATH=/home/step
    command: /usr/local/bin/step-ca --password-file /home/step/password.txt /home/step/config/ca.json
    networks:
      default:
        aliases:
          - acme.315551018.cs.nycu
    volumes:
      - ./step:/home/step
    healthcheck: 
      disable: true
        
    extra_hosts:
      - "ta.315551018.cs.nycu:192.168.255.123"
    labels:
      - "traefik.enable=true" # 告訴 Traefik 要代理這個容器
      - "traefik.http.routers.acme.rule=Host(`acme.315551018.cs.nycu`)" # 設定路由規則
      - "traefik.http.routers.acme.entrypoints=websecure" # 綁定在 443 port
      - "traefik.http.routers.acme.tls=true" # 告訴 Traefik 這個路由要啟用 TLS
      - "traefik.http.services.acme.loadbalancer.server.port=9000" # 這裡請填寫 step-ca 實際監聽的 Port
      - "traefik.http.services.acme.loadbalancer.server.scheme=https"  

  hello:  
    image: hashicorp/http-echo
    restart: always
    command: -text="I love NYCU NASA 2025"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.hello.rule=Host(`hello.315551018.cs.nycu`)"
      - "traefik.http.routers.hello.entrypoints=websecure"
      - "traefik.http.routers.hello.tls.certresolver=stepca"
      - "traefik.http.routers.hello.middlewares=sec-headers@file"

  auth:
    image: keycloak/keycloak:26.6
    container_name: auth
    restart: always
    command: start --http-enabled=true --proxy-headers=xforwarded
    environment:
      - KC_HOSTNAME=auth.315551018.cs.nycu
      - KC_HOSTNAME_STRICT=false
      # 這是 Keycloak 網頁後台的最高管理員帳密
      - KC_BOOTSTRAP_ADMIN_USERNAME=admin
      - KC_BOOTSTRAP_ADMIN_PASSWORD=admin
      # 是連線到底層 Postgres 資料庫的設定與帳密
      - KC_DB=postgres
      - KC_DB_URL=jdbc:postgresql://postgres:5432/keycloak
      - KC_DB_USERNAME=keycloak_user
      - KC_DB_PASSWORD=keycloak_password
    volumes:
      - auth-data:/data
    networks:
      default:
        aliases:
          - auth.315551018.cs.nycu
    depends_on:
      - postgres
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.auth.rule=Host(`auth.315551018.cs.nycu`)"
      - "traefik.http.routers.auth.entrypoints=websecure"
      - "traefik.http.routers.auth.tls.certresolver=stepca"
      # Keycloak 內建的網頁服務連接埠是 8080
      - "traefik.http.services.auth.loadbalancer.server.port=8080"
      # 安全考量：強制加上我們之前設定好的 HSTS 等安全防禦標頭
      - "traefik.http.routers.auth.middlewares=sec-headers@file"


  matrix:
    image: matrixdotorg/synapse:latest
    container_name: matrix
    restart: always
    volumes:
      - ./homeserver.yaml:/data/homeserver.yaml:ro
      # 給它一個目錄存放 Log、媒體檔案和自動產生的密鑰
      - ./synapse-data:/data
      # ⚠️ 關鍵！把 Step-CA 憑證掛載進去，讓 Synapse 信任 MAS 的憑證
      - /home/judge/hw3/mas_ca_bundle.crt:/etc/ssl/certs/ca-certificates.crt:ro
    environment:
      # 告訴 Synapse 去哪裡找信任的根憑證 (Python 的 requests 庫支援這個變數)
      - SYNAPSE_LOG_LEVEL=INFO
    networks:
      default:
        aliases:
          - matrix.315551018.cs.nycu
    healthcheck:
      disable: true
    depends_on:
      - postgres
      - mas
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.matrix.rule=Host(`matrix.315551018.cs.nycu`)"
      - "traefik.http.routers.matrix.entrypoints=websecure"
      - "traefik.http.routers.matrix.tls.certresolver=stepca"
      - "traefik.http.services.matrix.loadbalancer.server.port=8008"
      - "traefik.http.routers.matrix.middlewares=sec-headers@file"
  
  mas:
    image: ghcr.io/element-hq/matrix-authentication-service:1.19.0
    container_name: mas
    restart: always
    volumes:
      - ./mas-signing.pem:/mas-keys/mas-signing.pem:ro
      - ./mas-config.yaml:/mas.yaml:ro
      - /home/judge/hw3/mas_ca_bundle.crt:/etc/ssl/certs/ca-certificates.crt:ro

      - mas-data:/data

    environment:
      - MAS_CONFIG=/mas.yaml
    depends_on:
      - postgres
      - auth
    networks:
      default:
        aliases:
          - mas.315551018.cs.nycu
    labels:
      - "traefik.enable=true"
      
      # 🌟 1. 建立唯一的內部接收器
      - "traefik.http.services.mas-svc.loadbalancer.server.port=8080"

      # 🌟 2. 處理原本 MAS 網域的正常流量
      - "traefik.http.routers.mas-main.rule=Host(`mas.315551018.cs.nycu`)"
      - "traefik.http.routers.mas-main.entrypoints=websecure"
      - "traefik.http.routers.mas-main.tls.certresolver=stepca"
      - "traefik.http.routers.mas-main.service=mas-svc"

      # 🌟 3. 攔截 Matrix 網域的認證 API (使用 Traefik v3 的標準 OR 語法)
      - "traefik.http.routers.mas-auth.rule=Host(`matrix.315551018.cs.nycu`) && (PathPrefix(`/_matrix/client/v3/login`) || PathPrefix(`/_matrix/client/r0/login`) || PathPrefix(`/_matrix/client/v3/logout`) || PathPrefix(`/_matrix/client/v3/refresh`))"
      - "traefik.http.routers.mas-auth.priority=1000"
      - "traefik.http.routers.mas-auth.entrypoints=websecure"
      - "traefik.http.routers.mas-auth.tls.certresolver=stepca"
      - "traefik.http.routers.mas-auth.service=mas-svc"

  uurl:
    build: ./shortener
    container_name: shortener
    restart: always
    environment:
      - PGUSER=judge     # 替換為你的實際 DB 帳號
      - PGPASSWORD=judge # 替換為你的實際 DB 密碼
      - PGHOST=postgres         # 對接同一個 DB 容器

      - MATRIX_TOKEN=mat_yvj6T2NCTLXMR8QNifqqBWY0lrebIp_Q6KNi2


      - NODE_TLS_REJECT_UNAUTHORIZED=0      
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.shortener.rule=Host(`i.315551018.cs.nycu`)"
      - "traefik.http.routers.shortener.entrypoints=websecure"
      - "traefik.http.routers.shortener.tls.certresolver=stepca"
#    healthcheck:
#      test:
#        [
#          "CMD",
#          "node",
#          "-e",
#          "require('http').get('http://localhost:3000/healthz',r=>{console.log(r.statusCode);process.exit(r.statusCode===200?0:1)}).on('error',e=>{console.error(e);process.exit(1)})"  
#      ]
#      interval: 5s
#      timeout: 5s
#      retries: 10

  postgres:
    # Standard PostgreSQL image
    image: postgres:17-alpine
    container_name: postgres
    restart: always
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: password
    entrypoint: ["/pg-entrypoint.sh"]
    volumes:
      - ./pg-entrypoint.sh:/pg-entrypoint.sh:ro
      # 掛載 TLS 憑證 (包含 RootCA, Server Cert, Private Key)
      - /home/judge/hw3/ca_bundle.crt:/certs/sarootca.crt:ro
      - /home/judge/hw3/postgres_chain.crt:/certs/postgres.crt:ro
      - /home/judge/hw3/postgres.key:/certs/postgres.key:ro
      - postgres-data:/var/lib/postgresql/data
    networks:
      default: 
       aliases:
         - postgres.315551018.cs.nycu

# Defined volumes ensure "Service Health" persists after `docker compose down`
volumes:
  auth-data:
  mas-data:
  postgres-data:
```