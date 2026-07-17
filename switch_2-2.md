# 第三堂：登入 SONiC 並認識交換機（Linux CLI 與光模組診斷）

## 課程目標

上一堂課，我們已經完成：

- 使用 Console 線連接交換機
- 設定 Terminal Emulator
- 完成交換機開機
- 成功登入 SONiC

登入完成後，你應該會看到類似畫面：

```text
admin@edgecore:~$
```

這代表你已經正式登入交換機。

從現在開始，你輸入的每一個指令，都會直接由交換機執行。

這堂課將介紹：

- SONiC 是什麼？
- Linux CLI 基本概念
- 如何查看交換機資訊
- 如何查看 Port 狀態
- 如何檢查光模組資訊
- 如何利用 DDM（Digital Diagnostic Monitoring）判斷光模組是否正常
- 基本除錯（Troubleshooting）技巧

完成本章後，你將能夠利用 CLI 快速掌握交換機與光模組的健康狀態。

---

# SONiC 是什麼？

SONiC 全名：

> **Software for Open Networking in the Cloud**

它是一套由 Microsoft 發起、目前由 Linux Foundation 維護的開源網路作業系統（Network Operating System，NOS）。

SONiC 建立於：

```
Debian Linux
        │
        ▼
SONiC
        │
        ▼
Switch ASIC
```

因此：

SONiC 本身就是 Linux。

也就是說：

交換機其實就是一台專門處理網路封包的 Linux 電腦。

因此：

許多 Linux 指令都可以直接使用。

例如：

```bash
ls

pwd

cd

cat

grep

top

df
```

同時，

SONiC 也提供大量專屬的：

```
show
config
interface
port
```

等網路管理指令。

---

# Linux Prompt（命令提示字元）

登入成功後，

通常會看到：

```text
admin@edgecore:~$
```

這一行包含許多資訊。

例如：

```
admin
```

表示：

目前登入帳號。

```
edgecore
```

表示：

交換機主機名稱（Hostname）。

```
~
```

表示：

目前位於：

Home Directory。

最後：

```
$
```

代表：

一般使用者。

若看到：

```
#
```

通常表示：

Root（最高權限）。

---

# 第一部分：確認交換機身分

工程師登入第一件事情通常不是修改設定，

而是：

先確認：

> 我現在登入的是哪一台交換機？

避免誤操作其他設備。

---

# show platform summary

輸入：

```bash
show platform summary
```

可能看到：

```text
Platform: x86_64-accton_as7726_32d-r0

HwSKU: AS7726-32D

ASIC: Broadcom Tomahawk

Ports: 32
```

---

## Platform

表示：

平台名稱。

通常包含：

- CPU 架構
- 廠商
- 型號

例如：

```
x86_64-accton_as7726_32d-r0
```

代表：

Edgecore AS7726-32D。

---

## HwSKU

Hardware SKU。

就是：

交換機型號。

例如：

```
AS7726-32D
```

代表：

32 個高速 Port。

---

## ASIC

交換機最重要的晶片。

例如：

```
Broadcom Tomahawk

Broadcom Trident

Marvell

Intel Tofino
```

ASIC 決定：

- Port 數量
- 封包轉送能力
- Buffer 大小
- 最大速度

---

## Ports

例如：

```
32
```

表示：

共有：

32 個高速網路 Port。

---

# show version

接著輸入：

```bash
show version
```

可能看到：

```text
SONiC Software Version

SONiC-202311

Kernel

5.10

Uptime

15 days
```

---

## Software Version

表示：

目前 SONiC 版本。

例如：

```
SONiC-202311
```

方便確認：

是否需要更新。

---

## Kernel

Linux Kernel 版本。

例如：

```
5.10
```

代表：

SONiC 使用 Linux Kernel 5.10。

---

## Uptime

表示：

交換機已經連續開機多久。

例如：

```
15 days
```

如果只有：

```
5 minutes
```

可能表示：

設備剛重新開機。

---

