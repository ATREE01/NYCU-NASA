# 簡化版 DNS 設定教學


## 1. Authoritative DNS Server

### 1.1 設定 IP（OpenWRT 使用 luci）
- **Auth**: 192.168.12.53
- **Rslv**: 192.168.12.153

### 1.2 安裝 Bind 與配置

#### 安裝 bind9
```shell
sudo apt install bind9
```

#### 設定區域檔案引用
編輯檔案 `/etc/bind/named.conf.local`，加入：
```conf
include "/etc/bind/zones.nasa";
```

#### 建立 Zone 設定檔
編輯 `/etc/bind/zones.nasa`，新增：
```conf
zone "12.nasa" {
  type master;
  file "/etc/bind/db.12.nasa.signed";
};

zone "12.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/db.12.168.192.rev.signed";
};
```

#### 設定正向 Zone
將預設檔複製並編輯：
```shell
sudo cp /etc/bind/db.local /etc/bind/db.12.nasa
sudo vim /etc/bind/db.12.nasa
```
內容範例：
```conf
$TTL    38640
@       IN      SOA     ns1.12.nasa. root.12.nasa. (
            20250328   ; Serial
            604800     ; Refresh
            86400      ; Retry
            2419200    ; Expire
            604800 )   ; Negative Cache TTL
;
@       IN      NS      ns1.12.nasa.
@       IN      A       127.0.0.1
@       IN      AAAA    ::1
whoami  IN      A       10.113.12.1
dns     IN      A       192.168.12.153
ns1     IN      A       192.168.12.53
```

#### 設定反向 Zone
複製並編輯反向檔案：
```shell
sudo cp /etc/bind/db.local /etc/bind/db.12.168.192.rev
sudo vim /etc/bind/db.12.168.192.rev
```
範例內容：
```conf
$TTL 38400
12.168.192.in-addr.arpa.        IN      SOA     ns1.12.nasa. root.12.nasa. (
            20250512
            604800
            86400
            2419200
            86400 )
12.168.192.in-addr.arpa.        IN      NS      ns1.12.nasa.
153.12.168.192.in-addr.arpa.    IN      PTR     dns.12.nasa.
53.12.168.192.in-addr.arpa.     IN      PTR     ns1.12.nasa.
25.12.168.192.in-addr.arpa.     IN      PTR     mail.12.nasa.
```

#### 關閉遞迴解析
編輯 `/etc/bind/named.conf.options` 中的 options block，加上：
```conf
recursion no;
```

---

## 2. Resolver 配置

### 2.1 安裝 Bind 與引用 Zone
重複安裝動作後，在 `/etc/bind/named.conf.local` 加入：
```conf
include "/etc/bind/zones.nasa";
```

### 2.2 轉發設定
編輯 `/etc/bind/zones.nasa`，設定 forward zone：
```conf
zone "nasa" {
  type forward;
  forward first;
  forwarders { 192.168.254.3; };
};

zone "168.192.in-addr.arpa" {
  type forward;
  forward first;
  forwarders { 192.168.254.3; };
};
```

### 2.3 DNSSEC 與 Forwarders
在 `/etc/bind/named.conf.options` 中設定：
```conf
dnssec-validation yes;
empty-zones-enable no;

// 在 forwarders 區塊內加入
1.1.1.1;
```

---

## 3. ns1 DNSSEC 設定

### 3.1 產生金鑰
執行：
```shell
sudo dnssec-keygen -f KSK -a 13 12.nasa
sudo dnssec-keygen -a 13 12.nasa
```

### 3.2 將金鑰公鑰加入 Zone 檔
編輯 `/etc/bind/26.nasa.hosts`：
```conf
$INCLUDE "/etc/bind/K12.nasa.+013+48291.key"
$INCLUDE "/etc/bind/K12.nasa.+013+48479.key"
```

### 3.3 簽章並更新 Zone 檔
產生簽章：
```shell
sudo dnssec-signzone -g -o 12.nasa -k K12.nasa.+013+48291.key db.12.nasa K12.nasa.+013+48479.key
```
此動作將產生 `26.nasa.hosts.signed`，請更新 `/etc/bind/zones.nasa` 的 Zone 設定：
```conf
zone "26.nasa" {
  type master;
  file "/etc/bind/26.nasa.hosts.signed";
};
```
注意：DS record 將放在 `dsset-26.nasa.` 內，上傳前請確保沒有多餘空白。

同樣，反向 Zone 需新增 DS record，步驟相同，但對象改為 `12.168.192.in-addr.arpa`。

---

## 4. DoH 設定

下載並執行 doh-proxy：
```shell
sudo doh-proxy -i /home/ppodds/fullchain.pem -I /home/ppodds/privkey.pem -H dns.26.nasa -l 0.0.0.0:443 -u 127.0.0.1:53
```
fullchain.pem 與 privkey.pem 為預先提供的憑證。

---

## 5. DoT 設定

### 5.1 設定 TLS 資訊
在 `/etc/bind/named.conf.options` 添加：
```conf
tls local-tls {
  key-file "/etc/bind/private.pem";
  cert-file "/etc/bind/fullchain.pem";
};
```
確保檔案權限設定為 `root:bind`、660，並避免多個進程同時存取。

### 5.2 調整監聽設定
依需求設定：
```conf
listen-on { any; };
listen-on port 853 tls local-tls { any; };
```