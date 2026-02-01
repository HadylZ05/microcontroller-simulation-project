# Simulation Documentation (STM32CubeIDE + Proteus)

## 1. Purpose of This Section

This folder documents the **software implementation and simulation workflow** for the project:

**Multi-Peripheral Control Using STM32 Microcontroller and Proteus Simulation**

The system demonstrates real-time interaction between:
- **16×2 LCD (LM016L) in 4-bit mode**
- **4×3 matrix keypad (scanned via rows/columns)**
- **Green LED output indicator**
- **Proteus simulation using the compiled STM32 `.hex` file**

This section focuses on **how the code is structured**, **what each module does**, and **how the program behaves inside Proteus**.

---

## 2. Simulation Workflow (How It Runs in Proteus)

The implementation is written in C using **STM32CubeIDE** and uses **HAL (Hardware Abstraction Layer)** for GPIO access.

### 2.1 Build Output: `.hex` File
After compiling in STM32CubeIDE, the toolchain produces a `.hex` file, which is the machine-readable firmware image for the STM32F103.

### 2.2 Loading the Firmware into Proteus
In Proteus:
1. Place the **STM32F103C8** component in the schematic.
2. Open its properties.
3. In **Program File**, load the compiled `.hex` file.
4. Run the simulation.

Proteus then emulates how the microcontroller would behave on actual hardware, letting us validate behavior without a physical board.

---

## 3. Code Architecture (Modular Design)

The code is split into independent modules to keep the logic clean and maintainable:

| Module | Files | Responsibility |
|-------|-------|----------------|
| Main Control Logic | `main.c` | System init + main loop behavior |
| LCD Driver | `lcd.c`, `lcd.h` | Low-level LCD control in 4-bit mode |
| Keypad Driver | `keypad.c`, `keypad.h` | Row/column scanning + debouncing |

This modular separation allows the LCD and keypad drivers to be reused easily in other projects.

---

## 4. Main Application Logic (`main.c`)

### 4.1 What `main.c` Does
`main.c` is the “brain” of the program. It:
1. Initializes HAL + clock + GPIO
2. Initializes LCD
3. Prints **"Ready..."**
4. Repeatedly scans keypad input
5. Displays the pressed key
6. Controls the LED based on the key value

### 4.2 Core Startup Sequence
```c
HAL_Init();
SystemClock_Config();
MX_GPIO_Init();

lcd_init();
lcd_send_string("Ready...");
HAL_Delay(500);
```
### 4.3 Real-Time Loop Behavior

The main loop continuously polls the keypad to detect user input.  
When a key press is detected, the system immediately updates the LCD and controls the LED according to the pressed key.

```c
while (1)
{
    char key = keypad_get_key();

    if (key != 0)
    {
        lcd_clear();
        lcd_send_string("Key: ");
        lcd_send_data(key);

        if (key == '1')
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_13, GPIO_PIN_SET);
        else
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_13, GPIO_PIN_RESET);

        HAL_Delay(300); // debounce delay
    }
}
```
### 4.4 Behavior Summary

- The LCD always displays the last pressed key.
- The LED turns ON only when the key '1' is pressed.
- Pressing any other key turns the LED OFF.
- A software delay is applied to prevent multiple detections caused by key bouncing.

## 5. GPIO Initialization (`MX_GPIO_Init()`)

All GPIO configuration is centralized inside the `MX_GPIO_Init()` function to avoid scattered pin logic and to keep hardware initialization clear and maintainable.

---

### 5.1 LCD Pins (GPIOA)

The LCD control and data pins are configured as **push-pull outputs** on GPIOA.

**Pin assignment:**
- **RS** → PA1  
- **E** → PA2  
- **D4–D7** → PA3–PA6  

```c
GPIO_InitStruct.Pin = GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3 |
                      GPIO_PIN_4 | GPIO_PIN_5 | GPIO_PIN_6;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```
### 5.2 Keypad Pins (GPIOB)

The keypad is interfaced using a row–column scanning method.

- **Rows (PB0–PB3):** Configured as outputs and set to an idle HIGH state  
- **Columns (PB4–PB6):** Configured as inputs with internal pull-up resistors  

