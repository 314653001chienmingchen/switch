# 第五堂：Python 自動化管理交換機與光模組（SONiC + SSH + DDM Monitoring）

## 課程目標

前四堂課，我們已經學會：

- 認識交換機硬體
- 認識光模組與光纖
- 使用 Console 登入 SONiC
- 使用 CLI 查看交換機資訊
- 查看 Port 狀態
- 查看光模組 DDM（Digital Diagnostic Monitoring）
- 理解 MAC Address 與 VLAN

但是，如果今天實驗室有：

- 20 台交換機
- 每台 64 個 Port
- 每個 Port 都插了一顆 QSFP-DD 光模組

代表需要監控：

```
20 × 64 = 1280 個 Port
```

如果每五分鐘手動執行一次：

```bash
show interfaces transceiver dom
```

工程師一天可能要重複上百次相同工作，不僅耗時，也容易因人為疏忽漏掉異常。

因此，在 QA（Quality Assurance）、DV（Design Verification）、生產測試（Production Test）與資料中心維運（Operations）中，**自動化（Automation）** 已成為不可或缺的能力。

本章將介紹如何利用 **Python** 建立自動化工具，自動登入交換機、收集光模組健康資訊、分析異常並輸出報表。

完成本章後，你將能夠：

- 使用 Python 透過 SSH 遠端控制交換機
- 自動執行 SONiC CLI 指令
- 解析 CLI 回傳文字（Parsing）
- 篩選異常 DDM 數據
- 將資料存成 CSV
- 繪製監控圖表
- 建立簡單的光模組健康監控工具

---

# 為什麼需要自動化？

假設一位工程師負責：

```
32 台交換機
```

每台：

```
64 個 Port
```

總共有：

```
2048 個 Port
```

如果人工查看：

```
show interfaces transceiver dom
```

每個 Port 約需：

```
3 秒
```

全部檢查一次：

```
2048 × 3 秒

≈ 6144 秒

≈ 102 分鐘
```

也就是：

一個多小時。

如果利用 Python：

全部設備可在：

```
1~2 分鐘
```

內完成檢查。

這就是自動化最大的價值：

- 提高效率
- 降低人為錯誤
- 可長時間監控
- 可建立歷史資料
- 容易整合報表與告警系統

---

# Python 與交換機如何溝通？

交換機安裝 SONiC 後，本質上就是一台 Linux 主機。

除了 Console 外，更常見的遠端管理方式是：

> **SSH（Secure Shell）**

SSH 是一種加密的遠端登入協定，可安全地執行 Linux 指令。

架構如下：

```text
┌─────────────┐
│ Python      │
│ (Notebook)  │
└──────┬──────┘
       │ SSH
       ▼
┌─────────────┐
│ SONiC       │
│ Switch      │
└─────────────┘
```

Python 建立 SSH 連線後，就能像使用 Terminal 一樣，自動輸入指令並取得結果。

---

# Paramiko：Python 的 SSH 函式庫

Python 最常使用的 SSH 套件是：

```
paramiko
```

安裝方式：

```bash
pip install paramiko
```

它提供完整的 SSH Client 功能，包括：

- 建立 SSH 連線
- 執行遠端指令
- 上傳／下載檔案（SFTP）
- 金鑰認證
- 執行互動式 Shell

---

# 建立 SSH 連線

下面是一個最基本的連線範例：

```python
import paramiko

# 建立 SSH Client
ssh = paramiko.SSHClient()

# 自動接受主機金鑰（Lab 環境常見）
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())

# 建立連線
ssh.connect(
    hostname="192.168.1.100",
    username="admin",
    password="your_password"
)

# 執行指令
stdin, stdout, stderr = ssh.exec_command(
    "show interfaces transceiver dom"
)

# 讀取輸出
output = stdout.read().decode()

print(output)

ssh.close()
```

流程如下：

```text
Python
   │
   ▼
SSH Login
   │
   ▼
SONiC
   │
   ▼
show interfaces transceiver dom
   │
   ▼
回傳文字
   │
   ▼
Python 分析
```

> **建議：**
> 正式環境應優先使用 **SSH Key（金鑰登入）** 或 Secrets Manager 管理帳密，避免將密碼直接寫在程式碼中。

---

# Raw Text（原始文字）

CLI 回傳的內容通常都是：

```
String（字串）
```

例如：

```text
Port      Temp(C) Voltage(V) Tx(dBm) Rx(dBm)

Ethernet0 42.5    3.31       -2.10   -3.50

Ethernet4 45.2    3.30       -2.05   -40.00
```

Python 並不知道：

哪一欄是：

Temperature。

因此：

第一件工作就是：

> **Parsing（解析）。**

---

# Data Parsing（資料解析）

最簡單的方法就是：

```
split()
```

例如：

```python
line = "Ethernet4 45.2 3.30 -2.05 -40.00"

data = line.split()

print(data)
```

輸出：

```python
[
    "Ethernet4",
    "45.2",
    "3.30",
    "-2.05",
    "-40.00"
]
```

這時就可以取得：

```python
port = data[0]
temp = float(data[1])
voltage = float(data[2])
tx = float(data[3])
rx = float(data[4])
```

Python 已將原始文字轉換成可運算的數值。

---

# 篩選異常光模組

假設：

收到：

```python
raw_lines = [
    "Ethernet0 42.5 3.31 -2.10 -3.50",
    "Ethernet4 45.2 3.30 -2.05 -40.00",
    "Ethernet8 38.9 3.28 -2.12 -4.10"
]
```

我們可以：

逐行分析：

```python
for line in raw_lines:

    data = line.split()

    port = data[0]

    temp = float(data[1])

    rx = float(data[4])

    if rx < -15:

        print(f"{port} 光功率異常")

    if temp > 45:

        print(f"{port} 溫度過高")
```

