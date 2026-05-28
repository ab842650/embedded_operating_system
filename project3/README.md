# STM32F407 Lab3 - Motion Detection with FreeRTOS

## 簡介

使用 STM32F407VG 搭配 LIS3DSH 加速度感測器，實作搖動偵測與 FreeRTOS Deferred Interrupt Handling。

## 硬體環境

| 元件 | 說明 |
|------|------|
| MCU | STM32F407VGTx |
| Motion Sensor | ST MEMS LIS3DSH（SPI 介面） |
| IDE | STM32CubeIDE |
| RTOS | FreeRTOS |

### Pin 腳對應

| 功能 | Pin |
|------|-----|
| SPI1 (SCK/MISO/MOSI) | PA5 / PA6 / PA7 |
| Sensor CS | PE3 |
| Sensor INT | PE0 |
| Green LED | PD12 |
| Orange LED | PD13 |
| Red LED | PD14 |

## 功能說明

### 運作流程

```
搖板子
  → Sensor 產生中斷 (PE0)
    → EXTI0_IRQHandler
      → HAL_GPIO_EXTI_Callback (ISR)
        → Red LED toggle
        → Give Semaphore
          → Handler_Task 醒來
            → Orange LED 閃爍 5 次
            → 讀 OUTS1 (reset sensor interrupt)

同時持續執行：
  LED_Green_Task → Green LED 每 500ms 閃爍
```

### LED 行為

| LED | 行為 |
|-----|------|
| Green (PD12) | 程式啟動後持續 500ms 閃爍 |
| Red (PD14) | 每次搖動切換一次狀態（toggle） |
| Orange (PD13) | 每次搖動後閃爍 5 次（共 2 秒），期間不可再觸發 |

## 程式架構

### FreeRTOS Tasks

| Task | Priority | 說明 |
|------|----------|------|
| Handler_Task | 2 (高) | 等待 semaphore，負責 Orange LED 閃爍與 reset sensor |
| LED_Green_Task | 1 (低) | 持續閃爍 Green LED |

### Sensor 設定（LIS3DSH）

| Register | 值 | 說明 |
|----------|----|------|
| CTRL_REG4 | 0x67 | 100Hz，X/Y/Z 軸全啟用 |
| CTRL_REG3 | 0x48 | Interrupt active high，INT1 啟用 |
| THRS1_1 | 0x40 | 閾值約 1g，超過才觸發 |
| MASK1_B/A | 0x3F | 六個方向（±X ±Y ±Z）全部偵測 |
| SETT1 | 0x01 | 狀態機設定（SITR=1） |
| ST1_1 | 0x05 | GNTH1：超過閾值進下一步 |
| ST1_2 | 0x11 | CONT：觸發 interrupt 並重置 |
| CTRL_REG1 | 0x01 | 啟用狀態機 SM1 |

### Deferred Interrupt Handling

ISR 只做最少的事（toggle Red LED、give semaphore），將耗時的工作（Orange LED 閃爍）交給 Handler_Task 處理。

```
ISR                    Handler_Task
 |                          |
 |--- xSemaphoreGiveFromISR --> xSemaphoreTake (醒來)
 |                          |
 |                     Orange LED 閃 5 次
 |                          |
 |                     讀 OUTS1 (reset sensor)
 |                          |
 |                     xSemaphoreTake (等下次)
```

### 為什麼 Orange 閃爍期間無法再觸發？

Handler_Task 閃完 5 次後才讀 `OUTS1 (0x5F)` register。在此之前 sensor 的 interrupt 線維持 asserted，即使再搖板子也不會產生新的邊緣觸發，因此自然鎖住了再觸發的可能。

## 檔案結構

```
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       ├── main.c          # 主程式、Tasks、ISR Callback
│       └── stm32f4xx_it.c  # 中斷向量（EXTI0_IRQHandler）
├── FreeRTOS/               # FreeRTOS kernel
├── Drivers/                # STM32 HAL Drivers
└── README.md
```

## NVIC 設定注意事項

Sensor interrupt priority 設為 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`，確保在 ISR 中可以安全呼叫 FreeRTOS API（`xSemaphoreGiveFromISR`）。

`SVC_Handler`、`PendSV_Handler`、`SysTick_Handler` 由 FreeRTOS `port.c` 定義，不可在 `stm32f4xx_it.c` 重複定義。
