
# HW 2-1 Manage SFTP file server
##  架構總覽

* **根目錄 ($SFTP_ROOT):** `/mnt/hw2/sftp`
* **使用者角色:**
* **Administrator (管理員):** `sysadm` (具備完整 SSH/SFTP 權限，不受 Chroot 限制)
* **Registered Users (註冊使用者):** `sftp-u1`, `sftp-u2` (僅限 SFTP，被關在 Chroot 內)
* **Anonymous (訪客):** `anonymous` (僅限 SFTP，被關在 Chroot 內，僅具備唯讀與下載權限)

* **認證方式:** 所有使用者皆透過同一組 SSH 公鑰 (Judge Key) 登入，禁止密碼連線。

---

## 🛠️ Step 1: 建立使用者與群組

首先，建立使用者帳號、指定 Shell，並建立專屬的 SFTP 群組以利後續權限管理。

```bash
# 1. 建立管理員 (配發正常 bash Shell 並加入 sudo 群組)
sudo useradd -m -s /bin/bash -G sudo sysadm

# 2. 建立受限使用者 (Shell 設為 nologin，禁止使用一般終端機)
sudo useradd -m -s /usr/sbin/nologin sftp-u1
sudo useradd -m -s /usr/sbin/nologin sftp-u2
sudo useradd -m -s /usr/sbin/nologin anonymous

# 3. 建立 SFTP 專用群組，並將註冊使用者加入
sudo groupadd sftp-users
sudo usermod -aG sftp-users sftp-u1
sudo usermod -aG sftp-users sftp-u2

```

---

## 🔑 Step 2: 佈署 SSH 公鑰 (Judge Key)

將公鑰發配給所有使用者，確保他們都能使用同一把私鑰登入。
*(請將 `PASTE_JUDGE_PUBLIC_KEY_HERE` 替換為真實的公鑰字串)*

```bash
JUDGE_KEY="PASTE_JUDGE_PUBLIC_KEY_HERE"

for user in sysadm sftp-u1 sftp-u2 anonymous; do
    sudo mkdir -p /home/$user/.ssh
    echo "$JUDGE_KEY" | sudo tee /home/$user/.ssh/authorized_keys > /dev/null
    
    # 設定嚴格的 SSH 目錄權限
    sudo chmod 700 /home/$user/.ssh
    sudo chmod 600 /home/$user/.ssh/authorized_keys
    sudo chown -R $user:$user /home/$user/.ssh
done

```

---

## 📁 Step 3: 建立目錄與設定群組權限

這是確保「註冊使用者可以讀寫但只能刪自己的檔案，訪客只能唯讀，管理員擁有全權」的核心步驟。

```bash
# 1. 建立目錄架構
sudo mkdir -p /mnt/hw2/sftp/public
sudo mkdir -p /mnt/hw2/sftp/private/hidden

# 2. 設定 Chroot 根目錄 (必須為 root 擁有且為 755)
sudo chown root:root /mnt/hw2/sftp
sudo chmod 755 /mnt/hw2/sftp

# 3. 設定 Public 目錄 (群組法 + Sticky Bit)
# 擁有者：管理員 / 群組：sftp-users
sudo chown sysadm:sftp-users /mnt/hw2/sftp/public
# 權限 1775：1(Sticky Bit防誤刪), 7(sysadm完整權限), 7(sftp-users完整權限), 0(others無權限)
sudo chmod 1770 /mnt/hw2/sftp/public

# 4. 設定 Private 目錄 (Execute-Only 隱藏法)
# 擁有者：管理員 / 群組：管理員
sudo chown -R sysadm:sysadm /mnt/hw2/sftp/private
# 權限 711：其他人只有 x (可 cd 進入)，沒有 r (無法 ls 偷看)
sudo chmod 711 /mnt/hw2/sftp/private
sudo chmod 711 /mnt/hw2/sftp/private/hidden

# 5. 放置並設定寶藏檔案
echo "Congratulations! You found the treasure!" | sudo tee /mnt/hw2/sftp/private/hidden/treasure
sudo chmod 644 /mnt/hw2/sftp/private/hidden/treasure


# 6.
# 強制給予 sysadm 對現有檔案的 rwx 權限
# 透過 ACL 單獨對 anonymous 開放「進入與讀取」權限 (達成表格 V 需求)

# 預設 ACL：確保「未來上傳的檔案」依然符合表格規範
sudo setfacl -d -m u:sysadm:rwx /mnt/hw2/sftp/public       # sysadm 永遠能完全控制
sudo setfacl -d -m g:sftp-users:rwx /mnt/hw2/sftp/public   # 註冊用戶能互讀互寫
sudo setfacl -d -m u:anonymous:r-x /mnt/hw2/sftp/public    # 匿名者永遠只能讀 (get)
sudo setfacl -d -m o::--- /mnt/hw2/sftp/public             # 其他人絕對禁止
```

