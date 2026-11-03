# B11413138JOEY
微算機期中考
# 末日機甲戰術控制手環（ NUCLEO-G474RE 期中專題）

## 1. 情境故事：
未來末日降臨我發明了機甲戰術控制手環主要畫面是要測試手環發現遠處有喪屍所以按下按鈕發射導彈然後喪屍近身了按下緊急按鈕來噴射火焰解決近身的喪屍後來地下指揮部打來通話要開會總部直接用總部電腦來控制手環開啟防護模式
## 2. 替身對照：故事裡的模組 → 板上替身
按下按鈕發射導彈->按下按鈕黃燈亮
緊急防護按鈕按下噴射火焰->按鈕按下紅燈呼吸燈
總部遠端控制手環開防護罩->UART電腦輸入綠燈亮
末日機甲能源爐->PLL 170 MHz 板子上石英晶體驅動
## 3. 功能與驗收條件：動作、預期、實測結果
按鈕1按下黃燈亮
按鈕2按下紅燈亮呼吸燈
電腦輸入GREEN ON 綠燈亮
輸入GREEN OFF綠燈滅
## 4. 用到的技術：第 3 ～ 7 週勾選表（同投影片）
【GPIO / 時脈 / 中斷系統】
☑ GPIO 輸出：戰術指示燈：PA11 (黃燈)、PC6 (綠燈) | MODER = 01 | 推挽輸出
☑ GPIO 輸入：機甲狀態檢查按鈕：PA12 按下=狀態良好 | MODER = 00 | 內部上拉
☑ 複用功能 AF：LPUART1 收發與 TIM3 PWM (PC8) 輸出 | MODER = 10 | AFR 選 LPUART1, TIM3
☑ PLL 170 MHz：系統與鮑率的共同基準 | HSI16 + PLL | 系統時脈 170 MHz
☑ 外部中斷 EXTI：呼吸燈模式開關按鈕：PC9 按下=切換紅燈呼吸 | GPIO_MODE_IT_FALLING
☑ NVIC 致能：確保 ISR 能夠順利進入 | EXTI line[9:5] (EXTI9_5_IRQn)
☐ 優先權與巢狀：中斷優先權與巢狀中斷設定 | 預設優先權，未實作巢狀搶佔
☐ 軟體去彈跳：模式切換按鈕軟體去彈跳 | 未額外撰寫，依賴硬體上拉
【UART / Timer / PWM 系統】
☐ UART 傳送：傳送回報功能 | 未實作 (僅單向接收電腦指令)
☑ UART 接收 + 緩衝解析：接收指令控制綠燈 (LPUART1)：指令 GREEN ON、GREEN OFF | 輪詢接收，遇換行 strcmp 解析
☐ Timer 定時中斷：定時更新機甲數據 | 未啟用定時器中斷，改用 HAL_GetTick()
☑ PWM 輸出：呼吸燈控制：控制 PC8 紅燈實現漸亮漸暗呼吸效果 | TIM3 PSC = 169, ARR = 999
☐ 輸入捕捉：訊號頻率與脈寬量測 | 未用到
## 5. 腳位表：用到的每支腳與用途
腳位我想要用PA12外接一個按鈕1PULL UP按下然後PA11輸出黃色LED亮沒按就是滅然後PC9也是一個按鈕2PULL UP是有中斷的然後按下去PC8紅色LED燈做最後我要用UART不要用環形的在電腦輸入GREEN ON讓PC6輸出的綠色LED亮輸入GREEN OFF讓他滅Baud Rate115200WORD LENGTH8BITS PARITY NONE STOP BITS 1

## 6. 建置與執行：開發環境、燒錄步驟、終端機設定、命令
呼吸燈PCS是169Counter Period是999
Baud Rate115200WORD LENGTH8BITS PARITY NONE STOP BITS 1
## 7. 程式結構：哪個檔做什麼、 ISR 與主迴圈分工
 STM32 專案的程式碼檔案結構與ISR（中斷服務常式）與主迴圈 (while(1)) 分工的詳細解析
 一、 程式專案結構（各檔案負責什麼？）在 STM32CubeIDE 產生的專案中，程式碼主要分佈在 Core/Src（原始碼）與 Core/Inc（標頭檔）資料夾下：main.c（核心主程式）職責：整個專案的心臟。包含系統時脈設定 (SystemClock_Config)、各周邊初始化呼叫 (MX_GPIO_Init, MX_TIM3_Init, MX_LPUART1_Init)。關鍵內容：放置 while(1) 主迴圈，執行所有的非阻塞背景任務（按鈕輪詢、PWM 時間差狀態機、UART 指令解析）。放置中斷回呼函式（如 HAL_GPIO_EXTI_Callback）。main.h（全域標頭檔）職責：定義專案中會用到的共用巨集、引入 HAL 函式庫標頭檔，並宣告全域變數或硬體控制代碼（如 hlpuart1, htim3）。stm32g4xx_it.c（中斷服務常式檔）職責：處理所有硬體中斷的底層進入點（例如外部中斷向量 EXTI9_5_IRQHandler）。當硬體事件發生時，CPU 會先跳到這裡，再由它去呼叫 HAL 庫的中斷處理函式。stm32g4xx_hal_msp.c（低層硬體支援檔）職責：專門負責周邊硬體的底層初始化（MCU Support Package），例如開啟 GPIO 時脈 (__HAL_RCC_GPIOA_CLK_ENABLE())、設定接腳的複用功能（Alternate Function）等。
 二、 ISR 與主迴圈 (while(1)) 的分工為了滿足「嚴禁阻塞」的要求，本專案將工作完美拆分為「事件驅動（ISR）」與「非阻塞輪詢／狀態機（Main Loop）」：程式碼片段graph TD
    subgraph ISR [中斷服務常式 ISR - 高優先級、瞬間響應]
        A[PC9 按鈕按下觸發 EXTI9_5_IRQHandler] --> B[進入 HAL_GPIO_EXTI_Callback<br>反轉 breathing_enabled 旗標]
    end

    subgraph MainLoop [while 1 主迴圈 - 非阻塞背景任務]
        C[任務一：輪詢 PA12 按鈕<br>控制 PA11 黃色 LED] --> D[任務二：檢查 HAL_GetTick<br>非阻塞更新 PC8 呼吸燈 PWM]
        D --> E[任務三：非阻塞讀取 UART1 位元組<br>解析指令控制 PC6 綠色 LED]
        E --> C
    end
