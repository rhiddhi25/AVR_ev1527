# EV1527 Library - API Reference

This library provides a complete decoder implementation for the EV1527 encoding protocol commonly used in 433MHz/315MHz RF remote controls and wireless transmitters. The decoder uses Timer1 and external interrupt (INT0) to capture and decode pulse widths, extracting 24-bit data frames consisting of a 20-bit unique address and 4-bit key code.

---

## Protocol Overview

### EV1527 Encoding Specifications

The EV1527 protocol uses pulse width modulation to encode binary data:

**Data Format:**
- **Total bits:** 24 bits per frame
- **Address:** 20 bits (unique transmitter ID: 0 to 1,048,575)
- **Key/Button:** 4 bits (button identifier: 0 to 15)

**Bit Encoding:**
- **Logic '0':** Short HIGH (1×T) + Long LOW (3×T)
  - HIGH: ~300µs, LOW: ~900µs
- **Logic '1':** Long HIGH (3×T) + Short LOW (1×T)
  - HIGH: ~900µs, LOW: ~300µs
- **Base period (T):** ~300-350µs typical

**Frame Structure:**
```
[Preamble] [20-bit Address] [4-bit Key] [Sync]
```

**Preamble Pattern:**
- Very long LOW pulse: ~10ms (31×T)
- Short HIGH pulse: ~320µs (1×T)
- Preamble ratio: LOW is 25-40× longer than HIGH

### Timing Analysis (16MHz AVR)

With Timer1 prescaler /8:
- **Timer resolution:** 0.5µs per tick (16MHz / 8 = 2MHz)
- **Logic '0':** 600 ticks (HIGH) + 1800 ticks (LOW) = 2400 ticks total
- **Logic '1':** 1800 ticks (HIGH) + 600 ticks (LOW) = 2400 ticks total
- **Preamble:** 20000 ticks (LOW) + 640 ticks (HIGH)

---

## Hardware Requirements

### Required Components

1. **RF Receiver Module:**
   - 433MHz or 315MHz superheterodyne receiver
   - Common models: RXB6, RXB8, MX-RM-5V

2. **Microcontroller:**
   - AVR microcontroller with Timer1 and INT0
   - Tested on: ATmega328P
   - Clock frequency: 16MHz recommended

3. **External Interrupt Pin:**
   - INT0 pin (typically PD2 on ATmega328P)
   - Connected to RF receiver data output

### Pin Connections

```
RF Receiver Module          AVR Microcontroller
─────────────────          ──────────────────
VCC    ──────────────────> 5V
GND    ──────────────────> GND
DATA   ──────────────────> INT0 (PD2)
```

### Optional: Debugging Output

The library includes debug output on PC0 for LogicAnalyzer verification:
```
PC0    ──────────────────> LogicAnalyzer probe
```

> [!NOTE]
> The debug output generates a brief pulse on PC0 during each interrupt call, allowing you to verify timing and interrupt frequency.

---

## API Functions

### Initialization

#### `void ev1527_Init(void)`

**Description:**  
Initializes the EV1527 decoder hardware (Timer1 and INT0 external interrupt). This function **must be called first** before attempting to decode RF signals.

**Operation:**
- Configures Timer1 in normal mode with prescaler /8
- Sets INT0 to trigger on rising edge
- Enables INT0 external interrupt
- Initializes internal state machine variables

**Parameters:**  
None

**Returns:**  
None

**Timing Configuration:**
- Timer1 prescaler: /8 (0.5µs resolution at 16MHz)
- Timer1 mode: Normal counting (counts from 0 to 0xFFFF)
- INT0 trigger: Rising edge initially (automatically switched during operation)

**Example:**
```c
#include "aKaReZa.h"
#include "ev1527.h"

int main(void) 
{
    ev1527_Init(); // Initialize decoder
    sei();         // Enable global interrupts
    
    while(1)
    {
        // Main loop
    }
}
```

> [!IMPORTANT]
> - Global interrupts must be enabled using `sei()` after calling `ev1527_Init()`
> - Ensure INT0 pin is correctly connected to RF receiver data output
> - Verify F_CPU is correctly defined for accurate timing

---

### Deinitialization

#### `void ev1527_deInit(void)`

**Description:**  
Disables the EV1527 decoder and releases hardware resources. Use this function to stop decoding operations and save power when RF reception is not needed.

