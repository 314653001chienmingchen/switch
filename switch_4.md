# 第四堂：交換機的核心原理——MAC Address 與 VLAN

## 課程目標

前三堂課，我們已經學會：

- 認識交換機硬體
- 使用 Console 登入交換機
- 使用 SONiC CLI 查看設備資訊
- 查看光模組與 Port 狀態

但是，到目前為止，我們還沒有真正了解：

> **交換機到底是如何轉送網路封包的？**

很多初學者認為：

> 「交換機就是把資料送出去。」

事實上，交換機並不是隨便把封包送到所有 Port，而是透過一套非常精密的學習機制，自動建立一份網路拓樸資料庫，並依照規則決定封包應該送到哪裡。

這堂課將介紹交換機最重要的兩個核心功能：

- **MAC Address Learning（MAC 位址學習）**
- **VLAN（Virtual LAN，虛擬區域網路）**

這兩個功能幾乎存在於所有企業交換機中，也是所有網路工程師最基本的知識。

---

# 網路七層模型（OSI Model）

在了解交換機之前，必須先知道交換機位於哪一層。

OSI（Open Systems Interconnection）模型將網路通訊分成七層：

| Layer | 名稱 | 常見設備 |
|--------|------|----------|
| Layer 7 | Application（應用層） | Web Server、Email |
| Layer 6 | Presentation（表示層） | SSL、TLS |
| Layer 5 | Session（會議層） | Session 管理 |
| Layer 4 | Transport（傳輸層） | TCP、UDP |
| Layer 3 | Network（網路層） | Router（路由器） |
| **Layer 2** | **Data Link（資料鏈結層）** | **Switch（交換機）** |
| Layer 1 | Physical（實體層） | 光纖、RJ45、光模組 |

交換機主要工作於：

> **Layer 2（L2）**

因此：

交換機主要處理的是：

**Ethernet Frame（乙太網路框架）**

而不是：

IP Packet。

---

# Layer 2 與 Layer 3 的差別

很多人容易混淆：

MAC Address

與

IP Address。

其實兩者負責完全不同的工作。

| 比較項目 | MAC Address | IP Address |
|-----------|-------------|------------|
| 所屬層級 | Layer 2 | Layer 3 |
| 用途 | 區域網路識別 | 跨網路路由 |
| 是否固定 | 通常固定於網路卡 | 可由 DHCP 或管理者設定 |
| 長度 | 48 bits | IPv4：32 bits / IPv6：128 bits |
| 範例 | `00:1A:2B:3C:4D:5E` | `192.168.1.10` |

交換機主要認識的是：

> **MAC Address**

只有路由器才會根據：

IP Address

決定資料該往哪個網段送。

---

# MAC Address 是什麼？

MAC（Media Access Control）Address 是：

每張網路卡（NIC）出廠時就寫入的唯一硬體位址。

它就像：

> **網路卡的身分證字號。**

例如：

```text
00:0A:95:9D:68:16
```

共：

48 bits

通常寫成：

```
AA:BB:CC:DD:EE:FF
```

前 24 Bits：

代表：

製造商（OUI，Organizationally Unique Identifier）。

例如：

```
00:1B:21
```

可能代表：

某家網路設備廠商。

後 24 Bits：

代表：

產品序號。

因此：

世界上理論上不會有兩張網路卡擁有完全相同的 MAC Address。

---

# MAC Address Table（MAC 位址表）

交換機最大的特色就是：

> **具有自我學習能力。**

交換機不需要人工設定：

哪一台電腦接在哪個 Port。

它會自己學。

交換機內部維護一張資料表：

```
MAC Address Table
```

又稱：

- CAM Table
- Forwarding Database（FDB）

裡面記錄：

```
MAC Address
        │
        ▼
Port
```

例如：

| MAC Address | Port |
|-------------|------|
| 00:0A | Ethernet1 |
| 00:0B | Ethernet2 |
| 00:0C | Ethernet3 |
| 00:0D | Ethernet4 |