# 第二部分：查看所有 Port 狀態

真正開始管理交換機，

最常使用的一條指令就是：

```bash
show interfaces status
```

它可以一次列出：

所有 Port。

例如：

```text
Port        Status  Speed

Ethernet0   up      100G

Ethernet4   down    100G

Ethernet8   up      400G
```

---

# Port Name

例如：

```
Ethernet0

Ethernet4

Ethernet8
```

這就是：

SONiC 內部的 Port 名稱。

注意：

它不一定和面板上的：

```
Port 1

Port 2

Port 3
```

完全一致。

因此：

通常需要：

Port Mapping Table

才能對照。

---

# Status

最重要的一欄。

只有兩種：

```
up
```

表示：

- 光模組存在
- 光纖已連接
- 對端設備正常
- Link 建立成功

通常：

交換機 LED：

亮綠燈。

---

另一種：

```
down
```

表示：

可能：

- 沒插模組
- 沒插光纖
- 光纖斷掉
- 對端沒開機
- Speed 不一致

因此：

Port 尚未建立 Link。

---

# Speed

例如：

```
25G

100G

400G
```

代表：

目前協商成功的速度。

若看到：

```
N/A
```

通常：

表示：

尚未建立連線。

---

# VLAN

例如：

```
Vlan100

Vlan200
```

表示：

此 Port 屬於哪個 VLAN。

若：

```
routed
```

則表示：

Layer 3 Port。

---

# 第三部分：查看光模組資訊

光模組插入後，

交換機會透過：

```
I2C Bus
```

與光模組內部 EEPROM 通訊。

因此：

交換機可以讀取：

- 廠商
- 型號
- 序號
- 波長
- 距離

等等資訊。

---

# EEPROM 是什麼？

EEPROM：

全名：

```
Electrically Erasable
Programmable
Read Only Memory
```

它是一顆：

可重複寫入的小型記憶體。

每顆光模組都會內建 EEPROM。

裡面存放：

```
Vendor

Part Number

Serial Number

Distance

Connector

Wavelength
```

因此：

交換機一插上模組，

就知道：

這是哪一顆模組。

---

# show interfaces transceiver info

輸入：

```bash
show interfaces transceiver info
```

可能看到：

```text
Port

Ethernet0

Vendor

Edgecore

Serial

EC12345678

Connector

LC

Cable Length

10 km
```

---

## Vendor

表示：

製造商。

例如：

```
Edgecore

Intel

Finisar

Innolight

Broadcom
```

---

## Serial Number

每顆光模組都有：

唯一序號。

方便：

保固、

追蹤、

RMA。

---

## Connector

例如：

```
LC

MPO

MTP
```

代表：

光纖接頭種類。

---

## Cable Length

例如：

```
100m

2km

10km

40km
```

表示：

模組設計支援的最遠距離。

---

# 第四部分：DDM（Digital Diagnostic Monitoring）

除了基本資料，

更重要的是：

查看：

光模組健康狀態。

輸入：

```bash
show interfaces transceiver dom
```

某些版本：

```bash
show interfaces transceiver diagnostic
```

即可。

---

# DDM 是什麼？

DDM：

Digital Diagnostic Monitoring。

可以即時回報：

光模組內部感測器數據。

工程師每天都會看。

---

# Temperature（溫度）

例如：

```
45°C
```

一般：

30~70°C

都屬正常。

若：

```
85°C
```

表示：

散熱可能異常。

可能原因：

- 機房溫度過高
- 風扇故障
- 通風不良

---

# Voltage（電壓）

通常：

```
3.30V
```

若：

低於正常範圍，

可能：

供電異常。

---

# Bias Current（偏壓電流）

這是：

驅動 Laser 的電流。

例如：

```
6 mA

8 mA

10 mA
```

如果：

隨著時間越來越高，

代表：

Laser 老化。

需要更高電流才能維持相同亮度。

---

# TX Power（發射光功率）

表示：

本端射出的光。

單位：

```
dBm
```

