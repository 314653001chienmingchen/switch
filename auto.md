**模組載入與環境設定**

```python
import os
# 載入 os 模組：用來讀取作業系統的環境變數，避免將帳號密碼寫死在程式碼中

import sys
# 載入 sys 模組：提供與 Python 解譯器互動的功能，這裡主要用來在連線失敗時強制終止程式

import paramiko
# 載入 paramiko 模組：Python 處理 SSHv2 協定的核心套件，用來控制遠端交換機
```

**全局配置區**
```python
# ==========================================
# 1. 全局配置區 (Configuration)
#    優先從環境變數讀取帳密，避免敏感資訊直接硬編碼在程式碼中
# ==========================================
DEVICE_CONFIG = {
    # 設定目標交換機的 IP：優先讀取環境變數 SWITCH_IP，若沒有設定則預設使用 "192.168.1.100"
    "ip": os.getenv("SWITCH_IP", "192.168.1.100"),

    # 設定 SSH 埠號：讀取環境變數 SWITCH_PORT 並轉成整數，預設為 22
    "port": int(os.getenv("SWITCH_PORT", 22)),

    # 設定登入帳號：讀取環境變數 SWITCH_USER，預設為 "admin"
    "username": os.getenv("SWITCH_USER", "admin"),

    # 設定登入密碼：讀取環境變數 SWITCH_PASS，預設為 "your_password"
    "password": os.getenv("SWITCH_PASS", "your_password"),

    # 設定連線逾時時間（秒）：若超過 10 秒未回應則判定連線失敗
    "timeout": 10,
}
```

**核心底層類別**

```python
# ==========================================
# 2. 核心底層類別 (Core Engine)
# ==========================================
class SwitchRunner:
    # 類別建構子：當建立 SwitchRunner 物件時自動呼叫
    def __init__(self, config):
        self.config = config   # 將傳入的設定字典（DEVICE_CONFIG）儲存為物件內部屬性
        self.client = None     # 初始化 SSH 客戶端屬性，預設為 None（代表尚未連線）

    def connect(self):
        """建立 SSH 連線並載入主機驗證"""
        try:
            self.client = paramiko.SSHClient()
            # 建立一個新的 Paramiko SSHClient 實體

            # [安全性修復] 嘗試讀取本地 known_hosts 避免中間人攻擊
            # 如果是第一次連線且無 known_hosts，建議配合 AutoAddPolicy，但加上警告提示
            self.client.load_system_host_keys()
            # 載入作業系統現有的 SSH 已知主機金鑰（known_hosts）

            self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            # 設定當遇到未記錄的主機金鑰時自動信任並新增（適合自動化腳本，但需注意網路環境安全）

            self.client.connect(
                hostname=self.config["ip"],        # 指定連線目標 IP
                port=self.config["port"],          # 指定連線埠號 (22)
                username=self.config["username"],  # 指定登入帳號
                password=self.config["password"],  # 指定登入密碼
                timeout=self.config["timeout"],    # 指定 TCP 連線超時時間 (10秒)
                banner_timeout=15,  # 避免某些設備 Banner 載入過慢（設定等待設備 Banner 回應的超時時間為 15 秒）
            )
            print(f"[SUCCESS] 已成功連線至交換機: {self.config['ip']}")
            # 連線成功時，在終端機印出成功訊息

            # [效能與穩定性修復] 關閉 CLI 輸出分頁，避免 More 提示導致卡死
            self.run_command("terminal length 0")
            # 發送指令告訴交換機「輸出文字時不要分頁」，防止因為跳出 "--More--" 而讓程式卡住

            return True
            # 連線與初始化成功，回傳 True

        except paramiko.AuthenticationException:
            # 捕捉帳號或密碼錯誤的異常
            print("[ERROR] 帳號或密碼錯誤，連線失敗！")
            return False

        except paramiko.SSHException as e:
            # 捕捉 SSH 協定層級的錯誤（例如金鑰不匹配、演算法不支援）
            print(f"[ERROR] SSH 協定異常: {e}")
            return False

        except Exception as e:
            # 捕捉其他所有未預期的錯誤（例如 IP 無法連通、網路斷線）
            print(f"[ERROR] 無法連線至 {self.config['ip']}: {e}")
            return False

    def run_command(self, command, timeout=10):
        """傳送單一指令並回傳文字結果（含通道管理與超時保護）"""
        if not self.client:
            # 防呆檢查：如果 SSH 客戶端尚未建立，則拒絕執行指令
            print("[ERROR] 尚未建立連線，無法執行指令！")
            return None

        try:
            # 向交換機發送指令
            stdin, stdout, stderr = self.client.exec_command(
                command, timeout=timeout
            )
            # 在遠端執行指令，並取得標準輸入(stdin)、標準輸出(stdout)與標準錯誤(stderr)三條通道

            # 設定讀取超時時間
            stdout.channel.settimeout(timeout)
            # 為 stdout 通道設定讀取時限，避免遠端設備無回應時程式無限期等待

            # 讀取結果並解碼（使用 errors='ignore' 避免特殊非 UTF-8 字元卡死）
            output = stdout.read().decode("utf-8", errors="ignore")
            # 讀取標準輸出的位元組資料，以 UTF-8 解碼成字串，若遇非 UTF-8 字元則直接忽略不報錯

            error_msg = stderr.read().decode("utf-8", errors="ignore").strip()
            # 讀取標準錯誤的資料，解碼並去除前後空白字元

            if error_msg:
                # 若錯誤通道有內容，印出警告訊息
                print(f"[WARN] 指令 '{command}' 跳出警告/錯誤: {error_msg}")

            # [資源釋放] 顯式關閉 Channel 避免交換機 Session 殘留
            stdout.channel.close()
            # 顯式關閉此指令開啟的 SSH Channel，釋放遠端交換機的系統資源

            return output
            # 回傳執行結果字串

        except Exception as e:
            # 捕捉指令執行過程中的超時或中斷異常
            print(f"[ERROR] 指令 '{command}' 執行失敗或超時: {e}")
            return None

    def close(self):
        """安全關閉連線"""
        if self.client:
            # 檢查 SSH 客戶端是否存在
            self.client.close()
            # 關閉 SSH 連線並釋放 Socket 資源

            print(f"[INFO] 與 {self.config['ip']} 的連線已安全釋放。")
            # 印出連線釋放完成的訊息
```

