# 第二節：第一次登入交換機（Console 與 CLI 基礎）

## 課程目標

上一節課，我們已經認識了：

- 交換機（Switch）
- 光模組（Optical Transceiver）
- 光纖線（Fiber Optic Cable）

也完成了基本的硬體組裝。

但是，交換機不像一般電腦，它沒有：

- 螢幕
- 鍵盤
- 滑鼠

因此，當交換機第一次開機時，我們必須透過另一台電腦（通常是筆電）來控制它。

本課程將介紹如何建立交換機的第一條管理連線，並學會登入交換機的命令列介面（CLI）。

完成本章後，你將能夠：

- 認識 Console Port 的用途
- 使用 Console 線連接交換機
- 安裝並設定 Terminal Emulator
- 理解 Serial 連線參數
- 看懂交換機的開機流程
- 成功登入交換機 CLI

---

# 為什麼不用滑鼠？

大部分人每天使用 Windows 或 macOS，都習慣透過圖形介面（GUI，Graphical User Interface）操作，例如：

- 點擊圖示
- 開啟視窗
- 拖曳滑鼠
- 使用按鈕

例如：

```
┌─────────────────────┐
│  🖥 Windows Desktop │
│                     │
│   📁 📄 🌐 ⚙️       │
│                     │
└─────────────────────┘
```

但是，在企業級網路設備中，幾乎所有設定都透過：

> **CLI（Command Line Interface，命令列介面）**

完成。

CLI 就是一個只能輸入文字指令的操作環境，例如：

```text
show version

show interface

configure terminal
```

交換機會根據你輸入的命令執行對應工作。

原因包括：

- 操作速度快
- 可遠端管理
- 可批次自動化
- 功能完整
- 是所有網路設備共同標準

Cisco、Juniper、Arista、Nokia、SONiC、Linux 幾乎都採用 CLI。

因此，學會 CLI 是成為網路工程師的重要第一步。

---

# Console Port 是什麼？

交換機第一次開機時：

- 沒有 IP
- 沒有網路
- 沒有 SSH
- 沒有 Web 管理介面

因此根本無法透過網路登入。

這時只能利用：

> **Console Port（控制台埠）**

Console Port 可以理解成：

**交換機專用的鍵盤與螢幕接口。**

它提供最底層的管理通道，即使交換機完全沒有網路，也能進行設定。

例如：

```
┌────────────┐
│    筆電     │
└──────┬─────┘
       │
 Console Cable
       │
┌──────▼─────┐
│  Switch     │
│ Console Port│
└────────────┘
```

只要接上 Console 線，就可以看到交換機開機畫面，並輸入指令。

---

# Console 線（Console Cable）

Console 線就是筆電與交換機之間的「管理通道」。

它並不是拿來傳送網路封包，而是傳送文字訊息。

---

## 舊式 Console 線

早期交換機普遍採用：

```
RJ45
 │
 │
RS-232
```

其中：

RJ45 看起來與一般網路線接頭相同，

另一端則是：

RS-232（DB9）

```
 ___________
\  ● ● ● ● /
 \ ● ● ● ● /
  ‾‾‾‾‾‾‾‾‾
```

由於現代筆電幾乎都取消了 RS-232 接孔，因此通常需要：

```
USB
 │
USB to RS232
 │
RS232
 │
RJ45
 │
Switch
```

才能使用。

---

## 新式 Console 線

目前大部分 Open Networking 交換機採用：

### Type-C → RJ45

```
Type-C
    │
────┘
       RJ45
```

或：

### Type-C → Micro USB

```
Type-C
    │
────┘
   Micro USB
```

這些線材可直接連接現代筆電，不需額外轉接器。

---

# 如何正確插線？

交換機前面板通常有許多 RJ45 插孔。

例如：

```
□□□□□□□□□□□□□□□□
□□□□□□□□□□□□□□□□

      ○ Console
```

請注意：

**Console Port**

雖然外觀與 Ethernet Port（網路孔）幾乎相同，

但用途完全不同。

請務必確認標示：

```
Console
```

不要誤插到：

```
Eth1

Eth2

Eth3
```

否則將無法登入交換機。

---

# Terminal Emulator（終端機模擬器）

接上 Console 線後，

筆電還需要一套軟體來：

- 顯示交換機輸出的文字
- 傳送你輸入的命令

這類軟體稱為：

> **Terminal Emulator（終端機模擬器）**

它會模擬一台真正的終端機，透過 Serial（序列埠）與交換機通訊。

---

## Windows 常用軟體

### PuTTY

特色：

- 免費
- 小巧
- 安裝簡單
- 幾乎所有網路工程師都用過

---

### Tera Term

特色：

- 免費
- 支援巨集（Macro）
- 適合長時間除錯

---

## macOS / Linux

不需要額外安裝軟體。

直接使用內建：

```
Terminal
```

即可。

常見指令：

```bash
screen /dev/ttyUSB0 115200
```

或：

```bash
minicom
```

---

# Serial Connection（序列埠設定）

打開 PuTTY 或 Tera Term 後，

第一件事就是選擇：

```
Serial
```

接著需要設定 Serial Port 參數。

如果設定錯誤，

畫面可能：

- 完全沒反應
- 出現亂碼
- 無法輸入指令

因此每個參數都很重要。

---

## COM Port

Windows 會替 Console 線分配一個：

```
COM3

COM4

COM5
```

可在：

```
裝置管理員
(Device Manager)
```