輸出：

```text
Ethernet4 光功率異常

Ethernet4 溫度過高
```

這就是：

最基本的：

Monitoring Script。

---

# 常見 DDM 告警條件

實際專案中，工程師通常會依據光模組 Datasheet 設定告警門檻，例如：

| 監控項目 | 建議告警條件（示例） | 可能原因 |
|-----------|----------------------|----------|
| Temperature | > 70°C | 散熱不良、風扇異常 |
| Voltage | 超出模組規格 | 電源異常 |
| Bias Current | 持續偏高 | 雷射老化 |
| TX Power | 超出規格 | 發射器異常 |
| RX Power | < -15 dBm（依模組規格） | 光纖鬆脫、折損、對端異常 |

> **注意：**
> 各家模組（SR、LR、ER、ZR、DWDM 等）的正常範圍不同，實際門檻應以 Datasheet 為準。

---

# 更好的資料結構

如果：

Port 很多，

建議：

使用：

Dictionary。

例如：

```python
transceiver = {

    "port": "Ethernet4",

    "temperature": 45.2,

    "voltage": 3.30,

    "tx_power": -2.05,

    "rx_power": -40.0

}
```

之後：

即可：

```python
print(transceiver["rx_power"])
```

或：

建立：

List：

```python
ports = [

    {...},

    {...},

    {...}

]
```

方便：

後續分析。

---

# 使用 Pandas 建立表格

Python 最常用資料分析工具：

```
pandas
```

例如：

```python
import pandas as pd

df = pd.DataFrame(ports)

print(df)
```

即可得到：

| Port | Temp | Voltage | TX | RX |
|------|------|----------|----|----|
| Ethernet0 | 42.5 | 3.31 | -2.1 | -3.5 |
| Ethernet4 | 45.2 | 3.30 | -2.0 | -40 |
| Ethernet8 | 38.9 | 3.28 | -2.1 | -4.1 |

之後：

即可：

排序、

搜尋、

統計。

---

# CSV 報表

可直接：

輸出：

CSV：

```python
df.to_csv("ddm_report.csv", index=False)
```

產生：

```
ddm_report.csv
```

Excel

即可：

開啟。

方便：

QA

每日：

保存紀錄。

---

# Matplotlib 視覺化

如果：

每分鐘：

收集一次：

Temperature。

一天：

就有：

1440 筆。

利用：

Matplotlib：

```python
import matplotlib.pyplot as plt

plt.plot(time, temperature)

plt.xlabel("Time")

plt.ylabel("Temperature")

plt.show()
```

即可：

畫出：

```
Temperature

70│
60│
50│      ╭───╮
40│──────╯   ╰────
30│
  └───────────────
        Time
```

可觀察：

- 是否逐漸升溫
- 是否突然異常
- 是否散熱不足

---

# watch 指令

若：

暫時：

不想寫 Python。

Linux：

也提供：

```
watch
```

例如：

```bash
watch -n 2 "show interfaces transceiver dom"
```

代表：

```
每 2 秒
```

重新執行一次：

```
show interfaces transceiver dom
```

畫面會自動更新。

非常適合：

現場：

測試：

光纖。

---

# 字串解析注意事項

CLI 格式：

不同版本：

可能不同。

例如：

有時：

```
Port Temp Voltage
```

有時：

```
Port  Temp  Voltage
```

空格數量：

不同。

因此：

不要：

依賴：

固定位置。

建議：

- `split()`
- `strip()`
- `re`（Regular Expression）
- `TextFSM`
- `TTP`

等工具，提高程式容錯能力。

---

# 自動化流程總整理

完整流程如下：

```text
Python
    │
    ▼
SSH Login（Paramiko）
    │
    ▼
執行 CLI 指令
    │
    ▼
取得 Raw Text
    │
    ▼
Parsing
    │
    ▼
建立 Data Structure
    │
    ▼
異常判斷
    │
    ▼
CSV / Database
    │
    ▼
Matplotlib
    │
    ▼
報表 / 告警
```

---

# 第五堂課常用工具

| 工具 | 功能 |
|------|------|
| Paramiko | SSH 遠端登入交換機 |
| Pandas | 整理、分析 DDM 資料 |
| CSV | 儲存測試結果 |
| Matplotlib | 繪製溫度、光功率趨勢圖 |
| watch | Linux 即時刷新 CLI 指令 |
| Regular Expression（Regex） | 強化字串解析能力 |

---

# 第五堂課重點整理

- 在 QA、DV 與資料中心維運中，自動化可以大幅降低人工操作時間，並提升測試一致性與可靠性。
- Python 可透過 **SSH（Secure Shell）** 遠端登入 SONiC 交換機，自動執行 CLI 指令，其中 **Paramiko** 是最常用的 SSH 函式庫。
- `show interfaces transceiver dom` 回傳的是原始文字（Raw Text），需經過 **Data Parsing**（如 `split()`、Regex、TextFSM 等）才能轉換為可分析的資料結構。
- 可利用 Python 對 DDM 數據（Temperature、Voltage、Bias Current、TX Power、RX Power）設定門檻，自動篩選異常光模組並產生告警。
- **Pandas** 可用於整理大量 DDM 資料，並輸出為 **CSV** 報表；**Matplotlib** 可進一步繪製溫度、光功率等趨勢圖，協助分析長時間運作狀況。
- Linux 的 `watch` 指令可定期自動執行 CLI 命令，例如 `watch -n 2 "show interfaces transceiver dom"`，適合現場即時觀察光模組狀態。
- 撰寫自動化腳本時，應考慮不同 SONiC 版本可能造成 CLI 格式差異，避免依賴固定欄位位置，提升程式的容錯能力與可維護性。
