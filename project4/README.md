# Lab4 — FreeRTOS Memory Management (heap_2.c)

## Overview

This lab explores FreeRTOS heap_2 memory management. Two features are implemented:
1. `vPrintFreeList` — prints the current free block list via UART
2. Modified `prvInsertBlockIntoFreeList` — merges physically adjacent free blocks to eliminate fragmentation

---

## Hardware

| Item | Spec |
|------|------|
| MCU | STM32F407VGTx |
| Clock | HSI 16MHz → PLL → 50MHz |
| UART | USART2, 115200 baud, 8N1 |
| LED | PD12 (Green), PD14 (Red) |

---

## Task List

| Task | Priority | Stack | Description |
|------|----------|-------|-------------|
| `red_LED_task` | 0 | 100 words | Toggles red LED every 500ms |
| `green_LED_task` | 0 | 130 words | Toggles green LED every 1000ms |
| `task1` | 0 | 50 words | Deletes itself immediately |
| `task2` | 0 | 30 words | Deletes itself immediately |
| `task3` | 0 | 40 words | Deletes itself immediately |
| `print_task` | 0 | 130 words | Calls `vPrintFreeList()` every 1000ms |

`task1`, `task2`, `task3` are designed to exercise the free/merge mechanism by allocating and immediately freeing their memory.

---

## heap_2 Background

heap_2 manages a statically allocated array `ucHeap[configTOTAL_HEAP_SIZE]` as a linked list of free blocks:

```
xStart(size=0) → [small block] → [medium block] → [large block] → xEnd(size=MAX)
```

Each block has a header (`BlockLink_t`) at the start:

```
┌──────────────────────┬─────────────────────┐
│   BlockLink_t header │   data area          │
│  pxNextFreeBlock     │                      │
│  xBlockSize          │                      │
└──────────────────────┴─────────────────────┘
↑                                             ↑
StartAddress                                  StartAddress + xBlockSize (EndAddress)
```

- Free list is sorted in **ascending order by block size**
- `xBlockSize` covers the entire block (header + data area)
- `heapSTRUCT_SIZE` = size of `BlockLink_t` after alignment = **8 bytes**

---

## Part 1 — vPrintFreeList

### Purpose

Traverse the free list and print each free block's information via UART every 1000ms.

### Output Format

```
StartAddress heapSTRUCT_SIZE xBlockSize EndAddress
0x20000968         8            264         0x20000a70
0x200004e8         8            528         0x200006f8
0x20000f50         8           7064         0x20002ae8
configADJUSTED_HEAP_SIZE: 10232 xFreeBytesRemaining: 7856
```

### Implementation

```c
void vPrintFreeList(void)
{
    char data[80];
    BlockLink_t *pxBlock = xStart.pxNextFreeBlock;

    /* Print header */
    sprintf(data, "StartAddress heapSTRUCT_SIZE xBlockSize EndAddress\n\r");
    HAL_UART_Transmit(&huart2, (uint8_t *)data, strlen(data), 0xffff);

    /* Traverse the free list and print each free block */
    while (pxBlock != &xEnd) {
        sprintf(data, "%p         %d           %4d         %p\n\r",
            pxBlock,
            heapSTRUCT_SIZE,
            pxBlock->xBlockSize,
            (uint8_t *)pxBlock + pxBlock->xBlockSize);
        HAL_UART_Transmit(&huart2, (uint8_t *)data, strlen(data), 0xffff);
        pxBlock = pxBlock->pxNextFreeBlock;
    }

    /* Print summary */
    sprintf(data, "configADJUSTED_HEAP_SIZE: %0d xFreeBytesRemaining: %0d\n\r",
        configADJUSTED_HEAP_SIZE, xFreeBytesRemaining);
    HAL_UART_Transmit(&huart2, (uint8_t *)data, strlen(data), 0xffff);
}
```

### Logic

- Start from `xStart.pxNextFreeBlock` and follow `pxNextFreeBlock` until `&xEnd`
- For each block, compute `EndAddress = (uint8_t*)pxBlock + pxBlock->xBlockSize`
- `heapSTRUCT_SIZE` is a fixed constant (8 bytes on STM32F4)

---

## Part 2 — prvInsertBlockIntoFreeList with Merge

### Problem with Original heap_2

Original heap_2 does not merge adjacent free blocks after freeing. This causes fragmentation:

```
Physical memory:
┌────────┬────────┬────────┐
│ free A │  free B(freed)  │ free C │
└────────┴────────┴────────┘

Without merge — free list has 3 separate small blocks:
xStart → A → B → C → xEnd

With merge — free list has 1 large block:
xStart → ABC → xEnd
```

### Merge Strategy

Since the free list is sorted by **size** (not address), finding physically adjacent blocks requires a full traversal. Two cases are checked for each block in the list:

**Case A — free block is physically before pxBlockHolding:**
```
(uint8_t*)freeBlock + freeBlock->xBlockSize == (uint8_t*)pxBlockHolding
→ merged start = freeBlock (move pxBlockHolding back)
→ merged size  = freeBlock->xBlockSize + pxBlockHolding->xBlockSize
```

**Case B — free block is physically after pxBlockHolding:**
```
(uint8_t*)pxBlockHolding + pxBlockHolding->xBlockSize == (uint8_t*)freeBlock
→ merged start = pxBlockHolding (unchanged)
→ merged size  = pxBlockHolding->xBlockSize + freeBlock->xBlockSize
```

### Implementation

```c
#define prvInsertBlockIntoFreeList( pxBlockToInsert )
{
    BlockLink_t *pxIterator;
    size_t xBlockSize;
    BlockLink_t *pxPrevious, *pxBlockHolding = pxBlockToInsert;

    /* Phase 1: Traverse the free list and merge physically adjacent blocks */
    pxPrevious = &xStart;
    pxIterator = xStart.pxNextFreeBlock;
    while( pxIterator != &xEnd )
    {
        if( (uint8_t*)pxIterator + pxIterator->xBlockSize == (uint8_t*)pxBlockHolding )
        {
            /* Case A: pxIterator is just before pxBlockHolding */
            pxIterator->xBlockSize += pxBlockHolding->xBlockSize;
            pxBlockHolding = pxIterator;
            pxPrevious->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
            pxIterator = pxPrevious->pxNextFreeBlock;
        }
        else if( (uint8_t*)pxBlockHolding + pxBlockHolding->xBlockSize == (uint8_t*)pxIterator )
        {
            /* Case B: pxIterator is just after pxBlockHolding */
            pxBlockHolding->xBlockSize += pxIterator->xBlockSize;
            pxPrevious->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
            pxIterator = pxPrevious->pxNextFreeBlock;
        }
        else
        {
            /* Not adjacent, move to next block */
            pxPrevious = pxIterator;
            pxIterator = pxIterator->pxNextFreeBlock;
        }
    }

    /* Phase 2: Insert merged block into free list sorted by size */
    xBlockSize = pxBlockHolding->xBlockSize;
    for( pxIterator = &xStart;
         pxIterator->pxNextFreeBlock->xBlockSize < xBlockSize;
         pxIterator = pxIterator->pxNextFreeBlock ) {}

    pxBlockHolding->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
    pxIterator->pxNextFreeBlock = pxBlockHolding;
}
```

### Key Points

- `pxPrevious` is required to remove a node from the linked list: `pxPrevious->pxNextFreeBlock = pxIterator->pxNextFreeBlock`
- When merging Case A, `pxBlockHolding` must move back to `pxIterator`'s address (the earlier address in memory)
- After removing a merged block, `pxPrevious` does **not** advance — the next block to check is `pxPrevious->pxNextFreeBlock`
- A single traversal handles multiple adjacent blocks (e.g., A+B+C all merged in one pass)

---

## UART Output Example

After all tasks are created and task1/task2/task3 delete themselves:

```
StartAddress heapSTRUCT_SIZE xBlockSize EndAddress
0x20000968         8            264         0x20000a70
0x200004e8         8            528         0x200006f8
0x20000f50         8           7064         0x20002ae8
configADJUSTED_HEAP_SIZE: 10232 xFreeBytesRemaining: 7856
```

Only 3 free blocks remain (instead of more fragmented blocks), confirming that merge is working correctly.

---

## File Structure

```
├── Core/
│   └── Src/
│       └── main.c                        # Task definitions
├── FreeRTOS/
│   └── portable/
│       └── MemMang/
│           └── heap_2.c                  # vPrintFreeList + prvInsertBlockIntoFreeList
└── README_Lab4.md
```