```c
// Rows OUTPUT (idle HIGH)
GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

// Columns INPUT_PULLUP
GPIO_InitStruct.Pin = GPIO_PIN_4 | GPIO_PIN_5 | GPIO_PIN_5;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
GPIO_InitStruct.Pull = GPIO_PULLUP;
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
```
### 5.3 LED Pin

The LED is connected to PB13, which is configured as a digital output and initialized in the OFF state.

```c
GPIO_InitStruct.Pin = GPIO_PIN_13;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
```

## 6. LCD Driver (lcd.c / lcd.h)
### 6.1 Why 4-bit Mode?

The LCD is used in 4-bit mode to reduce GPIO usage.
Instead of sending 8 bits at once, each byte is split into:
- High nibble (bits 7..4)
- Low nibble (bits 3..0)

### 6.2 LCD Initialization

The initialization sequence switches the LCD into 4-bit mode and configures the display:
```c
void lcd_init(void)
{
    HAL_Delay(40);
    lcd_send_cmd(0x33);
    lcd_send_cmd(0x32);
    lcd_send_cmd(0x28);
    lcd_send_cmd(0x0C);
    lcd_send_cmd(0x06);
    lcd_clear();
}
```
### 6.3 Sending Commands vs Data
```c
lcd_send_cmd() sets RS=0 (command mode)

lcd_send_data() sets RS=1 (data mode)

void lcd_send_cmd(uint8_t cmd)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET); // RS=0
    lcd_write4(cmd >> 4);
    lcd_write4(cmd & 0x0F);
}


void lcd_send_data(uint8_t data)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET); // RS=1
    lcd_write4(data >> 4);
    lcd_write4(data & 0x0F);
}

```
### 6.4 Enable Pulse (Latch Data)

The LCD reads the nibble when E pulses HIGH → LOW:
```c
static void lcd_pulse_enable(void)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_2, GPIO_PIN_SET);
    HAL_Delay(1);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_2, GPIO_PIN_RESET);
    HAL_Delay(1);
}
```
## 7. Keypad Driver (keypad.c / keypad.h)
### 7.1 Keypad Scanning Concept

A matrix keypad works by:
- Setting one row LOW at a time
- Reading which column becomes LOW
- Mapping the (row, column) into a character

### 7.2 Key Mapping Table
```c
static const char keymap[4][3] =
{
    {'1','2','3'},
    {'4','5','6'},
    {'7','8','9'},
    {'*','0','#'}
};
```
### 7.3 Key Detection Algorithm
```c
char keypad_get_key(void)
{
    for (int r = 0; r < 4; r++)
    {
        HAL_GPIO_WritePin(GPIOB, row_pins[0]|row_pins[1]|row_pins[2]|row_pins[3], GPIO_PIN_SET);
        HAL_GPIO_WritePin(GPIOB, row_pins[r], GPIO_PIN_RESET);

        HAL_Delay(1);

        for (int c = 0; c < 3; c++)
        {
            if (HAL_GPIO_ReadPin(GPIOB, col_pins[c]) == GPIO_PIN_RESET)
            {
                HAL_Delay(20); // debounce
                while (HAL_GPIO_ReadPin(GPIOB, col_pins[c]) == GPIO_PIN_RESET);
                return keymap[r][c];
            }
        }
    }
    return '\0';
}
```

### 7.4 Why Debouncing Matters

Mechanical buttons can “bounce”, causing multiple quick transitions.
The debounce delay + waiting for release ensures:
- One press = one detection
- No duplicate characters printed

 ## 8. Error Handling

A critical HAL failure triggers a safe halt:
```c
void Error_Handler(void)
{
    __disable_irq();
    while (1)
    {
        // System halted on error
    }
}
```
## 9. What This Simulation Proves

This simulation successfully demonstrates:
- Multi-peripheral GPIO interfacing
- LCD output in 4-bit mode
- Matrix keypad scanning with debouncing
- Conditional logic controlling a digital output (LED)
- End-to-end validation using Proteus + firmware .hex
The result is a clean, modular, and testable embedded system that reflects real hardware behavior inside a simulation environment.
