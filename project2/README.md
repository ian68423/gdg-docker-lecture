# Project 2：Package Manager 對照 - apt (Debian) vs apk (Alpine)

> 從 OS base image 開始，學會用兩種主流套件管理工具裝東西。
> 用 PHP 當案例：兩邊裝同樣的 PHP CLI + extensions，跑同一個 script。

## 這個專案在做什麼

兩個 Dockerfile，**做同樣的事**：

- 從 `debian:bookworm-slim` 或 `alpine:3.20` 開始（純 OS，沒有 PHP）
- 裝 PHP CLI + curl + mbstring + openssl extension
- 跑同一個 `script.php`：打 GitHub API、測試 mbstring 處理中文

差別只在**用哪個 package manager**——`apt` vs `apk`。

## 目錄結構

```
project2/
├── README.md
├── script.php           # 共用的 PHP demo（兩個 Dockerfile 都 COPY 它）
├── Dockerfile.alpine    # Alpine + apk
└── Dockerfile.debian    # Debian + apt
```

## 跑起來

```bash
cd project2

# Build 兩個 image
docker build -f alpine/Dockerfile.alpine -t demo-alpine .
docker build -f debian/Dockerfile.debian -t demo-debian .

# 跑跑看（兩邊輸出應該一模一樣）
docker run --rm demo-alpine
docker run --rm demo-debian

# 比較 image 大小
docker images | grep demo-
```

注意 `-f` flag：當 Dockerfile **不叫做 `Dockerfile`** 時，用 `-f` 指定要用哪個。最後那個 `.` 是 **build context**（傳給 Docker 的資料夾路徑）。

---

# apt vs apk 完整對照

## 核心指令對照

| 操作 | Debian (apt) | Alpine (apk) |
|---|---|---|
| 更新索引 | `apt-get update` | `apk update`（多數情況可省）|
| 裝套件 | `apt-get install -y <pkg>` | `apk add <pkg>` |
| 不互動 | 加 `DEBIAN_FRONTEND=noninteractive` | 預設就不互動 |
| 不裝建議套件 | `--no-install-recommends` | 預設不裝 |
| 不留快取 | `&& rm -rf /var/lib/apt/lists/*` | `--no-cache` flag |
| 移除套件 | `apt-get remove <pkg>` | `apk del <pkg>` |
| 搜尋套件 | `apt-cache search <keyword>` | `apk search <keyword>` |
| 看裝了什麼 | `dpkg -l` | `apk info` |

## 套件命名慣例不同

這是學生最容易混亂的地方：

| 你想要的 | Debian | Alpine |
|---|---|---|
| PHP 命令列 | `php-cli`（裝的是 Debian 維護的版本）| `php83-cli`（**版本綁在名字**）|
| PHP curl 擴充 | `php-curl` | `php83-curl` |
| Python 3 | `python3` | `python3` |
| Python pip | `python3-pip` | `py3-pip` |
| Node.js | `nodejs` | `nodejs` |
| Build 工具集 | `build-essential` | `build-base` |
| C compiler | `gcc` | `gcc`（但需要再裝 `musl-dev`）|

**Alpine 的版本綁名字** 是雙面刃：
- ✅ 想換 PHP 版本：改 `php83` → `php82` 就好
- ❌ 升級主要版本要全部改一次
- ❌ Dockerfile 寫死版本，跨年不太能無腦更新

**Debian** 反過來：
- ✅ 寫 `php-cli` 不管哪一版都對
- ❌ 你拿到的版本是該 Debian 版本維護的——`bookworm` 是 PHP 8.2、`bullseye` 是 PHP 7.4。**換 Debian 版本可能就連帶換 PHP 版本**。

## 容易踩的坑（課堂講重點）

### 1. Layer cache 陷阱 ⚠️ 最常踩

```dockerfile
# ❌ 錯誤示範
RUN apt-get update
RUN apt-get install -y curl
```