查看。

例如：

```
Ports

 └─ USB Serial Port (COM4)
```

那麼 PuTTY 就要選：

```
COM4
```

---

## Baud Rate（鮑率）

Baud Rate 表示：

每秒傳輸多少位元。

就像兩個人講電話，

必須說同樣的語速，

否則就聽不懂。

目前 Open Networking 常見設定：

```
115200
```

部分舊設備：

```
9600
```

若設定錯誤，

可能看到：

```
▒▒▓█▒▓▒█▓▒▓▒▓
```

這就是典型亂碼。

---

## Data Bits

固定：

```
8
```

代表：

每個資料單位使用：

8 Bits。

---

## Parity（同位元檢查）

通常：

```
None
```

代表：

不使用額外錯誤檢查。

---

## Stop Bits

固定：

```
1
```

表示：

每個資料結束後加入一個停止位元。

---

## Flow Control

一般設定：

```
None
```

代表：

不使用硬體流量控制。

---

## 標準設定總整理

| 項目 | 設定值 |
|------|---------|
| Connection Type | Serial |
| COM Port | COM3、COM4…（依實際裝置） |
| Baud Rate | **115200** |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Flow Control | None |

---

# 第一次開機流程

完成 Serial 設定後：

按下：

```
Open
```

接著：

接上交換機電源。

黑色 Terminal 視窗就會開始顯示大量開機訊息。

Open Networking 交換機通常經過三個階段：

```
BIOS / GRUB
      │
      ▼
ONIE
      │
      ▼
NOS（SONiC、EcNOS）
```

---

# 第一階段：BIOS / GRUB

這個階段就像 PC 的 BIOS。

主要工作包括：

- CPU 初始化
- 記憶體檢查
- SSD 偵測
- 硬體初始化

完成後，

進入：

GRUB

開機選單。

例如：

```text
GNU GRUB

ONIE

SONiC

Rescue
```

可以選擇：

- 啟動作業系統
- 安裝系統
- Rescue Mode

---

# 第二階段：ONIE

如果交換機尚未安裝 NOS，

便會自動進入：

```
ONIE

(Open Network Install Environment)
```

---

## ONIE 是什麼？

ONIE 可以想像成：

> **交換機版的安裝程式。**

它本身是一個極簡 Linux。

唯一工作就是：

- 找 NOS 安裝檔
- 安裝作業系統
- 救援系統

例如：

```
ONIE

↓

DHCP

↓

HTTP

↓

Download Image

↓

Install SONiC
```

因此：

ONIE 並不是正式工作的系統。

---

## ONIE Discovering...

最常看到：

```text
ONIE: Discovering...
```

意思就是：

ONIE 正在：

- 嘗試取得 DHCP
- 尋找 HTTP Server
- 尋找安裝映像檔

如果一直看到：

```text
Discovering...
```

通常表示：

交換機目前還沒有安裝 NOS。

---

# 第三階段：NOS（Network Operating System）

如果交換機已安裝完成，

便會進入：

正式的網路作業系統（NOS）。

例如：

- SONiC
- EcNOS
- Cumulus Linux
- Cisco NX-OS
- Arista EOS

完成開機後，

畫面通常停留在：

```text
localhost login:
```

此時，

就可以輸入：

```text
Username:

admin
```

再輸入：

```text
Password:

******
```

成功登入後，

會看到命令提示字元，例如：

```text
admin@switch:~$
```

或：

```text
sonic#
```

代表已成功進入交換機 CLI。

---

# 第二堂課常見問題

## 畫面完全沒有反應

請依序檢查：

- Console 線是否插緊
- 是否插到真正的 Console Port
- COM Port 是否選對
- Baud Rate 是否為 115200
- USB Driver 是否正確安裝

---

## 畫面出現亂碼

通常代表：

Baud Rate 設定錯誤。

請確認：

```
115200
```

是否與交換機一致。

---

## 一直出現 ONIE: Discovering...

這代表：

交換機目前仍停留在 ONIE。

可按：

```
Enter
```

或輸入：

```bash
onie-shell
```

即可中斷自動搜尋，

進入 ONIE Shell。

例如：

```bash
ONIE:/ #
```

之後便可手動執行安裝、檢查網路或進行系統維護。

---

# 第二堂課重點整理

- 新交換機第一次開機時沒有 IP，因此無法透過 SSH 或 Web 管理，只能利用 **Console Port** 建立第一條管理連線。
- Console 線負責傳送管理指令，而不是網路資料。常見線材包括 **RJ45 ↔ RS-232**（舊式）以及 **Type-C ↔ RJ45**、**Type-C ↔ Micro-USB**（新式）。
- 筆電需要透過 **Terminal Emulator**（如 PuTTY、Tera Term、macOS Terminal）與交換機進行序列埠（Serial）通訊。
- Open Networking 交換機最常使用的 Serial 設定為：**115200、8、N、1、Flow Control = None**。
- 交換機開機流程依序為：**BIOS / GRUB → ONIE → NOS（如 SONiC、EcNOS）**。
- **ONIE（Open Network Install Environment）** 是交換機的安裝與救援環境，不是正式的網路作業系統；若畫面持續顯示 `ONIE: Discovering...`，表示設備正在尋找可安裝的 NOS。
- 成功開機後，系統會顯示登入提示（如 `localhost login:`），輸入帳號與密碼後即可進入交換機的 CLI，開始後續設定與管理工作。