**Operation:**
- Disables INT0 external interrupt
- Stops Timer1 by clearing clock source
- Clears edge detection configuration
- Resets timer mode to normal

**Parameters:**  
None

**Returns:**  
None

**Use Cases:**
- Power saving in battery-operated devices
- Preventing re-triggering after successful code reception
- Temporarily disabling decoder during other operations
- Before reconfiguring hardware for different purposes

**Example:**
```c
// After successful code reception
if (ev1527_Data.Bits.Detect) 
{
    // Process received data
    processCode(ev1527_Data.Bits.Address, ev1527_Data.Bits.Keys);
    
    // Disable decoder to prevent re-triggering
    ev1527_deInit();
    
    // Optional: Re-enable after processing
    _delay_ms(100);
    ev1527_Data.Bits.Detect = 0; // Clear detection flag
    ev1527_Init();               // Re-enable decoder
}
```

> [!NOTE]
> The decoder automatically calls `ev1527_deInit()` after successfully receiving a complete 24-bit frame to prevent immediate re-triggering on the same transmission.

---

## Data Structure

### `ev1527_T` Union

The decoded RF data is stored in a global volatile union structure that provides both raw 32-bit access and bit-field access to individual components.

**Declaration:**
```c
volatile ev1527_T ev1527_Data;
```

**Structure Definition:**
```c
typedef union 
{
    uint32_t rawValue;           // Direct 32-bit access
    
    struct
    {
        uint32_t Address : 20;   // 20-bit transmitter address
        uint32_t Keys    : 4;    // 4-bit key/button code
        uint32_t Detect  : 1;    // Detection flag
        uint32_t Reserve : 7;    // Reserved bits
    } Bits;
} ev1527_T;
```

### Field Descriptions

#### `rawValue` (uint32_t)
- Direct access to entire 32-bit decoded value
- Useful for storage, comparison, or transmission
- Contains all 24 data bits plus status flags

**Example:**
```c
ev1527_Data.rawValue;
```

#### `Address` (20 bits)
- Unique transmitter/remote control identifier
- Range: 0 to 1,048,575 (0x00000 to 0xFFFFF)
- Same for all buttons on a single remote
- Used to distinguish between different transmitters

**Example:**
```c
if (ev1527_Data.Bits.Address == 0x12345) {
    // Authorized remote detected
}
```

#### `Keys` (4 bits)
- Button/key identifier on the remote control
- Range: 0 to 15 (0x0 to 0xF)
- Different for each button on the same remote
- Used to identify which button was pressed

**Example:**
```c
switch(ev1527_Data.Bits.Keys) {
    case 0: // Button A
        executeActionA();
        break;
    case 1: // Button B
        executeActionB();
        break;
    // Additional buttons...
}
```

#### `Detect` (1 bit)
- Detection status flag
- **1 (true):** Valid code successfully received
- **0 (false):** No detection or flag cleared
- Automatically set by decoder upon successful reception
- Must be manually cleared by application after processing

**Example:**
```c
if (ev1527_Data.Bits.Detect) {
    // Valid code received
    processRemoteCommand();
    ev1527_Data.Bits.Detect = 0; // Clear flag
}
```

#### `Reserve` (7 bits)
- Reserved for future use or alignment
- Not used by current implementation
- Can be utilized for custom flags or extensions


### Memory and Volatility

**Why Volatile:**
The `ev1527_Data` structure is declared as `volatile` because it is modified by the interrupt service routine (ISR) and accessed by the main program. This prevents compiler optimizations that might cache the value and miss updates from the ISR.

**Memory Usage:**
- Total size: 4 bytes (32 bits)
- Located in SRAM (global variable)
- Single instance shared between ISR and main code

> [!WARNING]
> - Always check `Detect` flag before accessing `Address` and `Keys`
> - Clear `Detect` flag after processing to avoid re-processing same data
> - The decoder overwrites data on each new reception
> - Consider copying data immediately if needed for later processing

---

## Complete Examples

### Basic RF Remote Control