1. ISR（中斷服務常式）的分工特性：由硬體直接觸發，擁有最高優先級，必須「快進快出」。本專案實作：負責 PC9 外部中斷（當使用者按下 PC9 按鈕時）。   觸發後會在 HAL_GPIO_EXTI_Callback() 中執行：   Cbreathing_enabled = !breathing_enabled; // 瞬間切換呼吸燈開關狀態
絕不做的事：絕對不在 ISR 內使用 HAL_Delay()、做複雜運算或等待 UART 傳輸，以免卡死系統。2. 主迴圈 (while(1)) 的分工特性：持續不斷地循環執行，利用非阻塞技巧同時處理多個任務，互相不干擾。本專案實作：任務一：PA12 按鈕輪詢與黃色 LED 控制   在每次迴圈直接讀取 PA12 電平，按下即讓 PA11 黃色 LED 亮，放開即滅。   任務二：PWM 呼吸燈狀態機（時間差計算）   利用 HAL_GetTick() 搭配 last_tick 記錄時間差。當達到設定間隔（例如 2ms 或 10ms）時，根據 breathing_enabled 旗標自動遞增或遞減 pulse_value，平滑調整 PC8 的紅燈亮度。   任務三：UART 序列埠非阻塞字串解析   每次迴圈利用極短逾時（1ms）的 HAL_UART_Receive 嘗試讀取 1 個位元組。當收到換行符號（\r 或 \n）時，立刻比對字串，若為 "GREEN ON" 或 "GREEN OFF" 則切換 PC6 綠色 LED 狀態。   
## 8. 實作照片：每張附一行圖說

![硬體全貌與終端機測試](螢幕擷取畫面%202026-11-03%20152122.png)
*圖一：NUCLEO-G474RE 開發板與麵包板配線全貌，畫面顯示透過序列埠終端機成功發送 GREEN ON 指令控制綠燈[span_2](start_span)[span_2](end_span)[span_3](start_span)[span_3](end_span)。*

![按鈕與 LED 實體配線特寫](螢幕擷取畫面%202026-11-03%20151938.png)
*圖二：麵包板上的按鈕與 LED 
按鈕1按下黃燈亮
按鈕2按下紅燈亮呼吸燈
實體電路特寫[span_4](start_span)[span_4](end_span)。*

## 9. AI 使用說明：哪段請 AI 、你怎麼查核
1. 哪段請 AI 協助？
非阻塞架構與 UART 解析：請 AI 協助設計 while(1) 主迴圈內的非阻塞時間差機制（使用 HAL_GetTick() 取代 HAL_Delay()），以及利用極短逾時的 HAL_UART_Receive 進行單一字元接收與 strcmp 字串解析（GREEN ON / GREEN OFF）。
邏輯除錯與型態修正：在實作 PWM 呼吸燈時，紅燈亮起後卡在最亮無法遞減。經 AI 診斷發現是由於 pulse_value 宣告為無號數 (uint32_t) 導致數值下溢（Underflow），並協助將其修正為有號整數 (int32_t)。
2. 你怎麼查核？
編譯驗證（Build Check）：將 AI 提供的程式碼與修改建議貼入 STM32CubeIDE 進行 Incremental Build，確認編譯零錯誤（0 errors, 0 warnings），確保語法與變數型態完全正確。
上板實測驗證（Hardware Verification）：
按鈕一（PA12）：實際按下按鈕，確認黃色 LED 能即時點亮，放開即熄滅[cite: 1]。
按鈕二（PC9 外部中斷）：按下按鈕觸發 EXTI，確認紅燈（PC8）能夠平滑地進行漸亮、漸暗的循環呼吸[cite: 1]。
UART 遠端指令：透過電腦序列埠輸入 GREEN ON 與 GREEN OFF，實際觀察 PC6 綠色 LED 的開關狀態是否與指令完全相符[cite: 1, 4]。
## 10. 已知限制：哪些用替身、換真貨要改哪裡
黃燈改導彈發射器
紅燈改噴射火焰
綠燈改防護罩
電腦改總指揮中心