這就是交換機的「聯絡簿」。

---

# MAC Learning（MAC 位址學習）

交換機如何建立這本聯絡簿？

答案是：

> **偷聽（Learning）。**

每當收到封包時，

交換機第一件事情就是：

查看：

```
Source MAC
```

也就是：

寄件者。

假設：

```
PC A

MAC：

00:0A
```

接在：

```
Ethernet1
```

當 PC A 發送第一個封包時：

```
PC A
   │
   ▼
Switch
```

交換機立刻記錄：

```
00:0A

↓

Ethernet1
```

完成第一次學習。

因此：

每一個封包，

交換機都會持續更新：

MAC Table。

---

# MAC Forwarding（查表轉送）

現在：

PC A 想傳送資料給：

PC D。

封包內容：

```
Source MAC：

00:0A

Destination MAC：

00:0D
```

交換機收到後，

開始查詢：

```
MAC Table
```

如果：

已經找到：

```
00:0D

↓

Ethernet4
```

那麼：

交換機只會：

把封包送到：

```
Ethernet4
```

其他 Port：

完全收不到。

這種方式稱為：

> **Unicast（單播）**

也是交換機最有效率的轉送方式。

---

# Unknown Unicast Flooding（未知單播洪泛）

如果：

MAC Table 中沒有：

```
00:0D
```

怎麼辦？

交換機不知道：

PC D 在哪裡。

因此：

它只能：

把封包：

複製出去。

例如：

```
Ethernet1

↓

Switch

↓

Ethernet2

Ethernet3

Ethernet4

Ethernet5
```

除了：

來源 Port：

不送。

其他：

全部送。

這就是：

> **Flooding（洪泛）**

當：

PC D 回覆封包時，

交換機就會學到：

```
00:0D

↓

Ethernet4
```

之後，

就不用 Flood。

---

# MAC Address Aging（老化機制）

如果：

某台電腦拔掉了。

交換機不能：

永遠保留：

MAC Table。

因此：

交換機有：

Aging Timer（老化時間）。

例如：

```
300 秒
```

若：

300 秒都沒有收到：

某個 MAC 的封包，

便自動刪除。

如此：

MAC Table 才能保持最新狀態。

---

# 查看 MAC Table

SONiC：

輸入：

```bash
show mac
```

可能看到：

```text
MAC Address        VLAN    Port        Type

00:0A:95:9D:68:16  10      Ethernet0  Dynamic

00:0B:12:45:11:22  20      Ethernet4  Dynamic
```

---

## Dynamic

表示：

交換機：

自己學到。

最常見。

---

## Static

表示：

管理者：

手動設定。

通常：

特殊設備、

安全需求

才會使用。

---

# VLAN（Virtual LAN）

如果：

一台交換機有：

48 個 Port。

所有電腦：

都接在一起。

那麼：

任何：

Broadcast

都會：

送給：

48 個 Port。

例如：

ARP Request：

```
誰是：

192.168.1.10？
```

全部：

都會收到。

當設備很多時，

會造成：

- 網路塞車
- 頻寬浪費
- 安全風險

因此：

需要：

VLAN。

---

# VLAN 是什麼？

VLAN：

Virtual LAN。

中文：

> **虛擬區域網路。**

它可以：

把一台交換機，

切成：

很多個：

獨立的小交換機。

例如：

```
Switch
│
├── VLAN10
│
├── VLAN20
│
└── VLAN30
```

彼此：

完全隔離。

---

# VLAN ID

每個 VLAN 都有：

一個編號：

```
1

10

20

100

4094
```

有效範圍：

```
1~4094
```

例如：

| VLAN ID | 部門 |
|----------|------|
| 10 | 財務部 |
| 20 | 研發部 |
| 30 | 人資部 |
| 100 | 實驗室 |

每個部門：

互相：