```c
#include "aKaReZa.h"
#include "ev1527.h"

#define LED_PORT PORTB
#define LED_PIN  0

int main(void) 
{
    // Configure LED pin as output
    GPIO_Config_OUTPUT(DDRB, LED_PIN);
    
    // Initialize EV1527 decoder
    ev1527_Init();
    
    // Enable global interrupts
    sei();
    
    while(1)
    {
        // Check if valid code received
        if (ev1527_Data.Bits.Detect)
        {
            // Check for specific remote address
            if (ev1527_Data.Bits.Address == 0x12345)
            {
                // Check which button was pressed
                switch (ev1527_Data.Bits.Keys)
                {
                    case 0: // Button A - LED ON
                        bitSet(LED_PORT, LED_PIN);
                        break;
                        
                    case 1: // Button B - LED OFF
                        bitClear(LED_PORT, LED_PIN);
                        break;
                        
                    case 2: // Button C - LED Toggle
                        bitToggle(LED_PORT, LED_PIN);
                        break;
                        
                    default:
                        // Ignore other buttons
                        break;
                }
            }
            
            // Clear detection flag
            ev1527_Data.Bits.Detect = 0;
            
            // Re-enable decoder for next transmission
            ev1527_Init();
        }
    }
}
```

---

## Troubleshooting Guide

### No Reception Issues

**Symptom:** `Detect` flag never sets to 1

**Possible Causes and Solutions:**

1. **Global interrupts not enabled:**
   ```c
   sei(); // Add after ev1527_Init()
   ```

2. **RF receiver not powered:**
   - Check 5V supply to receiver
   - Verify ground connection
   - Measure voltage at receiver VCC pin

3. **INT0 pin not connected:**
   - Verify physical connection to PD2 (ATmega328P)
   - Check for loose wires
   - Test continuity with multimeter

4. **Receiver not detecting signal:**
   - Move transmitter closer
   - Check transmitter battery
   - Verify antenna on receiver
   - Test with LED: `LED = !RX_DATA` (should blink during transmission)

5. **Wrong clock frequency:**
   - Verify F_CPU matches actual crystal/oscillator
   - Common issue: F_CPU=16000000 but using 8MHz clock

---

### Partial or Corrupt Data

**Symptom:** `Detect` flag sets but wrong address/key values

**Possible Causes and Solutions:**

1. **Weak RF signal:**
   - Move transmitter closer to receiver
   - Replace transmitter battery
   - Improve antenna (use 17.3cm wire for 433MHz)
   - Change receiver location (avoid metal enclosures)

2. **RF interference:**
   - Test in different location
   - Turn off other 433MHz devices
   - Add shielding to receiver
   - Use shielded cable for data line

3. **Timing calibration:**
   - Verify F_CPU definition
   - Check crystal accuracy
   - Adjust timing thresholds if needed

4. **Multiple transmitters:**
   - Ensure only one transmitter active
   - Collision can cause corrupt data
   - Add delay between transmissions

---

### False Detections

**Symptom:** Random detections without transmitter activity

**Possible Causes and Solutions:**

1. **RF noise:**
   - 433MHz band is crowded
   - Tighten pulse validation thresholds:
   ```c
   // In ev1527.h, adjust these values
   #define HPL_min 500  // Increase from 450
   #define HPL_Max 8000 // Decrease from 8500
   ```

2. **Electrical noise:**
   - Add decoupling capacitors near receiver (0.1µF, 10µF)
   - Use separate ground for RF module if possible
   - Add ferrite bead on data line

3. **Receiver sensitivity too high:**
   - Some receivers have adjustable sensitivity
   - Reduce if causing false triggers

---

### Decoder Stops Working

**Symptom:** Works initially, then stops detecting

**Possible Causes and Solutions:**

1. **Decoder auto-disabled:**
   - Library calls `ev1527_deInit()` after detection
   - Must re-enable manually:
   ```c
   if (ev1527_Data.Bits.Detect) {
       processData();
       ev1527_Data.Bits.Detect = 0;
       ev1527_Init(); // Re-enable decoder
   }
   ```

2. **Detect flag not cleared:**
   ```c
   // Must clear flag after processing
   ev1527_Data.Bits.Detect = 0;
   ```

3. **State machine stuck:**
   - Power cycle microcontroller
   - Add watchdog timer for auto-recovery
   
---

## FAQ (Frequently Asked Questions)

**Q: Can I use multiple receivers with this library?**  
A: Yes, but you need to modify the code to handle multiple interrupt pins. The current implementation uses INT0 only.

**Q: Can I decode multiple transmitters simultaneously?**  
A: No, the library decodes one transmission at a time. Simultaneous transmissions will cause collision and data corruption.

**Q: How do I find the address of my remote?**  
A: Use the UART debug example to display received addresses. Press remote button and note the address value.

