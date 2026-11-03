# 太空艙最後三分鐘

**目前發布狀態：README 先行版。** 本儲存庫本階段僅提供專題說明；STM32 專案、Python 控制面板、簡報與實作照片尚未上傳。下文的程式設定與操作方式依製作者本機既有實作整理，涉及原始碼的步驟須在取得程式後才能執行。

NUCLEO-G474RE 微算機期中專題：透過 UART 控制面板接通備援、設定核心 PWM，並在 180 秒內按實體 B1 完成重啟。

**標準流程：START → BACKUP ON → POWER 60 → 連續維持至少 5 秒 → 按實體 B1 → 重啟成功。**

本文件依目前的 STM32 專案與既有 Python 控制面板查核整理。計時與成敗由 STM32 判定；Python 顯示板子回報的狀態。PWM 百分比是模擬設定值，沒有量測真實電壓、電流或功率。

GitHub 網址：[b11413126-glitch/SpaceCapsule](https://github.com/b11413126-glitch/SpaceCapsule)。

## 1. 專題簡介與情境故事

太空艙的主要供電系統突然故障，警報響起，操作員只剩最後三分鐘完成維修。操作員必須先接通備援電源，才能調整核心輸出；接著將核心維持在 60%，連續穩定至少 5 秒，再按下艙內確認鍵，重啟系統。

操作次序錯誤時，系統回覆原因，任務仍持續倒數。若在 180 秒到期前沒有完成有效確認，任務失敗。成功或失敗後，需要送出 `RESET` 才能開始下一輪。

本專題整合 GPIO、PLL 時脈、EXTI、NVIC、UART、Timer 定時中斷與 PWM。韌體主要使用 **STM32 HAL 與 HAL 巨集**，不是全部直接操作暫存器的實作。

## 2. 故事設備與實體替身對照

| 故事中的設備／事件 | 本次實體替身 | 實際訊號與功能 |
| --- | --- | --- |
| 維修操作台 | Python tkinter 視窗 | 透過 pyserial 傳送 UART 指令，顯示 STM32 狀態 |
| 備援電源接通 | PA7 外接綠色 LED | `BACKUP ON` 設定備援旗標並點燈；沒有真正切換電源 |
| 艙內警報 | PB0 外接紅色 LED | 維修中以名義 2 Hz 閃爍，逾時失敗後恆亮 |
| 核心輸出調整 | PA6 外接 PWM LED | TIM3_CH1 的占空比代表 0～100% 輸出設定 |
| 核心穩定檢查 | STM32 中的設定值與時間戳 | 檢查設定是否持續等於 60%，不讀取感測器 |
| 人工重啟確認 | 板上 B1 USER 按鍵 | PC13 上升緣 EXTI，經 50 ms 軟體去彈跳後記錄事件 |
| 最後三分鐘 | TIM2 毫秒計數 | 180000 個名義毫秒為期限；故事時間與程式時間採 1:1，沒有時間縮放 |
| 維修成功／失敗結果 | UART 訊息與視窗狀態 | `EVENT RESTART_SUCCESS`／`EVENT TIMEOUT FAILED`；沒有實際重啟太空艙設備 |

## 3. 操作流程、四種狀態與轉換規則

### 3.1 完整操作流程

1. 完成接線、燒錄與序列埠連線，確認板子回報 `state=IDLE`。
2. 按控制面板「啟動任務」，送出 `START`。STM32 進入 `RUNNING`，核心設定歸零，開始 180 秒倒數，紅燈閃爍。
3. 按「接通備援」，送出 `BACKUP ON`。綠燈亮起，此時才允許調整核心輸出。
4. 在核心設定欄輸入 `60`，按「送出輸出設定」，送出 `POWER 60`。
5. 保持設定不變至少 5 秒，觀察 `stable_ms=5000`、`ready=1`，視窗顯示「可按 B1」。
6. 在期限內按板上 **B1 USER**。收到 `EVENT RESTART_SUCCESS`，狀態變成 `SUCCESS`，紅燈熄滅。
7. 成功或失敗後按「結果後重設」，送出 `RESET`，回到待命。再送 `START` 才開始下一輪。

核心輸入欄改了數字後，必須按「送出輸出設定」才會傳送；只在視窗輸入不會改變 STM32。最後確認必須按實體 B1，程式沒有 UART 確認指令。

### 3.2 四種狀態

| 狀態 | 意義與時間顯示 | 綠燈／備援 | 紅燈／警報 | 核心 PWM |
| --- | --- | --- | --- | --- |
| `IDLE` | 待命；`remaining_ms=180000`，尚未開始倒數 | 關 | 滅 | 0% |
| `RUNNING` | 維修中；倒數持續進行 | 依備援旗標，接通後亮 | 每亮 250 ms、滅 250 ms，名義 2 Hz | 依接受的 `POWER N` 設定 |
| `SUCCESS` | 重啟成功；剩餘時間固定為成功按鍵時間所對應的值 | 保持亮 | 滅 | 保留 60% |
| `FAILED` | 逾時失敗；`remaining_ms=0` | 保留逾時前狀態 | 恆亮 | 保留逾時前設定，不會自動歸零 |

`SUCCESS` 的 `stable_ms` 固定回報 5000；`FAILED` 的 `stable_ms` 回報 0。只有 `RUNNING` 且符合條件時 `ready` 才為 1，因此成功後 `ready=0` 是正常行為。

### 3.3 轉換與防止重設規則

| 目前狀態 | 事件／指令 | 結果 |
| --- | --- | --- |
| `IDLE` | `START` | → `RUNNING`；備援關、PWM 0%，建立本輪任務起點 |
| `RUNNING` | `BACKUP ON` | 留在原狀態，備援旗標設為 1 |
| `RUNNING`、備援已開 | 合法 `POWER N` | 留在原狀態，更新占空比；N=60 開始／延續穩定計時，其他值清除穩定資格 |
| `RUNNING` | 有效 B1：備援開、核心 60%、已穩定至少 5000 ms、按下時未達 180000 ms | → `SUCCESS` |
| `RUNNING` | 未符合資格就按 B1 | 回覆錯誤原因，維修與倒數繼續 |
| `RUNNING` | 時間達 180000 ms，尚未成功 | → `FAILED` |
| `SUCCESS` 或 `FAILED` | `RESET` | → `IDLE`；備援關、PWM 0%、清除本輪結果 |
| 任意狀態 | `STATUS` | 只回報狀態，不改變任務 |

- 只有 `IDLE` 可以 `START`。維修中不能重新 START 延長時間，結束後也必須先 RESET。
- 只有 `SUCCESS`、`FAILED` 可以 `RESET`；`IDLE` 與 `RUNNING` 都拒絕此指令。
- 未接備援時，所有合法 `POWER N` 都被拒絕，包含 `POWER 0`。
- 重送 `POWER 60` 不會重設已開始的穩定計時；偏離 60% 後，必須再送 `POWER 60`，從新起點累計 5 秒。
- 重送 `BACKUP ON` 不會延長任務或清除穩定計時。沒有 `BACKUP OFF` 指令。
- 成功或失敗後，備援與核心指令都不再接受。

**規則適用範圍：**這些限制由 UART 命令處理實作。板上 B2／NRST 重置或斷電重開仍會重新執行韌體並回到待命，目前沒有跨重置保存任務的機制。B1 是任務確認鍵，B2 是硬體重置鍵，兩者用途不同。

## 4. 硬體材料、腳位與接線方式

### 4.1 材料

| 材料 | 數量 | 用途 |
| --- | --- | --- |
| NUCLEO-G474RE（STM32G474RET6） | 1 | 控制與判定 |
| 可傳輸資料的 USB Micro-B 線 | 1 | 連接板上 ST-LINK 的 CN1，供電、燒錄與虛擬序列埠 |
| 麵包板 | 1 | 外接 LED 與電阻 |
| 綠色 LED | 1 | 備援顯示 |
| 紅色 LED | 1 | 警報顯示 |
| PWM 顯示用 LED | 1 | 核心輸出亮度；實際顏色待補 |
| 330 Ω 電阻 | 3 | 每顆外接 LED 各串一顆限流電阻 |
| 杜邦線 | 若干 | 訊號與共同 GND |
| 電腦 | 1 | STM32CubeIDE 與 Python 控制面板 |

B1 使用板上按鍵，不需要另接按鈕；本次沒有額外電源模組、功率感測器或馬達。

### 4.2 腳位表

| STM32 腳位 | 板上位置／標示 | 程式設定 | 本專題用途 |
| --- | --- | --- | --- |
| PA7 | D11，CN5 pin 4 | GPIO 推挽輸出 | 備援綠燈；高電位亮 |
| PB0 | A3，CN8 pin 4 | GPIO 推挽輸出 | 警報紅燈；高電位亮；此處 A3 當數位輸出，沒有使用 ADC |
| PA6 | D12，CN5 pin 5 | AF2，TIM3_CH1，PWM1，高有效 | 核心 PWM LED |
| PC13 | 板上 B1 USER | 輸入、內部下拉、上升緣 EXTI13 | 人工確認；`EXTI15_10_IRQn` |
| PA2 | 板內通往 ST-LINK VCP 的 TX 訊號 | AF7，USART2_TX | STM32 傳送狀態給電腦 |
| PA3 | 板內通往 ST-LINK VCP 的 RX 訊號 | AF7，USART2_RX | STM32 接收電腦指令 |
| GND | 任一板上 GND | 共同參考地 | 麵包板與三顆 LED 負極回路 |

PA7/D11、PB0/A3、PA6/D12 與 B1 的位置已對照 [ST 官方 UM2505 板卡手冊](https://www.st.com/resource/en/user_manual/um2505-stm32g4-nucleo64-boards-mb1367-stmicroelectronics.pdf) 的按鍵說明及接頭表；MCU 模式則以本專案 `.ioc`、`main.c` 與 MSP 程式為準。

UM2505 將預設 PA2／PA3 的 VCP 路由標為 LPUART1；本專案實際使用的是**同一組腳位上的 USART2 AF7**，不是 LPUART1。VCP 是板上的實體連線，韌體選哪個 UART 由 AF 決定。不要直接把 Arduino D0／D1 當成 PA3／PA2：其路由會受焊橋設定影響。若板卡曾改過焊橋，請依 UM2505 第 7.6.5 節確認 PA2／PA3 仍接往 ST-LINK VCP。

### 4.3 接線

先斷開供電再接線，將麵包板負電源軌接至板上 GND，各 LED 使用獨立電阻：

```text
PA7 / D11 ── 330 Ω ── 綠色 LED 正極 ── LED 負極 ── GND
PB0 / A3  ── 330 Ω ── 紅色 LED 正極 ── LED 負極 ── GND
PA6 / D12 ── 330 Ω ── PWM  LED 正極 ── LED 負極 ── GND

電腦 USB ── 可傳輸資料的 USB 線 ── CN1（USB ST-LINK）
B1 USER、PA2／PA3 到 ST-LINK：使用板內既有連線
```

電阻與 LED 串聯，不能讓三顆 LED 共用一顆限流電阻。訊號來自 MCU GPIO／PWM，LED 負極接 GND。PA5 的板上 LD2 不是本程式的三顆顯示燈。

## 5. 技術勾選表、時脈與頻率計算

☑ 表示目前應用程式實際使用；☐ 表示未實作該完整項目。以下列出對應硬體暫存器，方便查閱 HAL 的底層行為，**不代表應用程式逐一直接寫入這些暫存器**。完整保留課程的 13 項，本版勾選 11 項。

| 查核 | 技術項目 | 功能、腳位、周邊及相關暫存器 | 程式依據 |
| --- | --- | --- | --- |
| ☑ | GPIO輸出 | PA7 備援、PB0 警報；`GPIOx_MODER=01`、`OTYPER`、`BSRR/BRR`，RCC GPIO 時脈 | `MX_GPIO_Init()`、`App_UpdateLEDs()`，使用 `HAL_GPIO_Init/WritePin` |
| ☑ | GPIO輸入 | PC13 作為 B1 的 EXTI 輸入來源；`GPIOC_MODER=00`、`PUPDR` 下拉；不是主迴圈輪詢 `HAL_GPIO_ReadPin()` | `MX_GPIO_Init()` 的 `GPIO_MODE_IT_RISING`、`GPIO_PULLDOWN` |
| ☑ | 複用功能AF | PA6 AF2→TIM3_CH1；PA2／PA3 AF7→USART2；`GPIOA_MODER=10`、`AFRL` | `HAL_TIM_MspPostInit()`、`HAL_UART_MspInit()` |
| ☑ | PLL 170MHz | HSI16 ÷4 ×85 ÷2，供 CPU、APB 與 Timer；`RCC_CR`、`PLLCFGR`、`CFGR`、`FLASH_ACR`，PWR boost 設定 | `SystemClock_Config()`；`FLASH_LATENCY_4`、`PWR_REGULATOR_VOLTAGE_SCALE1_BOOST` |
| ☑ | 外部中斷EXTI | PC13 上升緣；`SYSCFG_EXTICR4` 選 GPIOC、`EXTI_RTSR1/FTSR1`、`IMR1`、`PR1`；HAL 處理並清除 pending | `EXTI15_10_IRQHandler()`→`HAL_GPIO_EXTI_IRQHandler()`→`HAL_GPIO_EXTI_Callback()` |
| ☑ | NVIC致能 | 致能 TIM2、USART2、EXTI15_10 三支 IRQ；NVIC `ISER`，向量表對應處理函式 | `HAL_NVIC_EnableIRQ()` 位於 `main.c` 與 MSP |
| ☐ | 優先權與巢狀 | 有設定優先權，但三支 IRQ 全為搶占優先權 0、子優先權 0；`IPR`、`SCB_AIRCR.PRIGROUP` 對應設定；沒有設計不同優先權互搶與驗證巢狀執行 | `HAL_Init()` 採 `NVIC_PRIORITYGROUP_4`；單有 `SetPriority()` 不算完成巢狀功能 |
| ☑ | 軟體去彈跳 | B1 兩次接受事件相隔至少 50 ms；首次按下可接受；軟體時間戳 `last_press`，無專用去彈跳暫存器 | `HAL_GPIO_EXTI_Callback()`、`DEBOUNCE_MS=50` |
| ☑ | UART傳送 | USART2，PA2 TX，115200 8N1；`BRR`、`CR1/CR2/CR3`、`ISR.TXE/TC`、`TDR`；HAL 輪詢阻塞傳送 | 主迴圈的 `App_Send()` 呼叫 `HAL_UART_Transmit()`，逾時參數 100 ms |
| ☑ | UART接收＋環形緩衝 | USART2，PA3 RX；`CR1.RXNEIE`、`ISR.RXNE`、`RDR`，每字元中斷入軟體環形緩衝；FIFOs 已停用 | `HAL_UART_Receive_IT()`、`HAL_UART_RxCpltCallback()`；主迴圈 `App_ProcessUART()` |
| ☑ | Timer定時中斷 | TIM2，PSC=169、ARR=999，名義每 1 ms 更新；`PSC`、`ARR`、`CNT`、`DIER.UIE`、`SR.UIF`、`CR1.CEN` | `MX_TIM2_Init()`、`HAL_TIM_Base_Start_IT()`、週期完成 callback 累加 `app_ms` |
| ☑ | PWM | TIM3_CH1／PA6 AF2；PSC=169、ARR=999，1 kHz PWM1；`CCMR1.OC1M`、`CCER.CC1E`、`CCR1`、`CR1.CEN` | `MX_TIM3_Init()`、`HAL_TIM_PWM_Start()`、`App_SetPower()` |
| ☐ | 輸入捕捉 | 未設定輸入捕捉通道，沒有量測外部脈波的頻率或脈寬 | 應用程式沒有啟動 `HAL_TIM_IC_*`；不能把 PWM 的 CCR1 使用算成輸入捕捉 |

### 5.1 系統時脈

依 `SystemClock_Config()`，時脈來源為內部 HSI 16 MHz，並非板上 HSE：

```text
PLL 輸入       = 16 MHz / PLLM(4)           = 4 MHz
PLL VCO        = 4 MHz × PLLN(85)           = 340 MHz
SYSCLK         = 340 MHz / PLLR(2)          = 170 MHz
HCLK           = SYSCLK / AHB分頻(1)        = 170 MHz
PCLK1、PCLK2   = HCLK / APB分頻(1)          = 170 MHz
TIM2、TIM3時脈 = PCLK1（APB1分頻為1）        = 170 MHz
```

APB1 分頻為 1，本設定不再把 Timer 時脈乘 2。`.ioc` 保留 HSE／LSE 腳位標示，不表示它們是本任務計時來源。

### 5.2 Timer、PWM 與 UART

TIM2、TIM3 皆向上計數，PSC=169、ARR=999。除數必須加 1：

```text
計數頻率       = 170000000 / (169 + 1)       = 1000000 Hz
一輪更新頻率   = 1000000 / (999 + 1)         = 1000 Hz
TIM2更新週期   = 1 / 1000                   = 1 ms
TIM3 PWM頻率   = 1000 Hz                    = 1 kHz
PWM比較值      = (ARR + 1) × N / 100         = 10 × N
POWER 0        → CCR1=0                     → 0%
POWER 60       → CCR1=600                   → 60%
POWER 100      → CCR1=1000                  → 100%
```

TIM3 是 PWM1、高有效，CNT 為 0～999，因此 CCR1=1000 可維持全週期有效。PWM 的頻率不隨 POWER 改變，變的是占空比。

USART2 使用 PCLK1=170 MHz、時脈預分頻 1、16 倍取樣。名義 `BRR ≈ round(170000000 / 115200) = 1476`，由 HAL 設定；這是參數推算，並非串列訊號的實測值。

上述 170 MHz、1 ms、1 kHz、2 Hz 都是依設定推得的名義值。本文件沒有示波器、邏輯分析儀或碼錶誤差數據，不宣稱實際時脈或計時零誤差。

## 6. 開發環境、編譯與燒錄

### 6.1 已確認的環境資訊

| 項目 | 本機與專案檔案已確認的資訊 |
| --- | --- |
| 開發板／MCU | NUCLEO-G474RE／STM32G474RET6，Arm Cortex-M4 |
| 專案形式 | STM32CubeIDE 專案；根目錄有 `.project`、`.cproject` 與 `SpaceCapsule.ioc` |
| STM32CubeIDE | **2.2.0**；依安裝目錄的 `.eclipseproduct` 核對 |
| CubeMX 設定檔版本 | `.ioc` 記錄 **6.18.1**；這是專案設定檔記錄，未另外確認 CubeMX 執行程式版本 |
| STM32CubeG4 套件 | `.ioc` 記錄 `STM32Cube FW_G4 V1.6.3` |
| 韌體函式庫 | 專案內的 STM32G4 HAL 與 CMSIS |
| 編譯器 | 既有 `Debug/makefile` 記錄 GNU Tools for STM32 `14.3.rel1`；C 原始檔使用 `gnu11` |
| 燒錄／除錯介面 | 板上 ST-LINK，使用 SWD |
| Windows | 版本 **25H2**，OS Build **26200.9711**；依本機登錄值核對 |
| Python | **3.13.15，64 位元**；以目前使用的 Python 執行檔查詢確認 |
| pyserial | **3.5**；由同一個 Python 環境匯入 `serial` 後核對 |
| GUI／Tcl/Tk | tkinter／ttk；Tcl/Tk **8.6.15**，由同一 Python 環境查詢 |
| ST-LINK 驅動 | 精確版本待補；本次無法透過系統查詢取得 |

以上安裝版本反映本輪核對時的電腦環境，不追溯宣稱為先前各次實機驗收當時的版本；本次未重新執行實機驗收。

製作者本機已有 `SpaceCapsule` 專案。**本公開儲存庫目前僅有 README，以下步驟須取得完整 STM32 專案後才能執行。**本次文件整理沒有重新編譯、燒錄或操作實機。資料夾中雖有既有 `Debug/SpaceCapsule.elf`，不能據此宣稱目前原始碼已通過本次建置。

### 6.2 匯入與編譯

1. 準備 STM32CubeIDE、ST-LINK 驅動與相符的 STM32CubeG4 套件。保留完整專案結構，不要只匯入 `main.c`。
2. 在 STM32CubeIDE 使用 `File → Import → General → Existing Projects into Workspace`；若版本提供 STM32 專案專用匯入項目，也可選該項。選擇含 `.project` 的 `SpaceCapsule` 根目錄並匯入。
3. 選取專案並開啟 `SpaceCapsule.ioc` 檢視設定，核對第 4、5 節。現有專案已有產生的程式碼，不需為了閱讀 `.ioc` 就重新產生程式。
4. 使用 `Debug` 組態，執行 `Project → Build Project`，檢查 Console 是否成功完成且沒有錯誤。
5. 確認輸出 `Debug/SpaceCapsule.elf`。`Debug/` 內 makefile 是產生檔，含原電腦的絕對路徑；移機後應由 CubeIDE 重建建置設定，不直接依賴複製來的舊 makefile。

### 6.3 燒錄與啟動

1. 核對接線，將 USB 線接到板上 `CN1 / USB ST-LINK`。
2. 在 CubeIDE 使用專案的 STM32 C/C++ Application 執行／除錯設定，確認目標為本板、介面為 ST-LINK／SWD、映像為剛編譯的 ELF。
3. 執行 Run 或 Debug 下載韌體。若 Debug 停在 `main()`，按 Resume 讓程式繼續執行。
4. 開啟控制面板並連線，按「查詢狀態」。應看到 `IDLE`、備援關閉、核心 0%；開機訊息只在韌體啟動時送出，較晚連線未看到該訊息也可用 `STATUS` 查詢。
5. 依第 3 節操作一輪。任務確認使用 B1 USER。

目前 CubeIDE、Windows、Python、pyserial 與 Tcl/Tk 版本已列於第 6.1 節。仍需補充 ST-LINK 驅動版本；若先前驗收使用不同環境，請另記錄差異。

## 7. Python 套件安裝、控制面板啟動與 COM 埠

### 7.1 Python 控制面板的交付狀態

既有的 `space_capsule_panel.py` 目前另行保存在製作者本機；先前比對的兩份檔案內容一致。本儲存庫尚未提供該程式，不能只下載本 README 就啟動控制面板。

後續預定交付位置為 `python/space_capsule_panel.py`。待程式實際上傳後，再加入可使用的檔案連結；本階段僅上傳 README。

### 7.2 安裝與啟動

在 Windows PowerShell 執行下列指令。此處是供使用者之後操作的步驟，本次沒有安裝套件或啟動面板：

```powershell
py -3 --version
py -3 -m pip install pyserial==3.5
py -3 -m tkinter
```

`pyserial` 的匯入名稱是 `serial`，套件安裝名稱是 `pyserial`。`tkinter` 是 Python 的標準 GUI 模組，`py -3 -m tkinter` 用來確認能開測試視窗；測試後將其關閉。若缺少 tkinter，請檢查 Python 安裝的 Tcl/Tk 元件，不要改裝名稱相近的 pip 套件。

取得控制面板並放到上述預定位置後，在專案根目錄啟動：

```powershell
py -3 .\python\space_capsule_panel.py
```

若系統沒有 `py` 啟動器，可使用 `python` 取代上述 `py -3`，但安裝套件與執行程式必須使用同一個 Python 環境。已確認本機可匯入 pyserial 與 tkinter，不需為了本次文件更新重新安裝。

### 7.3 選擇 COM 埠

1. 保持板子供電並接在 ST-LINK USB 接頭。
2. 開啟控制面板，按「重新掃描」。程式會記錄偵測到的 `PORT COMx: 描述`，優先預選描述含 ST-LINK 的埠，找不到時才選第一個埠；仍須人工核對。
3. 在 Windows 裝置管理員的「連接埠（COM 和 LPT）」確認 ST-LINK Virtual COM Port 的編號，再在面板下拉選單選取同一個 COM 埠。不要假設固定為 COM3 或其他編號。
4. 按「連線」。程式以 **115200 baud、8 data bits、無同位檢查、1 stop bit、無硬體流量控制** 開啟埠，先送一個換行，約 200 ms 後查詢 `STATUS`。
5. 看到「通訊正常｜已收到板子狀態」後再操作任務。只有「序列埠已開啟」還不能證明韌體有回覆。

同一 COM 埠同時只讓一個程式使用。若連線失敗，先關閉其他序列終端機，檢查 USB 資料線、驅動與埠號；若能開埠但沒有狀態，檢查韌體是否已執行、UART 設定及板卡 VCP 路由。

### 7.4 視窗與更新方式

| 視窗操作／顯示 | 對應功能 |
| --- | --- |
| 啟動任務／接通備援 | 傳送 `START`／`BACKUP ON` |
| 核心設定＋送出輸出設定 | 本機檢查 0～100 整數，傳送 `POWER N` |
| 查詢狀態／結果後重設 | 傳送 `STATUS`／`RESET` |
| 任務狀態、剩餘時間、備援、核心、穩定時間、確認資格 | 解析 STM32 的 `STATUS`，不在 Python 判定成敗 |
| 通訊紀錄 | `TX` 為送出、`RX` 為收到；前方時刻是電腦本機記錄時間 |

Python 使用 `root.after(50, poll)` 每約 50 ms 非阻塞讀取序列埠，沒有另開接收執行緒。剩餘毫秒以向上取整換成 `mm:ss` 顯示，穩定秒數顯示到小數點後兩位；這些顯示位數不代表量測精度。

STM32 只有在維修中才約每 250 ms 主動送一筆 STATUS。待命或結束後若超過 2 秒沒有新狀態，面板會提示數值是上次回報，可按「查詢狀態」更新，不能僅憑此提示判定通訊故障。斷線後舊數值也會保留。只關閉面板或斷開 COM、且板子仍持續供電時，任務繼續由 STM32 計時。

## 8. UART 指令、回覆格式與操作範例

### 8.1 傳輸規格

- USART2：115200、8N1、無流量控制，PA2 TX／PA3 RX。
- 指令為**區分大小寫的 ASCII 文字**，一行一個指令，以 LF（`\n`）、CR（`\r`）或 CRLF 結尾。面板送 LF，韌體回覆 CRLF；空行忽略。
- 指令不會自動去掉前後空白。除 `POWER` 與數字之間的一個空格外，請使用表中的原樣格式。
- `POWER N` 的 N 為 0～100、1～3 位 ASCII 數字；可有前導零，不能有正負號、小數或尾端空白。面板會轉成一般十進位整數再送出。
- 行緩衝長度 64 bytes，可接受至多 63 個字元，不含行尾；環形接收緩衝陣列 256 bytes，保留一格分辨滿／空，實際可排入 255 bytes。

### 8.2 指令表

| 指令 | 接受條件 | 接受後回覆與效果 |
| --- | --- | --- |
| `START` | `IDLE` | `OK START`，再回 STATUS；開始新任務 |
| `BACKUP ON` | `RUNNING` | `OK BACKUP_ON`，再回 STATUS；接通備援 |
| `POWER N` | `RUNNING` 且備援已開，N=0～100 | `OK POWER`，再回 STATUS；更新 PWM 與穩定計時資格 |
| `STATUS` | 任意狀態 | 回一筆 STATUS，不改變狀態 |
| `RESET` | `SUCCESS` 或 `FAILED` | `OK RESET`，再回 STATUS；回待命並清除本輪設定 |

沒有 `BACKUP OFF`、`STOP` 或以 UART 代按 B1 的命令。

### 8.3 狀態與事件格式

開機時輸出：

```text
SPACE_CAPSULE READY
COMMANDS: START | BACKUP ON | POWER N | STATUS | RESET
STATUS state=IDLE remaining_ms=180000 backup=0 power=0 stable_ms=0 ready=0
```

STATUS 的欄位順序如下：

```text
STATUS state=<狀態> remaining_ms=<剩餘毫秒> backup=<0或1> power=<0到100> stable_ms=<0到5000> ready=<0或1>
```

| 欄位 | 意義 |
| --- | --- |
| `state` | `IDLE`、`RUNNING`、`SUCCESS`、`FAILED` |
| `remaining_ms` | 本輪剩餘時間；待命為 180000，成功後凍結，失敗為 0 |
| `backup` | 韌體備援旗標，0 未接通、1 已接通 |
| `power` | PWM 輸出設定百分比，不是真實功率量測 |
| `stable_ms` | 連續維持 60% 的名義毫秒數，回報上限 5000 |
| `ready` | 仍在 RUNNING 且備援開、核心 60%、穩定至少 5 秒時為 1 |

格式示意如下，數字只用來解釋欄位，**不是實測紀錄**：

```text
STATUS state=RUNNING remaining_ms=120000 backup=1 power=60 stable_ms=5000 ready=1
```

B1 確認成功或任務逾時時，分別輸出下列事件，並再送一筆 STATUS：

```text
EVENT RESTART_SUCCESS
EVENT TIMEOUT FAILED
```

一般命令成功後會附 STATUS；一般錯誤回覆不一定附 STATUS。維修中的週期回報也會穿插在操作紀錄裡，接收端不能假設每送一個指令只會收到一行。

### 8.4 錯誤回覆

| 實際回覆字串 | 原因與處理方式 |
| --- | --- |
| `ERR START_REQUIRES_IDLE` | 目前非待命；任務結束後先 RESET |
| `ERR RESET_REQUIRES_FINISHED` | 尚未成功或失敗，不能靠 RESET 清除任務 |
| `ERR NOT_RUNNING` | 非維修中卻送備援或合法 POWER 指令 |
| `ERR BACKUP_REQUIRED` | 尚未接通備援就設定核心 |
| `ERR BUTTON_NOT_RUNNING` | 非維修中按 B1 |
| `ERR BUTTON_BACKUP_REQUIRED` | 未接通備援就按 B1 |
| `ERR BUTTON_POWER_MUST_BE_60` | 核心不是 60% 或沒有有效穩定計時 |
| `ERR BUTTON_WAIT_5_SECONDS` | 按下時穩定不足 5 秒，或按鍵事件早於本次穩定起點 |
| `ERR COMMAND_USE_START_BACKUP_ON_POWER_N_STATUS_RESET` | 未知命令、大小寫／空白不符，或 POWER 格式／範圍不合法 |
| `ERR ASCII_COMMANDS_ONLY` | 收到非可列印 ASCII 字元；該行丟棄，送換行後重送 |
| `ERR COMMAND_TOO_LONG` | 指令超過 63 字元；該行丟棄，送換行後重送 |
| `ERR RX_LOST_SEND_NEWLINE_THEN_RETRY` | 接收環形緩衝滿或 UART 錯誤；清除待收資料，先送換行建立新邊界再重送 |

### 8.5 最短操作範例

使用面板時依第 3 節按鈕操作；使用一般序列終端機時設定 115200 8N1，逐行傳送下列內容。以下只列命令，不把說明文字送進 UART：

```text
STATUS
START
BACKUP ON
POWER 60
```

等待回報 `ready=1`，且尚未逾時，再按板上 B1。收到 `EVENT RESTART_SUCCESS` 與 `state=SUCCESS` 後，才能送：

```text
RESET
```

若要重現防呆，可在 START 後先送 `POWER 60`，應收到 `ERR BACKUP_REQUIRED`；完成備援與 POWER 60 後，未滿 5 秒按 B1，應收到等待提示，任務繼續。避免密集連續灌入命令造成接收溢位。

## 9. 程式檔案分工、中斷與主迴圈

### 9.1 檔案分工

以下列出本機既有實作的檔案與職責；這些原始碼尚未上傳，因此暫不提供檔案連結。

| 檔案 | 職責 |
| --- | --- |
| `SpaceCapsule.ioc` | CubeMX 的 MCU、GPIO、AF、PLL、TIM2／TIM3、USART2 與 NVIC 設定來源 |
| `Core/Src/main.c` | 周邊初始化、四態狀態機、UART 命令解析、計時、LED、PWM、驗證條件及 HAL callbacks |
| `Core/Inc/main.h` | 共用宣告、PA7／PB0 LED 名稱與 GPIO 定義；PC13、PA6 等設定另在 `.ioc`／初始化程式內 |
| `Core/Src/stm32g4xx_it.c` | IRQ 入口，轉交 HAL 處理；SysTick 呼叫 `HAL_IncTick()` |
| `Core/Src/stm32g4xx_hal_msp.c` | 周邊時脈、PA6 AF2、PA2／PA3 AF7、TIM2 與 USART2 NVIC 設定 |
| `Core/Inc/stm32g4xx_hal_conf.h` | HAL 模組與系統 tick 等設定 |
| `Core/Startup/`、`Drivers/` | 啟動向量表、CMSIS 與 STM32 HAL 底層驅動 |
| `STM32G474RETX_FLASH.ld` | Flash 建置所用的記憶體連結配置 |
| `space_capsule_panel.py`（目前位於專案外，見第 7 節） | tkinter 視窗、COM 掃描、pyserial 收發、STATUS 解析及通訊紀錄 |
| `Debug/` | 既有建置輸出與產生檔，不是主要程式來源 |

### 9.2 中斷只處理短工作

| 來源／入口 | callback 工作 | 留給主迴圈的工作 |
| --- | --- | --- |
| TIM2 → `TIM2_IRQHandler()` → `HAL_TIM_IRQHandler()` | `HAL_TIM_PeriodElapsedCallback()` 每次只累加 `app_ms` | 以時間差判斷倒數、穩定時間與燈號 |
| USART2 → `USART2_IRQHandler()` → `HAL_UART_IRQHandler()` | `HAL_UART_RxCpltCallback()` 將單一字元入環形緩衝，重新掛上下一字元接收 | 組行、檢查 ASCII／長度、解析命令、回覆 |
| PC13 → `EXTI15_10_IRQHandler()` → `HAL_GPIO_EXTI_IRQHandler()` | `HAL_GPIO_EXTI_Callback()` 執行 50 ms 去彈跳，保留最早尚未處理的按鍵時間與旗標 | 判斷備援、60%、穩定 5 秒與任務期限，決定成敗 |
| UART 錯誤 callback | 記下錯誤；若 HAL 接收已結束則嘗試重掛接收 | 清空不可信的待收命令、回報重新同步提示 |
| SysTick → `SysTick_Handler()` | `HAL_IncTick()`，供 HAL 時基使用 | 任務主要計時使用 TIM2 的 `app_ms`，不是 Python 或 SysTick 倒數 |

中斷中沒有列印 UART 訊息或完整解析指令。`App_Send()` 只從主迴圈路徑呼叫，使用 HAL 阻塞傳送；因此主迴圈仍可能因傳送等待而延後處理其他工作。

### 9.3 主迴圈與共享資料

初始化順序為 `HAL_Init()` → 系統時脈 → GPIO／TIM3／USART2／TIM2 → `App_Init()`。`App_Init()` 設定待命、啟動 PWM、UART 單字元中斷接收、TIM2 定時中斷，再送開機訊息。

`while (1)` 持續執行 `App_Run()`：

1. 取出待處理的 B1 事件。
2. 檢查目前是否逾時。
3. 從環形緩衝取出 UART 字元，遇換行才執行命令；處理字元時也先檢查待處理的 B1。
4. 再檢查逾時，更新備援／警報 LED。
5. 若仍在維修中，距上次回報至少 250 ms 時送 STATUS。

`app_ms`、按鍵事件、環形緩衝索引等使用 `volatile` 與 ISR 共用；主迴圈取出按鍵事件／時間戳、清理接收錯誤時，短暫透過 PRIMASK 關閉中斷並還原原狀態。任務狀態、備援旗標、輸出設定等由主迴圈管理。

計時使用無號時間差，例如 `now - mission_start`，不在每輪任務清零全域 `app_ms`。B1 的 5 秒與 180 秒資格以 ISR 記錄的 `pressed_at` 判斷；主迴圈安排先處理按鍵再檢查目前逾時。非常接近期限時的中斷／主迴圈交錯仍列為待驗證，不宣稱所有邊界情境均已通過。

## 10. 驗收條件與結果

**結果來源：**以下只有 5 項標為「使用者實測回報通過」，依使用者既有實機測試回報整理。本次僅檢查檔案及撰寫文件，沒有重新跑實機測試。未提供原始通訊紀錄、量測數據或測試日期，不填入推測值。

| 編號 | 動作／驗收條件 | 預期觀察 | 結果 |
| --- | --- | --- | --- |
| A01 | START → BACKUP ON → POWER 60，連續至少 5 秒後、180 秒內按 B1 | `EVENT RESTART_SUCCESS`、`state=SUCCESS` | **使用者實測回報通過** |
| A02 | START 後，未接備援就送 POWER 60 | 拒絕調整；`ERR BACKUP_REQUIRED`，輸出設定不變 | **使用者實測回報通過** |
| A03 | 接備援並設 60%，未滿 5 秒就按 B1 | 拒絕確認；`ERR BUTTON_WAIT_5_SECONDS`，任務持續 | **使用者實測回報通過** |
| A04 | 核心 60% 已累計部分時間，改成其他值，再改回 60% | 偏離時穩定資格清除，回到 60% 後重新計時 | **使用者實測回報通過** |
| A05 | 開始任務後，180 秒內未完成有效確認 | `EVENT TIMEOUT FAILED`、`state=FAILED`、剩餘 0 | **使用者實測回報通過** |
| A06 | POWER 60 計時中再次送 POWER 60 | 穩定時間延續，不重新起算 | 待驗證；程式已有對應分支 |
| A07 | 維修中再次 START | `ERR START_REQUIRES_IDLE`，原任務期限不變 | 待驗證 |
| A08 | IDLE 或 RUNNING 時送 RESET | `ERR RESET_REQUIRES_FINISHED`；維修中的倒數不重設 | 待驗證 |
| A09 | 分別在成功、失敗後 RESET，再 START | 回待命時備援關、PWM 0%，下一輪可開始 | 待驗證 |
| A10 | 按住／快速重按 B1，檢查 50 ms 去彈跳與事件保留 | 去彈跳時間內的重複上升緣不重複入列，保留最早待處理事件 | 待驗證 |
| A11 | 送小寫命令、POWER -1／101／60.0、過長或非 ASCII 指令 | 依第 8 節拒絕，不改變任務設定 | 待驗證 |
| A12 | 測試環形緩衝滿與 UART 接收錯誤後重新同步 | 回接收遺失提示；送換行後可接受新完整指令 | 待驗證 |
| A13 | 觀察各狀態的三顆 LED；必要時使用儀器量 PA6 | 燈號符合第 3 節；PWM 名義 1 kHz，60% 時 CCR1=600 | 待驗證；沒有頻率、占空比或計時誤差實測數據 |
| A14 | 成功／失敗後再送備援、POWER 或 START | 備援／POWER 回 NOT_RUNNING；START 要求先回 IDLE | 待驗證 |
| A15 | 穩定時間接近 5000 ms、期限接近 180000 ms 時按 B1，並測同時到達的命令 | 依時間戳與處理順序判定；期限已到不可當成有效期限內按鍵 | 待驗證，需保留通訊與事件紀錄 |
| A16 | 維持板子供電，斷開／重連面板 COM | 韌體任務繼續；重連 STATUS 反映板上狀態 | 待驗證 |

「程式已有對應分支」只代表靜態檢查可找到該邏輯，不等於實機測試通過。後續驗證可在本表補上實際觀察、截圖與結果，不將目前待驗證項目提前改為通過。

## 11. 實作照片

本次已檢查專案根目錄：**目前沒有 `docs/` 資料夾，也沒有可引用的實作照片**，因此本文件不放不存在的圖片連結。

請補拍／擷取以下內容，之後存入專案 `docs/`，再依真正檔名加入相對路徑圖片與圖說：

| 建議檔名（尚無檔案） | 待補影像 | 拍攝內容與圖說重點 |
| --- | --- | --- |
| `hardware.jpg` | 板子與接線全景 | NUCLEO-G474RE、USB 線、麵包板及三顆 LED，能辨識實際組裝 |
| `wiring.jpg` | 接線特寫 | D11／A3／D12、各自 330 Ω 電阻、LED 極性與共同 GND；與第 4 節對得上 |
| `running.jpg` | 維修中燈號 | 備援綠燈亮、PWM 核心燈亮；紅燈閃爍可拍亮／暗兩張或短影片 |
| `success.png` | 重啟成功 | B1 確認後的板子與控制面板 `SUCCESS`／成功事件 |
| `timeout.png` | 逾時失敗 | 紅燈恆亮及控制面板 `FAILED`／逾時事件 |
| `serial-log.png` | 通訊畫面 | START、BACKUP ON、POWER 60、ready=1 與結果事件，可另補防呆錯誤回覆 |

可再補一張 `stable-reset.png`：先送 `POWER 60` 等約 3 秒，再送 `POWER 59`，截取 `power=59 stable_ms=0 ready=0` 的實際回報，用來佐證 A04。表中檔名只是建議，目前均未附上影像。成功與失敗的面板截圖應搭配板子照片，顯示實際燈號。

使用自己的實機照片，避免拍入臉、學生證、手機或其他個資；控制面板截圖只是通訊證據，仍需搭配實際板子與接線照片。

## 12. AI 使用說明、已知限制與待補資訊

### 12.1 AI 使用說明

AI 協助本專題的功能規畫、STM32 程式與 Python 控制介面製作、除錯，以及 README 等文件整理。由我執行開發環境設定、實體接線、韌體燒錄與實機驗收；第 10 節標示的 5 項通過結果來自我的實機測試回報。

前次 README 整理以既有 `.ioc`、STM32 原始檔、Python 控制面板與老師的 W08、W09 PDF 為依據，整理 PLL、Timer 與 PWM 參數，區分已回報通過與待驗證項目。本輪再讀取既有 README、`.ioc` 與 Python 控制面板，核對兩份 Python 檔案雜湊、目前安裝版本及 GitHub 專案資訊。

README 由 AI 協助整理與查核，並依本階段只發布說明文件的範圍調整。文件準備期間未修改 STM32／Python 程式、重新編譯或燒錄、操作板子；發布狀態以本儲存庫實際檔案與提交紀錄為準。照片與待驗證結果仍須由本人實際取得。

**需由我補充：**上述為目前提供的協作範圍；若個別功能實際由我自行撰寫、曾手動修改，或 AI 協助範圍不同，應補上具體檔案／功能與查核方式。現有檔案不能還原每段程式的完整作者歷程，不虛構逐行貢獻比例。

### 12.2 已知限制與背景補充

- 備援是軟體旗標與 LED，沒有真正的電源切換；60% 穩定是設定值維持不變，沒有閉迴路控制或實際電力感測。
- 失敗後紅燈恆亮，但備援旗標與 PWM 設定保留；只有 RESET 回待命才清除。這是現有程式行為，不把它描述成自動切斷核心。
- UART 的 START／RESET 防呆不能阻止實體重置或斷電。任務與結果沒有持久化保存。
- 計時基於 HSI／PLL 與 TIM2 ISR 累加，並非經校正的時間量測系統。UART 阻塞傳送、主迴圈負載與中斷處理會影響處理／回報延遲；尚無精確誤差資料。
- B1 事件緩衝只保留一筆最早尚未處理的有效按下，不是完整按鍵事件佇列。去彈跳為上升緣時間間隔法，沒有另外確認長按或釋放狀態。
- UART 環形緩衝容量有限，收到不完整、過長或遺失的資料時會丟棄並要求重新同步，不能保證任意高速灌入都無遺失。
- 控制面板只有在收到 STATUS 時更新數值，沒有自行倒數；斷線時可能保留舊值。GUI 每 50 ms 輪詢、韌體每約 250 ms 回報不代表端到端延遲已量測。
- Python 控制面板尚未放在專案內，也沒有套件版本鎖定檔；單獨拿到此 STM32 資料夾不足以啟動 GUI，需另取第 7 節的既有檔案。
- 中斷優先權雖有設定，但沒有實作不同優先權互搶的巢狀示範；輸入捕捉未使用。

### 12.3 集中待補清單

1. **實作照片與通訊截圖：**依第 11 節補拍，提供實際檔名與圖說。
2. **後續交付檔案：**公開儲存庫已建立，本階段只發布 README；STM32 專案、Python 控制面板、簡報與照片後續再補。
3. **交付用 Python 位置：**目前檔案位置、內容一致性與啟動方式已確認；仍須將既有檔案原樣複製至預定的 `python/space_capsule_panel.py`，完成後再更新第 7.1 節的交付狀態。
4. **剩餘環境資訊：**ST-LINK 驅動的精確版本、PWM LED 實際顏色，以及板卡焊橋是否曾修改。CubeIDE 2.2.0、Windows 25H2／26200.9711、Python 3.13.15、pyserial 3.5 與 Tcl/Tk 8.6.15 已由本輪核對補上。
5. **尚未回報的驗收結果：**第 10 節 A06～A16，尤其重複 POWER 60、START／RESET 防呆與期限邊界。若有原始紀錄可附上；沒有的測試日期、誤差與儀器數據不補造。
6. **AI 協作細節：**個別檔案／功能的 AI 協助與本人修改、查核經過，若與第 12.1 節不同請修正。

### 12.4 查核來源與課程對照

- 實作依據：`SpaceCapsule.ioc`、`Core/Src/main.c`、`Core/Inc/main.h`、`Core/Src/stm32g4xx_it.c`、`Core/Src/stm32g4xx_hal_msp.c`，並參照專案內 HAL／CMSIS 驅動確認暫存器對應。
- 控制面板依據：第 7 節列出的既有 `space_capsule_panel.py`，本輪重新核對兩份檔案的 SHA-256 相同。
- 本輪環境依據：CubeIDE 安裝目錄 `.eclipseproduct`、同一個 Python 執行檔的版本與模組查詢，以及 Windows 登錄值。
- 交付範圍：本階段只發布 README；原始碼、控制面板、簡報與實作照片列為後續補充。公開儲存庫為 [SpaceCapsule](https://github.com/b11413126-glitch/SpaceCapsule)。
- 課程文件：前次 README 整理已讀取本機可取得的 `1420微算機_W08_期中整合演練.pdf` 與 `1420微算機_W09_期中實作考.pdf`，並檢視 W09 技術勾選表、README 範本與實作照片頁。課堂 PDF 不在本專案內，故不建立不存在的相對連結。
- 板卡對照：[ST 官方 UM2505](https://www.st.com/resource/en/user_manual/um2505-stm32g4-nucleo64-boards-mb1367-stmicroelectronics.pdf)，用於接頭、B1 與 VCP 實體路由；應用邏輯以本專案程式為準。

W08／W09 要求技術勾選表完整列 13 項、只勾實際使用、提供驗收證據、照片、完整 README 與 AI 揭露；課程另要求公開 repo 及最終投影片。本次先發布 README 文件；原始碼、控制面板、簡報、照片與其他待驗證證據仍依上列清單補齊。