例如：

```
-2 dBm
```

代表：

發光正常。

若：

沒有 TX Power，

可能：

Laser 已故障。

---

# RX Power（接收光功率）

最重要的一項。

表示：

收到：

對端射來的光。

例如：

```
-3 dBm
```

代表：

收光正常。

如果：

```
-40 dBm
```

甚至：

```
-inf
```

代表：

幾乎：

沒有收到任何光。

可能原因：

- 光纖未插好
- 光纖斷裂
- 對端設備未開機
- 對端光模組故障
- 光模組 TX 壞掉
- 防塵帽未取下

因此：

RX Power 是工程師最常查看的診斷數據。

---

# dBm 是什麼？

光功率通常使用：

```
dBm
```

表示。

它是一種：

對數（Logarithm）單位。

常見數值：

| 光功率 | 說明 |
|---------|------|
| 0 dBm | 約 1 mW，光很強 |
| -3 dBm | 常見正常發射功率 |
| -10 dBm | 仍屬正常範圍（依模組規格） |
| -20 dBm | 光已明顯變弱 |
| -30 dBm | 幾乎收不到光 |
| -40 dBm | 幾乎無訊號 |
| -inf | 完全沒有收到光 |

> **提醒：**
> 不同光模組（如 SR、LR、ER、ZR）的正常 TX/RX 功率範圍並不相同，實際判斷應參考該模組的 Datasheet 規格。

---

# Hot-Plug（熱插拔）

光模組支援：

Hot-Plug。

也就是：

交換機不用關機，

即可直接：

拔出、

插入。

但是：

正確步驟非常重要。

---

## 正確拔除方式

先：

拉開：

```
Latch

（塑膠拉環）
```

例如：

```
┌─────────┐
│  SFP    │
│         │
└──┐   ┌──┘
   ▼
 Plastic Latch
```

再：

輕輕拔出。

不要：

直接硬拉。

否則：

可能損壞：

交換機內部 Cage（模組插槽）的金屬彈片，造成永久性的硬體故障。

---

# 第三堂課常用指令整理

| 指令 | 功能 |
|------|------|
| `show platform summary` | 查看交換機平台、型號、ASIC 與 Port 數量 |
| `show version` | 查看 SONiC 版本、Linux Kernel、開機時間 |
| `show interfaces status` | 查看所有 Port 的連線狀態、速度與 VLAN |
| `show interfaces transceiver info` | 查看光模組 EEPROM 資訊（廠商、序號、接頭、距離等） |
| `show interfaces transceiver dom` | 查看 DDM 診斷資訊（溫度、電壓、偏壓電流、TX/RX 光功率） |

---

# 第三堂課重點整理

- SONiC 是基於 **Debian Linux** 的開源網路作業系統，因此可同時使用 Linux 指令與 SONiC 專屬管理指令。
- 登入後，工程師通常會先執行 `show platform summary` 與 `show version`，確認交換機型號、ASIC、SONiC 版本及系統運行時間。
- `show interfaces status` 是日常最常用的指令，可快速確認所有 Port 的 **Link Status**、速度（Speed）與 VLAN 配置；其中 `up` 表示連線成功，`down` 則需進一步排查原因。
- `show interfaces transceiver info` 可讀取光模組 EEPROM，取得製造商、型號、序號、接頭種類及支援距離等基本資訊。
- `show interfaces transceiver dom`（或 `diagnostic`）可檢視 DDM 即時診斷資料，包括 **Temperature、Voltage、Bias Current、TX Power、RX Power**，是光模組除錯最重要的工具。
- 在光纖故障排查時，**RX Power** 是最常檢查的指標；若數值接近 **-40 dBm** 或顯示 **-inf**，通常代表未收到有效光訊號，應優先檢查光纖、光模組與對端設備。
- 光模組支援 **Hot-Plug（熱插拔）**，但拔除時應先拉開塑膠拉環（Latch），避免損壞交換機內部的模組插槽（Cage）。