看不到。

---

# Broadcast Domain（廣播網域）

VLAN 最大作用：

就是：

切割：

Broadcast Domain。

例如：

VLAN10：

```
PC1

PC2

PC3
```

PC1 發送：

ARP Broadcast。

只有：

PC2

PC3

收到。

VLAN20：

完全：

不知道。

因此：

大幅降低：

Broadcast。

---

# 查看 VLAN

SONiC：

查看設定：

```bash
show vlan config
```

查看摘要：

```bash
show vlan brief
```

可能看到：

```text
VLAN

10

Members

Ethernet0

Ethernet4
```

表示：

兩個 Port：

都屬於：

VLAN10。

---

# Access Port

Access Port：

最常見。

通常：

接：

- PC
- Server
- Printer

一個 Port：

只能：

屬於：

一個 VLAN。

例如：

```
Ethernet0

↓

VLAN10
```

封包：

不帶：

VLAN Tag。

因此：

終端設備：

不知道 VLAN 存在。

---

# Trunk Port

如果：

兩台交換機：

互連。

例如：

```
Switch A

──────────

Switch B
```

需要：

同時傳送：

VLAN10

VLAN20

VLAN30

怎麼辦？

不能：

拉三條線。

因此：

利用：

Trunk。

---

# VLAN Tag（802.1Q）

Trunk：

會在 Ethernet Frame 中加入：

```
802.1Q VLAN Tag
```

例如：

```
Frame

↓

VLAN ID = 10
```

另一台交換機收到：

知道：

這是：

VLAN10。

若：

```
VLAN ID = 20
```

便送到：

VLAN20。

因此：

一條 Trunk：

可以：

攜帶：

許多 VLAN。

---

# Access 與 Trunk 比較

| 項目 | Access Port | Trunk Port |
|------|-------------|------------|
| 常見用途 | PC、Server、印表機 | Switch、Router、Firewall |
| VLAN 數量 | 1 個 | 多個 |
| 是否帶 VLAN Tag | 否 | 是（IEEE 802.1Q） |
| 終端設備是否感知 VLAN | 否 | 是（網路設備需辨識 Tag） |

---

# 第四堂課常用指令整理

| 指令 | 功能 |
|------|------|
| `show mac` | 查看 MAC Address Table（MAC 位址表） |
| `show vlan config` | 查看 VLAN 詳細設定 |
| `show vlan brief` | 查看 VLAN 摘要與 Port 成員 |

---

# 第四堂課重點整理

- 交換機屬於 **OSI Layer 2（Data Link Layer）** 設備，主要依據 **MAC Address** 而非 IP Address 進行封包轉送。
- 每張網路卡都有唯一的 **MAC Address**，交換機透過觀察封包來源位址（Source MAC）自動建立 **MAC Address Table（CAM/FDB）**，無需人工設定設備位置。
- 當交換機知道目的 MAC 所在 Port 時，會採用 **Unicast（單播）** 將封包只送往該 Port；若尚未學到目的 MAC，則會進行 **Flooding（洪泛）**，等待目標設備回覆後完成學習。
- `show mac` 可查看交換機目前學到的 MAC 位址，其中 **Dynamic** 表示自動學習，**Static** 表示由管理者手動設定。
- **VLAN（Virtual LAN）** 可將一台實體交換機切割成多個彼此隔離的虛擬網路，降低 Broadcast 流量並提升網路安全性。
- 每個 VLAN 擁有唯一的 **VLAN ID（1～4094）**，不同 VLAN 之間預設無法直接通訊，若需互通，必須透過 **Layer 3 路由器或 Layer 3 Switch**。
- **Access Port** 通常連接終端設備，只屬於一個 VLAN；**Trunk Port** 則連接交換機、路由器等網路設備，可同時傳送多個 VLAN 的流量，並利用 **IEEE 802.1Q VLAN Tag** 標記封包所屬的 VLAN。
