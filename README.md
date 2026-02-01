# microcontroller-project-(Proteus Simulation)

##  Overview 

This project implements a **keypad-controlled user interface** using the **STM32F103C8 (Blue Pill)** microcontroller.  
A **4×3 matrix keypad**, **16×2 character LCD**, and **LED** are interfaced using GPIO and controlled through embedded C code developed with **STM32CubeIDE**.

The system was **designed and validated entirely using Proteus 8 Professional**, as required by the course, to simulate real hardware behavior without physical components.

---

##  Project Features

- STM32F103C8 (ARM Cortex-M3) microcontroller
- 16×2 LCD interfaced in **4-bit mode**
- 4×3 matrix keypad with **software debounce**
- LED control based on keypad input
- Modular and reusable embedded C drivers
- Full system validation via simulation

---

##  System Logic

**Behavior**
1. Initialize GPIO, LCD, and keypad
2. Display `"READY"` on the LCD
3. Continuously scan keypad:
   - Display pressed key on LCD
   - If key = `'1'` → LED ON
   - Else → LED OFF
   - Apply debounce delay

---

##  Tools & Technologies

- STM32CubeIDE
- STM32 HAL Library
- Proteus 8 Professional
- Embedded C
- ARM Cortex-M3

---

##  Notes
This project follows a **simulation-first embedded design approach**, commonly used in early development stages to validate logic, timing, and peripheral interfacing before hardware deployment.

---

## Author
**Hadil Zahiri**  
Computer Engineering Graduate