**Q: Can I change the timer from Timer1 to Timer0?**  
A: Yes, but you need to modify the timer initialization and access macros in both .h and .c files. Not recommended unless Timer1 is unavailable.

**Q: My receiver has 4 pins (VCC, GND, DATA, DATA). Which DATA pin do I use?**  
A: Both DATA pins output the same signal. Use either one, the other is redundant.

**Q: Can I use this for car key fobs?**  
A: Most modern car keys use proprietary encrypted protocols (not EV1527). This library won't work with car keys.

---

## Reference Tables

### Timer1 Prescaler Options (16MHz)

| Prescaler | Clock Frequency | Resolution | Max Time |
|-----------|-----------------|------------|----------|
| /1 | 16 MHz | 0.0625 µs | 4.096 ms |
| /8 | 2 MHz | 0.5 µs | 32.768 ms |
| /64 | 250 kHz | 4 µs | 262.144 ms |
| /256 | 62.5 kHz | 16 µs | 1.048 s |
| /1024 | 15.625 kHz | 64 µs | 4.194 s |

**Current setting:** /8 (0.5µs, optimal for EV1527 timing)

### Pulse Width Reference (16MHz, /8 prescaler)

| Description | Time (µs) | Timer Ticks |
|-------------|-----------|-------------|
| Short pulse (1×T) | 300 | 600 |
| Long pulse (3×T) | 900 | 1800 |
| Logic '0' total | 1200 | 2400 |
| Logic '1' total | 1200 | 2400 |
| Preamble LOW | 10000 | 20000 |
| Preamble HIGH | 320 | 640 |
| Min valid pulse | 225 | 450 |
| Max valid pulse | 4250 | 8500 |

### Common RF Modules

| Model | Type | Sensitivity | Price | Notes |
|-------|------|-------------|-------|-------|
| RXB6 | Superheterodyne | -107 dBm | Low | Good stability |
| RXB8 | Superheterodyne | -110 dBm | Medium | Better sensitivity |
| MX-RM-5V | Superheterodyne | -105 dBm | Low | Compact size |
| RX470-4 | Regenerative | -100 dBm | Very Low | Unstable, not recommended |
| HC-12 | Transceiver | -117 dBm | High | Bidirectional, overkill |

---

## Timing Diagrams

### EV1527 Bit Encoding

```
Logic '0': Short HIGH, Long LOW
    ____        _________________
___|    |______|
    <-T->  <------3T------>

Logic '1': Long HIGH, Short LOW
    ______________        ____
___|              |______|
    <-----3T------>  <-T->

Preamble: Very Long LOW, Short HIGH
                                    ____
___________________________________|    |___
    <----------31T--------->     <-T->
```

### Frame Structure

```
[Preamble][Bit23][Bit22]...[Bit1][Bit0][Sync]
    |        |                      |      |
  Start   Address              Keys  Stop
         (20 bits)           (4 bits)
```

### State Machine Flow

```
Power On
   |
   v
ev1527_Init() ─> Configure Timer1 & INT0
   |
   v
Wait for INT0 ─────────────────┐
   |                           |
   v                           |
First Edge ─> Reset Variables  |
   |                           |
   v                           |
Falling Edge ─> Capture HIGH   |
   |                           |
   v                           |
Rising Edge ─> Capture LOW     |
   |                           |
   v                           |
Preamble? ──No──> Continue ────┘
   |
  Yes
   v
Decode Bit ─> Store in rawValue
   |
   v
24 Bits? ──No──> Next Bit ─────┘
   |
  Yes
   v
Set Detect Flag
   |
   v
ev1527_deInit() ─> Disable decoder
   |
   v
User Processing
```
---

# 🌟 Support Me
If you found this repository useful:
- Subscribe to my [YouTube Channel](https://www.youtube.com/@aKaReZa75).
- Share this repository with others.
- Give this repository and my other repositories a star.
- Follow my [GitHub account](https://github.com/aKaReZa75).

# ✉️ Contact Me
Feel free to reach out to me through any of the following platforms:
- 📧 [Email: aKaReZa75@gmail.com](mailto:aKaReZa75@gmail.com)
- 🎥 [YouTube: @aKaReZa75](https://www.youtube.com/@aKaReZa75)
- 💼 [LinkedIn: @akareza75](https://www.linkedin.com/in/akareza75)