**業務邏輯區**

```python
# ==========================================
# 3. 業務邏輯區 (Task Handler)
# ==========================================
def task_check_ddm(runner):
    """任務 A：檢查光模組 DDM（含完善的欄位解析與例外防呆）"""
    print("\n---> 開始執行任務：光模組 DDM 健檢")
    # 印出任務開始訊息

    raw_data = runner.run_command("show interfaces transceiver dom")
    # 呼叫 runner 物件執行查詢光模組 DDM 狀態的指令，並接收文字結果

    if not raw_data:
        # 防呆：如果完全沒拿到資料（None 或空字串），印出警告並結束函式
        print("[WARN] 無法取得 DDM 數據")
        return

    lines = raw_data.splitlines()
    # 將回傳的大字串依據換行符號切割成字串串列（List of lines）

    for line in lines:
        # 逐行檢查輸出內容
        if "Ethernet" in line:
            # 只處理包含 "Ethernet" 字樣的行（過濾掉表頭與無關資訊）

            parts = line.split()
            # 將該行依據空白字元切割成多個欄位串列

            # [防呆修復] 確保欄位數量足夠才進行讀取，避免 IndexError
            if len(parts) < 5:
                # 若欄位數少於 5 個，代表資料格式不完整，直接跳過該行
                continue

            port = parts[0]
            # 第 0 個欄位通常是埠口名稱（例如：Ethernet1/1）

            raw_rx_power = parts[4]
            # 第 4 個欄位通常是接收光功率（Rx Power）的字串

            # [防呆修復] 安全將字串轉為 float，處理 N/A、-- 或 Off 的情況
            try:
                rx_power = float(raw_rx_power)
                # 嘗試將光功率字串轉換為浮點數（float）以利比對大小

                status = "⚠️ 異常" if rx_power < -10.0 else "✅ 正常"
                # 使用三元運算子：若光功率低於 -10.0 dBm 則標記為異常，否則為正常

                print(f"  {status} | Port: {port} | Rx Power: {rx_power} dBm")
                # 印出格式化後的結果

            except ValueError:
                # 數值無法轉換（例如無模組或未發光）
                # 若欄位值為 "N/A"、"--" 或 "Off" 導致轉成 float 失敗時觸發
                print(
                    f"  ℹ️ 未知/未啟用 | Port: {port} | Raw Value: {raw_rx_power}"
                )
                # 印出原始文字訊息，防止程式因轉型失敗而崩潰


def task_check_version(runner):
    """任務 B：檢查系統版本"""
    print("\n---> 開始執行任務：檢查系統版本")
    # 印出任務開始訊息

    raw_data = runner.run_command("show version")
    # 執行查詢交換機系統版本的指令

    if raw_data:
        # 如果順利抓到回傳文字
        valid_lines = [
            line.strip() for line in raw_data.splitlines() if line.strip()
        ]
        # 使用串列生成式：移除每行的前後空白，並去除掉純空白行，保留有效內容

        first_line = valid_lines[0] if valid_lines else "無內容"
        # 取得第一行非空白文字（通常包含設備型號與版本資訊）

        print(f"  系統版本資訊: {first_line}")
        # 印出系統版本資訊
```
**主程式流程控制**

```python
# ==========================================
# 4. 主程式流程控制 (Main Execution Flow)
# ==========================================
def main():
    runner = SwitchRunner(DEVICE_CONFIG)
    # 傳入設定檔，實例化 SwitchRunner 物件

    if not runner.connect():
        # 嘗試連線，如果 connect() 回傳 False（連線失敗）
        sys.exit(1)
        # 呼叫 sys.exit(1) 以離開碼 1 強制結束整個 Python 程式

    try:
        # 開始執行主要業務任務
        task_check_version(runner)
        # 執行任務 B：檢查系統版本

        task_check_ddm(runner)
        # 執行任務 A：檢查光模組 DDM

    except KeyboardInterrupt:
        # 捕捉使用者在 Terminal 按下 Ctrl+C 的中斷訊號
        print("\n[INFO] 使用者手動中斷程式執行。")

    except Exception as e:
        # 捕捉所有在任務執行過程中未預期的嚴重錯誤
        print(f"[CRITICAL] 未預期的系統錯誤: {e}")

    finally:
        # 無論 try 區塊是正常結束還是中途出錯/被中斷，finally 區塊保證一定會執行
        runner.close()
        # 確保最後一定會呼叫 close() 安全關閉 SSH 連線


if __name__ == "__main__":
    # 判斷此腳本是否為直接被執行的主程式（非被其他模組 import）
    main()
    # 呼叫 main() 開始執行程式
```

