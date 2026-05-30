# Lab2 — FreeRTOS Task Monitor

## 專案概述

本專案在 STM32F407VG 開發板上執行 FreeRTOS，實作一個 Task Monitor，每秒透過 UART2 輸出所有 task 的即時狀態，用以觀察 FreeRTOS 排程器內部的 task 管理機制。

---

## 硬體環境

| 項目 | 規格 |
|------|------|
| MCU | STM32F407VGTx |
| 時脈 | HSI 16MHz → PLL → 50MHz |
| UART | USART2，115200 baud，8N1 |
| LED | PD12（Green），PD14（Red） |

---

## Task 清單

| Task 名稱 | 優先級 | Stack 大小 | 功能 |
|-----------|--------|------------|------|
| `Red_LED` | 1 | 128 words | Toggle 紅燈，初始間隔 800ms，每次 +1ms |
| `Green_LED` | 1 | 128 words | Toggle 綠燈，初始間隔 1000ms，每次 +2ms |
| `Delay_App` | 14 | 128 words | 第一次 delay 1000ms，之後 delay 0xFFFFFFFF ticks（近乎永久） |
| `TaskMonitor` | 3 | 256 words | 每 1000ms 呼叫一次 Taskmonitor()，輸出所有 task 狀態 |

---

## UART 輸出格式

透過序列埠（115200 baud）每秒輸出一次，格式如下：

```
|Name       |Priority(Base/actual) |pxStack    |pxTopOfStack |State   |
TaskMonitor 3 /3                0x20008000  0x20007FA0    Ready
Red_LED     1 /1                0x20007800  0x20007750    Blocked
Green_LED   1 /1                0x20007400  0x20007350    Blocked
Delay_App   14/14               0x20007000  0x20006FB0    Overflow
```

### 欄位說明

| 欄位 | 說明 |
|------|------|
| Name | Task 名稱（最多 configMAX_TASK_NAME_LEN 字元） |
| Priority(Base/actual) | 基礎優先級 / 實際優先級（有 mutex 繼承時兩者可能不同） |
| pxStack | Stack 底部起始位址（heap 分配時的起點） |
| pxTopOfStack | Stack 頂端目前位址（context switch 後更新） |
| State | `Ready` / `Blocked` / `Overflow` |

### State 判斷依據

- **Ready**：task 在 `pxReadyTasksLists[]` 中（等待或正在執行）
- **Blocked**：task 在 `pxDelayedTaskList` 中（呼叫 vTaskDelay 等待中）
- **Overflow**：task 在 `pxOverflowDelayedTaskList` 中（delay 時間超過 tick counter 上限，發生溢位）

---

## 資料存取機制

### 1. FreeRTOS 內部資料結構

FreeRTOS 在 `tasks.c` 中以 file-scope 靜態變數維護 task 狀態：

```c
// tasks.c
static List_t pxReadyTasksLists[configMAX_PRIORITIES]; // 各優先級的 Ready list
static List_t *volatile pxDelayedTaskList;             // Blocked task list
static List_t *volatile pxOverflowDelayedTaskList;     // Overflow Blocked list
```

每個 task 的資料存放在 **TCB（Task Control Block）** 結構中：

```c
typedef struct tskTaskControlBlock {
    volatile StackType_t *pxTopOfStack;          // stack 頂端目前位址（必須是第一個成員）
    ListItem_t            xStateListItem;         // 掛在 Ready/Blocked list 的節點
    UBaseType_t           uxPriority;             // 實際優先級
    StackType_t          *pxStack;               // stack 底部起始位址
    char    pcTaskName[configMAX_TASK_NAME_LEN]; // task 名稱
    UBaseType_t           uxBasePriority;         // 基礎優先級
} TCB_t;
```

### 2. 為何能直接存取私有變數

`Taskmonitor()` 函式直接附加在 `tasks.c` 尾端，因此與上述所有靜態變數在同一個 translation unit 內，可以直接存取，不需要額外的 API 介面。

