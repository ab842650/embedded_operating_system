# Lab5 - WAV Audio Player on STM32F407

## Hardware Setup

**Platform:** STM32F407VG Discovery Board

### SD Card Module (SPI2)
| SD Module | STM32 Pin |
|-----------|-----------|
| CS        | PB1       |
| SCK       | PB13      |
| MOSI      | PB15      |
| MISO      | PB14      |
| VCC       | 5V        |
| GND       | GND       |

### CS43L22 Audio Codec (on-board)
| Interface | Pins |
|-----------|------|
| I2C1 (control) | SCL=PB6, SDA=PB9 |
| I2S3 (audio)   | WS=PA4, CK=PC10, SD=PC12, MCK=PC7 |
| RESET     | PD4 (active HIGH) |

### Button
- User button: PA0 (on-board)

---

## Features
- Plays WAV files (PCM, 44100Hz, stereo, 16-bit) from SD card sequentially
- Button control with single / double / long press detection
- Operation log written to `log.txt` on SD card

### Button Operations
| Mode | Single Press | Double Press | Long Press |
|------|-------------|--------------|------------|
| PLAYBACK_CONTROL | Play / Pause | Enter TRACK_SWITCHING | Enter VOLUME_ADJUST |
| TRACK_SWITCHING | Previous track | Next track | Back to PLAYBACK_CONTROL |
| VOLUME_ADJUST | Volume down | Volume up | Back to PLAYBACK_CONTROL |

---

## Key Issues & Solutions

### 1. SD Card Mount Failure (FR_NOT_READY)
SD card module requires **5V input** (has onboard regulator), not 3.3V directly. Also added retry loop in AudioPlayerTask since the card needs time to power up.

### 2. CS43L22 Initialization Failure
Two causes:
- I2C1 SDA was mapped to wrong pin (PB7 → fixed to **PB9**)
- RESET pin PD4 was floating → added GPIO output config with **PD4 = HIGH** in `HAL_I2C_MspInit`

### 3. Audio Playing at Half Speed (2x slower)
Root cause: SD card SPI driver (`rcvr_spi_multi`) was reading **byte-by-byte** using individual `HAL_SPI_TransmitReceive` calls. For a 2048-byte read, this meant 2048 HAL calls with overhead, taking ~10ms — exceeding the DMA half-buffer period of 11.6ms. The DMA replayed old buffer data in circular mode, stretching audio to 2x duration.

**Fix:** Replaced byte-by-byte loop with a single bulk `HAL_SPI_TransmitReceive` call per 512-byte block, reducing read time to ~1ms.

### 4. Playback Never Stopping (Audio Stuck at End)
`BufferCtl.fptr >= WaveFormat.FileSize` never triggered because `fptr` counts audio data bytes while `FileSize` includes RIFF header overhead (36 bytes difference). Added EOF detection by checking if `f_read` returns fewer bytes than requested.
