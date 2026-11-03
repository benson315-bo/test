STM32 NUCLEO-G474RE｜LED 警報解除模擬系統
這是一套使用 STM32 NUCLEO-G474RE、兩顆外接 LED、有源蜂鳴器、板載 B1 緊急停止按鈕，以及電腦 Python 命令列輸入密碼的教學專題。以 STM32CubeIDE + HAL C 程式實作，電腦端使用 Python + pyserial。本版 保留 PC13 原本的緊急停止功能。
> **注意：** 專題名稱是「拆彈模擬」，實際電路僅用於 LED、蜂鳴器與密碼互動，沒有任何實際爆炸裝置。
1. 功能
上電／按 RESET 後立即開始 30 秒倒數。
PC8 外接主 LED，每 500 ms 切換亮滅；每次密碼錯誤，切換間隔依序縮短為 400、300、200、100 ms。
PC6 / TIM3_CH1 外接 LED 提供 1 kHz PWM 呼吸燈，控制程式每 20 ms 更新占空比；最低亮度 5%，初始上限 20%，每次錯誤上限增加 20%，最多 100%。
使用 LPUART1（PA2 TX、PA3 RX，115200、8N1） 連接 ST-LINK 虛擬 COM 埠，輸入密碼 `1234` 可解除警報。
30 秒到期或密碼錯誤 5 次後失敗鎖定：PC8 恆亮、PC6 100%，PB1 蜂鳴器響 2 秒後停止；失敗後不能再用密碼解除。
PC13 板載 B1 按鈕可隨時緊急停止，關閉所有 LED、蜂鳴器和倒數。
PB0、PC7 是預留的玩家按鈕輸入，目前沒有觸發動作。
30 秒計時、LED 閃爍、呼吸燈與蜂鳴器定時均使用 `HAL_GetTick()` 非阻塞處理；UART 接收使用中斷。
2. 腳位表
用途	腳位	STM32CubeMX 設定
緊急停止（板載 B1，不改）	PC13	GPIO_EXTI13 / Rising Edge / No Pull
玩家 1 預留按鈕	PB0	GPIO_Input / Pull-up
玩家 2 預留按鈕	PC7	GPIO_Input / Pull-up
外接主 LED	PC8	GPIO_Output / Push-Pull / 初始 Low
外接呼吸 LED	PC6	TIM3_CH1 / Alternate Function AF2
有源蜂鳴器驅動	PB1	GPIO_Output / Push-Pull / 初始 Low
電腦 UART TX	PA2	LPUART1_TX / AF12
電腦 UART RX	PA3	LPUART1_RX / AF12
補充： 本版主 LED 已從板載 PA5 改到外接 PC8；PA5 不再由程式控制。PC6 必須配置為 TIM3_CH1，不能同時設為 GPIO_Output。
3. 外接硬體
LED
PC8 → 330 Ω 電阻 → 主 LED 正極；LED 負極 → GND。
PC6 → 330 Ω 電阻 → 呼吸 LED 正極；LED 負極 → GND。
玩家按鈕
PB0 → 按鈕 → GND。
PC7 → 按鈕 → GND。
兩者使用 MCU 內部 Pull-up，目前不會停止倒數。緊急停止仍使用板載 PC13/B1。
有源蜂鳴器
不要直接用 PB1 驅動未知電流規格的蜂鳴器。 建議使用 NPN 電晶體作為低側開關：
PB1 → 約 1 kΩ 電阻 → NPN 基極。
NPN 射極 → GND。
NPN 集極 → 有源蜂鳴器負極。
有源蜂鳴器正極 → 符合其額定規格的 3.3 V 或 5 V 電源。
STM32 GND 與蜂鳴器供電 GND 要共地；若蜂鳴器含感性元件，依其規格增設反向保護二極體。
蜂鳴器的電壓及額定電流取決於實際型號；尚未確認型號前，不應假設可直接由 GPIO 供電。
4. STM32CubeMX / STM32CubeIDE 設定
新增 STM32 專案，MCU 選 `STM32G474RETx`，使用 HAL。
RCC / Clock Configuration： HSI 16 MHz；SYSCLK=16 MHz；PLL OFF；AHB/APB1/APB2 Prescaler=1。
GPIO： 依上方腳位表設定 PC8、PB1、PB0、PC7、PC13。
TIM3： Internal Clock，Channel 1 選 `PWM Generation CH1`，Pin PC6 (AF2)：
Prescaler = `15`
Counter Period = `999`
Pulse = `50`
PWM Mode 1、Active High
PWM 頻率 = 16 MHz / [(15 + 1) × (999 + 1)] = 1 kHz。
LPUART1： `Asynchronous`，PA2 TX / PA3 RX (AF12)，Baud Rate=115200，8 data bits，None parity，1 stop bit，No Flow Control。
NVIC： 啟用 `EXTI15_10_IRQn`、`LPUART1_IRQn`；不要破壞 HAL 的 SysTick 時基。
Generate Code 後將 `main_PC13保留_新腳位_中文註解.c` 內容複製到專案 `Core/Src/main.c`。
避免重複中斷函式： 現有 `main.c` 已實作 `EXTI15_10_IRQHandler()` 和 `LPUART1_IRQHandler()`；如果 `Core/Src/stm32g4xx_it.c` 也有這兩個函式，請確保專案中各只保留一份定義。保留 `SysTick_Handler()` 以及其他必要的 HAL 初始化檔案。
編譯、ST-LINK 燒錄，按開發板 RESET 重新開始倒數。
> **ST-LINK VCP：** 若使用 ST-LINK USB 直接連電腦，請確認板子上 PA2/PA3 連至 ST-LINK VCP 的預設焊橋沒有更動。若 PA2/PA3 被 USART2 占用，請先停用 USART2 再改用 LPUART1。PC13/B1 按下使用上升緣中斷。
5. 安裝 Python（PyCharm）
PC 端需要 `pyserial`，安裝名稱 `pyserial`，匯入名稱 `serial`。
在 PyCharm 選 View → Tool Windows → Terminal，或按 Alt + F12。由於你的專案使用 `.venv`，建議在 PowerShell 直接執行下列指令，以避免安裝到錯誤 Python 環境：
```powershell
& "C:\Users\tguo9\PycharmProjects\PythonProject1\.venv\Scripts\python.exe" -m pip install pyserial
```
檢查套件是否成功裝到此環境：
```powershell
& "C:\Users\tguo9\PycharmProjects\PythonProject1\.venv\Scripts\python.exe" -m pip show pyserial
```
若 PyCharm 設定了不同專案路徑，請以執行畫面列出的 `.venv\Scripts\python.exe` 路徑為準。
6. 執行 Python
將 `stm32_alarm_console_LPUART1_中文註解.py` 放到 PyCharm 專案，執行程式。程式會列出 COM 埠，預設 COM3；若實際 ST-LINK 顯示 COM5 等其他編號，請輸入實際號碼。
以下為示意，非實機紀錄：
```text
===== STM32 LED 警報解除模擬 =====
可用 COM 埠：
  COM3：STMicroelectronics STLink Virtual COM Port
請輸入 COM 埠（直接 Enter 使用 COM3）：
已連線至 COM3，速率 115200 8N1
[STM32] START: 30s, PASSWORD VIA UART, B1=EMERGENCY STOP
[STM32] TIME=30s, ERRORS=0/5
密碼> 0000
[STM32] WRONG: 1/5, BLINK=400ms, PWM_MAX=40%
密碼> 1234
[STM32] SUCCESS: DISARMED, ALL OUTPUTS OFF
```
重要： MCU 的 30 秒計時從 MCU 啟動開始，不是 Python 連線後才開始。若已超時，請先讓 Python 連線，再按開發板 RESET 重新開始。
7. 測試表
操作	預期結果
RESET	30 秒倒數、PC8 每 500 ms 切換、PC6 呼吸
輸錯 1～4 次	閃爍間隔依序 400、300、200、100 ms；亮度上限依序 40%、60%、80%、100%
輸入 `1234`（倒數中）	成功解除，LED、蜂鳴器關閉
輸錯第 5 次	失敗鎖定，LED 恆亮、蜂鳴器響 2 秒
倒數 30 秒到	失敗鎖定，LED 恆亮、蜂鳴器響 2 秒
失敗後輸入 `1234`	仍維持鎖定，不能解除
隨時按 PC13 / B1	立即停止並關閉所有輸出
按 PB0 或 PC7	目前無操作效果（僅預留輸入）
8. 程式中的計時位置
`#define COUNTDOWN_MS 30000U`：倒數時間設為 30 秒。
`start_ms = HAL_GetTick();`：記錄開始時刻。
`HAL_GetTick() - start_ms >= COUNTDOWN_MS`：判斷倒數到期。
`last_blink_ms` + `blink_interval`：主 LED 閃爍間隔。
`last_breath_ms`：每 20 ms 更新一次 PWM 占空比。
`buzzer_start_ms` + `BUZZER_MS`：蜂鳴器響 2 秒後自動關閉。
9. 常見問題
`ModuleNotFoundError: No module named 'serial'`：請在 PyCharm 使用的 `.venv` 安裝 `pyserial`；不是安裝名叫 `serial` 的套件。
`WARNING: Package(s) not found: pyserial`：此訊息通常表示你在目前 Python 環境查不到已安裝套件。先執行 `python -m pip install pyserial`，並確認 `python` 是否就是執行 `main.py` 的虛擬環境。
找不到 COM 埠：檢查 ST-LINK USB、Windows 裝置管理員，以及 ST-LINK 驅動程式。
COM 埠無法開啟：可能被 CubeIDE 的 Serial Terminal、PuTTY 或其他程式占用。
有 COM 但沒有倒數訊息：確認 LPUART1 的 PA2/PA3、115200 8N1、焊橋與共同參考地；連線後按 RESET。
PC6 LED 不呼吸：確認 PC6 是 TIM3_CH1 (AF2) 而非普通 GPIO。
編譯 `multiple definition of ...IRQHandler`：確認中斷函式沒有同時在 `main.c` 與 `stm32g4xx_it.c` 定義。
10. 檔案說明與驗證範圍
`main_PC13保留_新腳位_中文註解.c`：STM32 完整 C 原始碼，含繁體中文註解。
`stm32_alarm_console_LPUART1_中文註解.py`：電腦端命令列密碼輸入與狀態接收程式。
`STM32_PC13保留_程式中文左右對照.html`：左邊原始碼、右邊中文說明。
`README.md`：本專題的設定、接線、安裝與測試手冊。
此專題原始碼曾使用模擬 HAL 標頭進行 GCC 語法檢查，Python 曾通過語法檢查；尚未在實際 STM32CubeIDE 與 NUCLEO-G474RE 開發板上完成編譯、燒錄和實機驗證。請務必依你的板子與蜂鳴器實際規格測試。