```
tasks.c
├── static List_t pxReadyTasksLists[]    ← FreeRTOS 私有資料
├── static List_t *pxDelayedTaskList     ← FreeRTOS 私有資料
├── ...其他 FreeRTOS 函式...
└── void Taskmonitor()                   ← Lab2 新增，直接存取上方私有變數
```

### 3. 遍歷 Linked List

每條 List 是雙向環狀 linked list，透過巨集依序取出每個 TCB：

```c
pxListEnd  = listGET_END_MARKER(&pxReadyTasksLists[xPriority]); // 取尾端哨兵
pxListItem = listGET_HEAD_ENTRY(&pxReadyTasksLists[xPriority]);  // 取第一個節點

while (pxListItem != pxListEnd) {
    pxTCB = (TCB_t *) listGET_LIST_ITEM_OWNER(pxListItem);      // 節點 → TCB 指標
    // 讀取 pxTCB 欄位...
    pxListItem = listGET_NEXT(pxListItem);                        // 前往下一個節點
}
```

`listGET_LIST_ITEM_OWNER` 讀取 `ListItem_t.pvOwner`，此 pointer 在 task 建立時即設定為指向該 task 的 TCB。

### 4. 暫停 Scheduler 防止競態條件

遍歷期間暫停排程器，確保 list 結構不會因 context switch 而被同時修改：

```c
vTaskSuspendAll();   // 暫停排程器（不觸發 context switch）
// ... 遍歷三條 list 並輸出 ...
xTaskResumeAll();    // 恢復排程器
```

### 5. 格式化並透過 UART 輸出

```c
sprintf(Monitor_data, "%-11s %-2u/%-2u               0x%08x  0x%08x    Ready\n\r",
    pxTCB->pcTaskName,
    (unsigned int)pxTCB->uxBasePriority,
    (unsigned int)pxTCB->uxPriority,
    (unsigned int)pxTCB->pxStack,
    (unsigned int)pxTCB->pxTopOfStack);

HAL_UART_Transmit(&huart2, (uint8_t *)Monitor_data, strlen(Monitor_data), 0xffff);
```

使用 blocking 模式傳送（timeout = 0xffff ticks），確保資料完整送出後再繼續。

---

## 完整資料流

```
xTaskCreate() 建立 Task
    └─ 在 heap 分配 TCB + Stack
    └─ 將 TCB 掛入 pxReadyTasksLists[priority]

TaskMonitor_App（每 1000ms 執行一次）
    └─ 呼叫 Taskmonitor()
          ├─ vTaskSuspendAll()                  鎖定
          ├─ 輸出表頭
          ├─ 遍歷 pxReadyTasksLists[]           → 輸出 Ready 狀態的 task
          ├─ 遍歷 pxDelayedTaskList             → 輸出 Blocked 狀態的 task
          ├─ 遍歷 pxOverflowDelayedTaskList     → 輸出 Overflow 狀態的 task
          │     每個節點：listGET_LIST_ITEM_OWNER → TCB*
          │               讀取欄位 → sprintf → HAL_UART_Transmit
          └─ xTaskResumeAll()                   解鎖
```

---

## 觀察重點

- **Delay_App** 在第二次 delay 後（`0xFFFFFFFF` ticks）會出現在 `Overflow` list，因為其 wake-up tick 超過了 tick counter 的最大值。
- **Red_LED / Green_LED** 的 delay 時間隨時間遞增，可觀察到 LED 閃爍頻率越來越慢。
- **TaskMonitor** 本身在輸出當下處於 `Ready` 狀態（正在執行），不會出現在 Blocked list 中。

## 檔案結構

```
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       └── main.c          # 主程式、Task 定義
├── FreeRTOS/
│   ├── tasks.c             # Taskmonitor() 實作於此檔尾端
│   └── include/
│       └── task.h
├── Drivers/                # STM32 HAL Drivers
└── README_Lab2.md
```