---

## 🛡️ Step 4: 配置 SSH Daemon 規則

修改 `/etc/ssh/sshd_config` 以套用 Chroot 監牢，並利用 `umask` 確保上傳的檔案自動拔除其他人的權限。

```bash
sudo nano /etc/ssh/sshd_config

```

### 1. 全域設定 (針對管理員)

找到 `Subsystem sftp` 該行並修改如下，讓管理員的 SFTP 預設起點落在 `$SFTP_ROOT`：

```text
Subsystem sftp /usr/lib/openssh/sftp-server -d /mnt/hw2/sftp

```

### 2. Match 區塊 (針對受限使用者)

在檔案的**最底端**加入以下設定：

```text
Match User sftp-u1,sftp-u2,anonymous
    # 將活動範圍鎖死在 $SFTP_ROOT
    ChrootDirectory /mnt/hw2/sftp
    
    # 強制 SFTP 模式，預設目錄為根目錄(/)，並加上 umask 0027 拔除 others 的權限
    ForceCommand internal-sftp -u 0027 -d /
    
    # 關閉網路通道與圖形轉發，確保資安
    AllowTcpForwarding no
    X11Forwarding no

```

---

##  Step 5: 檢查與重啟服務

設定完成後，檢查語法並重啟 SSH 服務。

```bash
# 檢查 sshd_config 語法 (如果沒有輸出代表語法正確)
sudo sshd -t

# 重新啟動 SSH 服務套用設定
sudo systemctl restart ssh

```

# HW2-2: SFTP logging and Monitoring daemon

## Step 1: 建立 venv 與安裝監控套件

```bash
sudo apt update
sudo apt install python3-venv

# 建立獨立虛擬環境
sudo python3 -m venv /opt/sftp_venv

# 安裝 watchdog 套件 (取代過時的 pyinotify)
sudo /opt/sftp_venv/bin/pip install watchdog

```

## Step 2: 建立 python script

```bash
sudo vim /usr/local/bin/sftp_watchd
sudo chmod +x /usr/local/bin/sftp_watchd
```

```python
#!/opt/sftp_venv/bin/python
import os
import pwd
import hashlib
import shutil
import time
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

SFTP_ROOT = "/mnt/hw2/sftp"
WATCH_DIR = os.path.join(SFTP_ROOT, "public")
VIOLATED_DIR = os.path.join(SFTP_ROOT, "private/.violated")

PROHIBITED_MD5 = {
    "209c6ec9c78249031b49b29aef2ee264",
    "288d9c9c945b95bcca9632a594f8ebfc",
    "84cbc60c4b110591d7c287da21067e70",
    "d214f689364c2c19bd02aeb087a354e2",
    "8806b1882ec4ee5f88f4e11641965285"
}

os.makedirs(VIOLATED_DIR, exist_ok=True)

class UploadHandler(FileSystemEventHandler):
    # 當檔案關閉寫入時觸發
    def on_closed(self, event):
        if event.is_directory:
            return
            
        filepath = event.src_path
        filename = os.path.basename(filepath)

        if not os.path.isfile(filepath):
            return

        try:
            stat_info = os.stat(filepath)
            user = pwd.getpwuid(stat_info.st_uid).pw_name
        except Exception:
            user = "unknown"

        # 檢查 ELF
        try:
            with open(filepath, 'rb') as f:
                header = f.read(4)
                if header == b'\x7fELF':
                    print(f"File {filepath} uploaded by user {user} is an ELF file. Moving to violated directory", flush=True)
                    shutil.move(filepath, os.path.join(VIOLATED_DIR, filename))
                    return
        except Exception:
            pass

        # 檢查 MD5
        try:
            hasher = hashlib.md5()
            with open(filepath, 'rb') as f:
                for chunk in iter(lambda: f.read(4096), b""):
                    hasher.update(chunk)
            
            file_hash = hasher.hexdigest()
            if file_hash in PROHIBITED_MD5:
                print(f"File {filepath} uploaded by user {user} matches prohibited MD5 hash. Moving to violated directory", flush=True)
                shutil.move(filepath, os.path.join(VIOLATED_DIR, filename))
                return
        except Exception:
            pass

if __name__ == "__main__":
    event_handler = UploadHandler()
    observer = Observer()
    observer.schedule(event_handler, WATCH_DIR, recursive=True)
    observer.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()

```