為什麼錯？Docker 會把 `RUN apt-get update` cache 起來。下次你改第二個 `RUN` 加個套件：

```dockerfile
RUN apt-get update                          # cache hit！用的是「舊的」 index
RUN apt-get install -y curl wget vim       # 用舊 index 找新套件 → 404 或裝到舊版
```

**正解**：永遠把 `update` 跟 `install` 綁在同一個 `RUN`：

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl wget vim \
    && rm -rf /var/lib/apt/lists/*
```

### 2. Alpine 沒寫 `--no-cache` → image 多 5-10 MB

```dockerfile
# ❌ 留下 apk index 在 /var/cache/apk/
RUN apk add curl

# ✅ 裝完不留快取
RUN apk add --no-cache curl
```

對應 Debian 那邊是更冗長的 `rm -rf /var/lib/apt/lists/*`。

### 3. Debian 沒設 `DEBIAN_FRONTEND=noninteractive`

某些套件（特別是 `tzdata`、`keyboard-configuration`）會問問題：

```
Configuring tzdata
------------------
Please select the geographic area in which you live.
  1. Africa  2. America  3. Antarctica  ...
```

CI 上沒人回答，build 永遠卡住。

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive    # 加這行，預設選 default
RUN apt-get install -y tzdata          # 不會問問題了
```

### 4. Alpine 找不到 C library → musl 不是 glibc

Alpine 用 **musl libc**，不是 Linux 業界主流的 **glibc**。多數套件沒事，但：

- Python 帶 C extension 的（numpy, pandas, lxml, asyncpg...）**沒有 prebuilt wheel for musl**，要從 source build → 慢 + 容易失敗
- 一些閉源 binary（Nvidia driver、某些商業 SDK）**只支援 glibc**
- libc 函式行為偶有微差（DNS 解析、locale）

**所以 Project 1 的 backend 用 `python:3.14-slim`（Debian-based）不是隨便選的**。

### 5. PHP 在 Alpine 要記得 symlink

Alpine 裝完 PHP 8.3 是 `/usr/bin/php83`，**沒有 `/usr/bin/php`**。你的 script 要嘛寫 `php83 script.php`，要嘛在 Dockerfile 裡 symlink：

```dockerfile
RUN ln -s /usr/bin/php83 /usr/bin/php
```

Debian 的 `php-cli` 預設就是 `/usr/bin/php`，沒這問題。

## Image 大小比較（預期值）

```bash
$ docker images | grep demo-
demo-alpine    latest    ~80 MB
demo-debian    latest    ~190 MB
```

差距主要來自：
- Alpine base 30MB vs Debian slim 80MB
- Alpine 的 PHP 套件比 Debian 緊湊
- Alpine 沒裝多餘的工具（連 `bash` 都沒有，預設是 `ash`）

**但小 ≠ 永遠選 Alpine**，看狀況：

| 場景 | 建議 |
|---|---|
| 純 PHP / Go / 靜態 binary | Alpine |
| Node.js（一般用途）| Alpine |
| **Python 帶 C extension** | **Debian-slim** |
| 需要特定商業軟體 / GPU | Debian |
| Debug 容易、團隊熟悉 | Debian |

## 為什麼這節要學

學生畢業後寫的每一個 Dockerfile，**第一行 `FROM` 之後幾乎都會接著裝套件**。能熟練用 apt / apk 是 Docker 基本功，比 multi-stage、cache mount 都基本。

而且 Dockerfile 裡的套件管理錯誤是「image 默默變大」「build 慢慢變慢」的元凶，這節學一次受用一輩子。

## 延伸練習（可選）

- 試試 `apt-get install vim`（沒加 `--no-install-recommends`），看 image 大多少
- 把 Alpine 換成 `php82`，跑跑看會怎樣
- 試著裝一個 Debian 有但 Alpine 沒有的套件（提示：商業軟體、特定 GPU 工具）