## Step 3: 註冊 systemd

```bash
sudo vim /etc/systemd/system/sftp_watchd.service
```

```ini
[Unit]
Description=SFTP Upload Watch Daemon
After=network.target

[Service]
Type=simple
# 執行剛才寫好的程式
ExecStart=/usr/local/bin/sftp_watchd
# 如果程式意外崩潰，自動重啟
Restart=on-failure
# 以 root 權限執行 (因為需要讀取所有人檔案並強制移動到 private)
User=root
# 將標準輸出與錯誤交給 systemd 的 journald 處理
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target

```

## Step 4: 載入 Watch Daemon

```bash
# 1. 重新載入 Systemd 設定檔
sudo systemctl daemon-reload

# 2. 啟動服務
sudo systemctl start sftp_watchd

# 3. 設定開機自動啟動
sudo systemctl enable sftp_watchd

# 4. 確認服務狀態 (應該會看到綠色的 active (running))
sudo systemctl status sftp_watchd
```

---

## Step 5: 配置 SFTP 獨立純淨日誌系統 (雙管道輸出)

目標：將 SFTP 操作獨立記錄，並支援 `journalctl` 與 `/var/log/sftp.log` 查詢，且區分 `sftp-server` 與 `internal-sftp` 標籤。

### 5.1 修改 SSH Daemon 設定

```bash
sudo vim /etc/ssh/sshd_config
```

確保設定檔包含以下內容（區分標籤與開啟 INFO 級別）：

```text
# 全域設定：供 sysadm 使用，產生 sftp-server 標籤
Subsystem       sftp    /usr/lib/openssh/sftp-server -d /mnt/hw2/sftp -l INFO

# Match 區塊：供受限使用者使用，產生 internal-sftp 標籤
Match User sftp-u1,sftp-u2,anonymous
        ChrootDirectory /mnt/hw2/sftp
        ForceCommand internal-sftp -u 0027 -d / -l INFO
        AllowTcpForwarding no
        X11Forwarding no

```

### 5.2 建立 Chroot 監牢日誌傳送門 (Bind Mount)

為了解決受限使用者被關在 Chroot 內無法送出 log 的問題，將系統的 journal 通訊端映射進監牢。

```bash
# 建立 dev 目錄與掛載點
sudo mkdir -p /mnt/hw2/sftp/dev
sudo touch /mnt/hw2/sftp/dev/log

# 執行綁定掛載
sudo mount --bind /run/systemd/journal/dev-log /mnt/hw2/sftp/dev/log

# 寫入 fstab 以確保開機自動掛載
echo '/run/systemd/journal/dev-log /mnt/hw2/sftp/dev/log none bind 0 0' | sudo tee -a /etc/fstab

```

### 5.3 建立 Rsyslog 純淨過濾規則

確保系統已安裝 `rsyslog` (`sudo apt install rsyslog`)。

> rsyslog會自動從journal那裏把Log抓過蘭所以會從上面的log那裡把東西抓出來

```bash
sudo vim /etc/rsyslog.d/10-sftp.conf
```

寫入精準過濾規則，攔截 SFTP 紀錄並阻止其流入 `syslog` 或 `auth.log`：



```text
if $programname == "internal-sftp" or $programname == "sftp-server" then {
    action(type="omfile" file="/var/log/sftp.log" FileCreateMode="0646")
    stop
}
### ===== 下面那部份是舊的寫法 貼上面這個就好 =====

# 當程式名稱為 sftp-server 時，寫入專屬 log 並停止後續導向
:programname, isequal, "sftp-server" /var/log/sftp.log
& stop

# 當程式名稱為 internal-sftp 時，寫入專屬 log 並停止後續導向
:programname, isequal, "internal-sftp" /var/log/sftp.log
& stop

```

### 5.4 重啟服務與驗證

```bash
sudo systemctl restart ssh
sudo systemctl restart rsyslog

```

**驗證指令（需先分別用 sysadm 與 sftp-u1 登入測試後）：**

```bash
# 驗證 1: 透過 journalctl 雙標籤查詢
sudo journalctl -t sftp-server -t internal-sftp

# 驗證 2: 查看實體獨立日誌檔
sudo cat /var/log/sftp.log

```

# HW2-3: BTRFS on LVM And BTRFS snapshot

## Step 1: 切分磁碟空間建立PV、LV

```bash
sudo parted -s /dev/sdb mklabel gpt
sudo parted -s /dev/sdb mkpart primary 0% 25%
sudo parted -s /dev/sdb mkpart primary 25% 50%
sudo parted -s /dev/sdb mkpart primary 50% 75%
sudo parted -s /dev/sdb mkpart primary 75% 100%
```

```bash
sudo apt install lvm2
sudo pvcreate /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4
sudo vgcreate SA2025vg /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4
```

```bash
# --type raid10 指定磁碟陣列類型
# -l 100%FREE 代表用盡 VG 所有空間
# -n SARaid10 是 LV 的名稱
sudo lvcreate --type raid10 -l 100%FREE -n SARaid10 SA2025vg

# 將剛切好的 LV 格式化為 BTRFS 檔案系統，並打上標籤 SAHW2。
sudo apt install btrfs-progs
sudo mkfs.btrfs -L SAHW2 /dev/mapper/SA2025vg-SARaid10
```

## Step 2: 建立 SubVolume

```bash
# 建立一個臨時施工區
sudo mkdir -p /mnt/btrfs_top

# 將 BTRFS 預設頂層 (id=5) 掛載進來
sudo mount /dev/mapper/SA2025vg-SARaid10 /mnt/btrfs_top

sudo btrfs subvolume create /mnt/btrfs_top/root
sudo btrfs subvolume create /mnt/btrfs_top/sftp
sudo btrfs subvolume create /mnt/btrfs_top/pool1
sudo btrfs subvolume create /mnt/btrfs_top/pool2

sudo mkdir -p /mnt/btrfs_top/snapshot/sftp
sudo mkdir -p /mnt/btrfs_top/snapshot/pool1
sudo mkdir -p /mnt/btrfs_top/snapshot/pool2

sudo umount /mnt/btrfs_top
```

檢查
```bash
sudo btrfs subvolume list /mnt/hw2/ -p -a -t
```

## Step 3: Persistence

`sudo vim /etc/fstab`
```bash

/dev/mapper/SA2025vg-SARaid10   /mnt/hw2         btrfs   defaults,noatime,subvol=root   0 0
/dev/mapper/SA2025vg-SARaid10   /mnt/hw2/sftp    btrfs   defaults,noatime,subvol=sftp   0 0
/dev/mapper/SA2025vg-SARaid10   /mnt/hw2/pool1   btrfs   defaults,noatime,subvol=pool1  0 0
/dev/mapper/SA2025vg-SARaid10   /mnt/hw2/pool2   btrfs   defaults,noatime,subvol=pool2  0 0
```

下面這個是為了確保noatime都有生效 PPT好像沒特別提到但是judge說要這個設定
```bash
sudo mount -o remount,noatime /mnt/hw2/pool2
sudo mount -o remount,noatime /mnt/hw2/sftp
sudo mount -o remount,noatime /mnt/hw2/pool1
```

掛載資料夾
```bash
# 先建立最上層資料夾
sudo mkdir -p /mnt/hw2

# 掛載剛剛寫在 fstab 的第一行 (/mnt/hw2)
sudo mount /mnt/hw2

# 接著在裡面建立子目錄
sudo mkdir -p /mnt/hw2/sftp /mnt/hw2/pool1 /mnt/hw2/pool2

# 一口氣掛載 fstab 裡剩下的所有東西
sudo mount -a
```


檢查
```bash
df -hT | grep btrfs
```


## Step 4: 建立 Snapper script

```bash
sudo vim /usr/local/bin/SnApper
```

```bash
#!/bin/bash

# ==========================================
# BTRFS Snapshot Manager: SnApper
# ==========================================

# 1. 顯示 Help 訊息 (完全符合作業 5/11 格式)
usage() {
    echo "Usage:"
    echo "SnApper [-h]: show this message"
    echo "SnApper snapshot SUBVOL [-c ROTATION_COUNT]"
    echo "SnApper list [-p SUBVOL] [-i ID]"
    echo "SnApper delete [ID]"
    echo "SnApper rollback ID"
    exit 1
}

if [ "$1" == "-h" ] || [ -z "$1" ]; then
    usage
fi

# 2. 自動處理 BTRFS 頂層掛載 (ID=5)
# 利用臨時資料夾確保能夠無痛存取 Flatten 架構下的所有平行 Subvolume
BTRFS_DEV="/dev/mapper/SA2025vg-SARaid10"
TOP_MNT="/tmp/SnApper_btrfs_top_$$"

mkdir -p "$TOP_MNT"
mount -o subvolid=5 "$BTRFS_DEV" "$TOP_MNT"

# 確保腳本結束或中斷時，自動卸載並清理臨時資料夾
cleanup() {
    umount "$TOP_MNT" 2>/dev/null
    rmdir "$TOP_MNT" 2>/dev/null
}
trap cleanup EXIT

COMMAND=$1
shift

case "$COMMAND" in
    snapshot)
        # 參數解析: SnApper snapshot SUBVOL [-c ROTATION_COUNT]
        SUBVOL=$1
        if [ -z "$SUBVOL" ]; then usage; fi
        
        COUNT=5 # 預設保留 5 份
        if [ "$2" == "-c" ] && [ -n "$3" ]; then
            COUNT=$3
        fi

        # 確保目標快照目錄存在
        mkdir -p "$TOP_MNT/snapshot/$SUBVOL"

        # 建立時間戳記名稱
        TS=$(date +"%Y%m%d-%H%M%S")
        SNAP_NAME="@$TS"
        SNAP_PATH="snapshot/$SUBVOL/$SNAP_NAME"

        # 建立唯讀快照 (-r)
        btrfs subvolume snapshot -r "$TOP_MNT/$SUBVOL" "$TOP_MNT/$SNAP_PATH" > /dev/null 2>&1

        # 取得剛建立的快照 ID 並輸出結果
        NEW_ID=$(btrfs subvolume list "$TOP_MNT" | awk -v path="$SNAP_PATH" '$NF == path {print $2}')
        echo "Snap '$SNAP_PATH' [$NEW_ID]"

        # 執行 Rotation 邏輯 (超過數量則刪除最舊的)
        while true; do
            # 抓出該 SUBVOL 的所有快照，按 ID 排序 (ID 越小越舊)
            SNAPS=$(btrfs subvolume list "$TOP_MNT" | awk -v p="snapshot/$SUBVOL/@" '$NF ~ p {print $2, $NF}' | sort -k1,1n)
            SNAP_TOTAL=$(echo "$SNAPS" | grep -c "^[0-9]")
            
            if [ "$SNAP_TOTAL" -le "$COUNT" ]; then
                break
            fi
            
            # 刪除 ID 最小的快照
            OLDEST_PATH=$(echo "$SNAPS" | head -n 1 | awk '{print $2}')
            btrfs subvolume delete "$TOP_MNT/$OLDEST_PATH" > /dev/null 2>&1
        done
        ;;

    list)
        # 參數解析: SnApper list [-p SUBVOL] [-i ID]
        LIST_SUBVOL=""
        LIST_ID=""
        
        while [[ "$1" != "" ]]; do
            case $1 in
                -p ) shift; LIST_SUBVOL=$1 ;;
                -i ) shift; LIST_ID=$1 ;;
            esac
            shift
        done

        # 輸出 Header
        printf "%-7s %-15s %s\n" "ID" "SUBVOLUME" "TIME"

        # 解析並輸出符合條件的快照
        btrfs subvolume list "$TOP_MNT" | awk '/snapshot\// {print $2, $NF}' | while read id path; do
            # 利用 Regex 擷取 SUBVOL 名稱與時間
            if [[ "$path" =~ snapshot/([^/]+)/@([0-9]{4})([0-9]{2})([0-9]{2})-([0-9]{2})([0-9]{2})([0-9]{2}) ]]; then
                subvol="${BASH_REMATCH[1]}"
                year="${BASH_REMATCH[2]}"
                month="${BASH_REMATCH[3]}"
                day="${BASH_REMATCH[4]}"
                hour="${BASH_REMATCH[5]}"
                min="${BASH_REMATCH[6]}"
                sec="${BASH_REMATCH[7]}"
                
                formatted_time="${year}-${month}-${day} ${hour}:${min}:${sec}"
                
                # 篩選條件
                if [ -n "$LIST_SUBVOL" ] && [ "$LIST_SUBVOL" != "$subvol" ]; then continue; fi
                if [ -n "$LIST_ID" ] && [ "$LIST_ID" != "$id" ]; then continue; fi
                
                printf "%-7s %-15s %s\n" "$id" "$subvol" "$formatted_time"
            fi
        done
        ;;

    delete)
        # 參數解析: SnApper delete [ID]
        DEL_ID=$1

        if [ -n "$DEL_ID" ]; then
            # 刪除單一指定快照
            DEL_PATH=$(btrfs subvolume list "$TOP_MNT" | awk -v id="$DEL_ID" '$2 == id {print $NF}')
            if [ -n "$DEL_PATH" ]; then
                btrfs subvolume delete "$TOP_MNT/$DEL_PATH" > /dev/null 2>&1
                echo "Destroy ID $DEL_ID"
            fi
        else
            # 刪除所有受管理的快照
            btrfs subvolume list "$TOP_MNT" | awk '/snapshot\// {print $2, $NF}' | while read id path; do
                btrfs subvolume delete "$TOP_MNT/$path" > /dev/null 2>&1
                echo "Destroy ID $id"
            done
        fi
        ;;

    rollback)
        # 參數解析: SnApper rollback ID
        RB_ID=$1
        if [ -z "$RB_ID" ]; then usage; fi

        # 自動尋找對應的 Subvolume
        RB_PATH=$(btrfs subvolume list "$TOP_MNT" | awk -v id="$RB_ID" '$2 == id {print $NF}')
        if [ -z "$RB_PATH" ]; then
            echo "Error: Snapshot ID $RB_ID not found."
            exit 1
        fi

        if [[ "$RB_PATH" =~ snapshot/([^/]+)/ ]]; then
            TARGET_SUBVOL="${BASH_REMATCH[1]}"
        fi

        # BTRFS 完美還原神技：
        # 1. 刪除現在壞掉的 subvolume
        btrfs subvolume delete "$TOP_MNT/$TARGET_SUBVOL" > /dev/null 2>&1
        # 2. 從唯讀快照中「複製」一份具有讀寫權限 (無 -r) 的 subvolume 到原本的位置
        btrfs subvolume snapshot "$TOP_MNT/$RB_PATH" "$TOP_MNT/$TARGET_SUBVOL" > /dev/null 2>&1

        echo "Rollback '$RB_PATH' [$RB_ID] to $TARGET_SUBVOL"
        ;;

    *)
        usage
        ;;
esac
```

```bash
sudo chmod +x /usr/local/bin/SnApper
```