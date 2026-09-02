# Master Controller Board (บอร์ดแม่) - Modular Smart LED Matrix (19.2x19.2 cm for 19.5cm Enclosure)

เอกสารนี้อธิบายรายละเอียดการออกแบบทางฮาร์ดแวร์ บล็อกการทำงาน หน้าที่ของแต่ละโมดูล/ชิ้นส่วน การเชื่อมต่อสัญญาณ และโครงสร้างแผ่นวงจรพิมพ์ (PCB Layout) ของ **Master Controller Board** (ไฟล์ KiCad: [Master_Matrix_4x4.kicad_sch](Master_Matrix_4x4.kicad_sch), [Master_Matrix_4x4.kicad_pcb](Master_Matrix_4x4.kicad_pcb)) อย่างครบถ้วน

---

## 1. ภาพรวมบอร์ดแม่ (Master Board Overview)

**Master Controller Board** คือ "สมองส่วนกลาง" ของระบบกระดานไฟ LED อัจฉริยะแบบต่อขยายได้ (Modular Smart LED Matrix) ขนาดแผ่น **$192.0 \times 192.0\text{ mm}$ ($19.2 \times 19.2\text{ cm}$)** สำหรับติดตั้งในโครง 3D Print ขนาด $19.5 \times 19.5\text{ cm}$ (พร้อมระยะเผื่อรอบขอบ $1.5\text{ mm}$) แบ่งเป็น **16 ช่องตาราง ($4 \times 4$ Cells ช่องละ $48.0 \times 48.0\text{ mm}$)**

### หน้าที่หลักของบอร์ดแม่:
1. **ศูนย์กลางประมวลผลและสร้างภาพ (Graphics & Game Engine):** รับคำสั่ง/ข้อมูลภาพผ่าน Wi-Fi หรือ Bluetooth และคำนวณตำแหน่งพิกเซล $(X, Y)$ สำหรับการแสดงผลเกมและเอฟเฟกต์ไฟทั้งหมด
2. **การส่งข้อมูลพิกเซลความเร็วสูง (High-Speed LED Data Transmission):** แปลงสัญญาณข้อมูลสี 800 kHz จากระดับแรงดัน $3.3\text{V}$ เป็น $5.0\text{V}$ ขับเข้าสู่ Matrix ไฟ 64 ดวงบนบอร์ดตัวเอง และกระจายต่อออกไปยังบอร์ดลูก (Slave) ผ่านขั้วต่อแม่เหล็ก 3 ทิศทาง
3. **การตรวจจับบอร์ดลูกรอบทิศ (Neighbor Auto-Discovery):** ตรวจจับว่ามีบอร์ดลูกมาเชื่อมต่อที่ทิศเหนือ, ทิศใต้ หรือทิศตะวันออก ผ่านพิน Sense (`D01`, `D11`, `D21`)
4. **พอร์ตพัฒนาและควบคุมฝั่งซ้าย (West-Side USB Service Port):** ขอบด้านซ้ายของบอร์ดเปิดโล่งสำหรับการเสียบสาย USB เข้ากับ ESP32 เพื่อ **Upload Code / Flash Firmware / Debug Serial Monitor** ได้อย่างสะดวกสบายโดยไม่ถูกบอร์ดอื่นบดบัง

```text
+-------------------------------------------------------------------------+
|                        Master Controller Board                          |
|                                                                         |
|                     [ J_NORTH (Female Mag-4P) ]                         |
|                                  ^                                      |
|                                  | (DOUT, SENSE_N, +5V, GND)            |
|                                  v                                      |
|  [ USB FLASH / DEBUG ] <--- +---------------+ ---> [ J_EAST (Female) ]  |
|  (West Edge Interface)      |   ESP32 MCU   |                           |
|                             |  DevKit V1    |                           |
|                             +-------+-------+                           |
|                                     | GPIO18 (3.3V Data)                |
|                                     v                                   |
|                             [ 74AHCT1G125 ] Level Shifter               |
|                                     | DIN (5.0V Data via 330R)          |
|                                     v                                   |
|                             +---------------+                           |
|                             |  LED Matrix   | (MOD1 -> MOD2 -> .. MOD16)|
|                             |  (64 LEDs)    | (16 Cells, 50x50mm each)  |
|                             +-------+-------+                           |
|                                     | DOUT                              |
|                                     v                                   |
|                     [ J_SOUTH (Male Mag-4P) ]                           |
+-------------------------------------------------------------------------+
```

---

## 2. รายการโมดูลและอุปกรณ์ทั้งหมด (Component Breakdown)

| หมายเลขอุปกรณ์ (Ref) | ชนิด / อุปกรณ์ (Value) | Footprint / Package | หน้าที่การทำงาน |
| :--- | :--- | :--- | :--- |
| **`U1`** | **esp32-wemos-d1-mini** | `esp32-wemos-d1-mini:esp32-wemos-d1-mini` (B.Cu) | ไมโครคอนโทรลเลอร์หลักขนาดกะทัดรัด (Dual-Core 240MHz, Wi-Fi, BLE 4.2) พร้อมโมเดล 3D Step และ Pinout เฉพาะของบอร์ด Wemos D1 Mini ESP32 |
| **`U2`** | **74AHCT1G125** | SOT-23-5 (B.Cu) | ไอซีแปลงระดับแรงดันสัญญาณความเร็วสูง (High-Speed Level Shifter) แปลงสัญญาณข้อมูลจาก 3.3V ของ ESP32 เป็น 5.0V มาตรฐาน WS2812 |
| **`MOD1 - MOD16`** | **WS2812_Matrix_2x2** | `ws2812-matrix-2x2:WS2812_Matrix_2x2_15x15mm` (F.Cu) | โมดูลไฟ LED RGB Addressable แบบ SMD 15x15mm (8 Pads: 4 IN / 4 OUT) (โมดูลละ 4 LEDs รวม 64 หลอด ต่อเรียงแบบ Serpentine Daisy-Chain) |
| **`C_MOD1 - C_MOD16`** | **100nF (0.1µF)** | SMD 0805 Standard (B.Cu) | ตัวเก็บประจุ Decoupling ประจำแต่ละโมดูล LED ช่วยกรองสัญญาณรบกวนและสำรองกระแสไฟ 5V (Footprint IPC-7351 มาตรฐาน) |
| **`C1`** | **100nF** | SMD 0603 (B.Cu) | ตัวเก็บประจุ Decoupling สำหรับเส้นไฟ 3.3V ของ ESP32 |
| **`C2`** | **100nF** | SMD 0603 (B.Cu) | ตัวเก็บประจุ Decoupling สำหรับไฟเลี้ยง 5.0V ของไอซี `U2` (74AHCT1G125) |
| **`R1`** | **330Ω** | SMD 0603 (B.Cu) | ตัวต้านทานต่ออนุกรมหัวแถว Data In (Damping Resistor) เพื่อป้องกันการสะท้อนของสัญญาณ (Signal Ringing) |
| **`R2`** | **10kΩ** | SMD 0603 (B.Cu) | ตัวต้านทาน Pull-up บนขา EN (Enable/Reset) ของ ESP32 ดึงไปที่ +3.3V เพื่อความเสถียร |
| **`R_CC1, R_CC2`** | **5.1kΩ** | SMD 0603 (B.Cu) | ตัวต้านทาน Pull-down บนขา CC1/CC2 ของพอร์ต USB-C เพื่อรองรับหัวชาร์จ PD / Fast Charge / Power Bank ทุกรุ่น |
| **`D_TVS`** | **SMAJ5.0A** | SMD SMA / DO-214AC (F.Cu) | ไดโอดป้องกันไฟกระชาก (TVS Surge Protection 5V 400W) ป้องกันแรงดันเกินและ ESD จากการเสียบสาย |
| **`C_BULK`** | **100µF** | SMD 1206 (F.Cu) | ตัวเก็บประจุสำรองพลังงานหลัก (Bulk Reservoir Cap) พยุงแรงดันไฟ 5V ป้องกันไฟตกเมื่อ LED 64 ดวงเปิดสว่างพร้อมกัน |
| **`J_PWR`** | **USB_C_6P_3A_5V_IN** | USB Type-C 6P SMD (4 PTH Shell Tabs) | พอร์ตรับไฟเลี้ยงหลัก 5V กระแสสูง 3A-5A มาตรฐาน USB Type-C 6-Pin (Power Only) บัดกรีง่าย แข็งแรง รองรับสายชาร์จและ Power Bank ทุกรุ่น |
| **`SW1`** | **SW_SPDT** | `Button_Switch_SMD:SW_SPDT_PCM12` | สวิตช์เลื่อนเปิด-ปิดระบบหลัก (Main Power Switch) ควบคุมการจ่ายไฟ 5V จากพอร์ต USB-C เข้าสู่บอร์ด |
| **`J_NORTH`** | **Mag_North (Female)** | 4-Pin Magnetic 90° | ขั้วต่อแม่เหล็กฝั่งทิศเหนือ (รับ/ส่งไฟ 5V, GND, Data Out, ตรวจจับทิศ D01) |
| **`J_EAST`** | **Mag_East (Female)** | 4-Pin Magnetic 90° | ขั้วต่อแม่เหล็กฝั่งทิศตะวันออก (รับ/ส่งไฟ 5V, GND, Data Out, ตรวจจับทิศ D11) |
| **`J_SOUTH`** | **Mag_South (Male)** | 4-Pin Magnetic 90° | ขั้วต่อแม่เหล็กฝั่งทิศใต้ (รับ/ส่งไฟ 5V, GND, Data Out, ตรวจจับทิศ D21) |
| **`H1 - H4`** | **MountingHole_3.2mm_M3** | NPTH Hole $\varnothing 3.2\text{ mm}$ | รูยึดน็อต M3 ที่มุมทั้ง 4 ด้าน พร้อมวงแหวน Keepout $\varnothing 6.0\text{ mm}$ |
| **`FID_F1-3, FID_B1-3`** | **Fiducial_1mm** | SMT Pad $\varnothing 1.0\text{ mm}$ | จุดมาร์คตำแหน่ง Fiducial สำหรับเครื่องจักร SMT Assembly บน F.Cu และ B.Cu |

---

## 3. รายละเอียดการทำงานของวงจรสำคัญ

### 3.1 วงจรแปลงระดับสัญญาณ (Level Shifter Circuit - 74AHCT1G125)
* **ปัญหาทางเทคนิค:** ขา GPIO ของ ESP32 จ่ายสัญญาณ Logic HIGH ที่ **3.3V** แต่หลอด WS2812B/SK6812 ที่ใช้ไฟเลี้ยง 5.0V ต้องการแรงดัน $V_{IH}$ ขั้นต่ำที่ $0.7 \times V_{DD} = 3.5\text{V}$ หากต่อตรงโดยไม่แปลงระดับ สัญญาณอาจกระพริบ เพี้ยน หรือทำงานไม่เสถียร
* **การทำงานของ 74AHCT1G125:**
  - ชิปตระกูล **AHCT** มี $V_{IH}$ ขั้นต่ำเพียง **2.0V** ที่ไฟเลี้ยง 5V (TTL-compatible) จึงรับสัญญาณ 3.3V จาก ESP32 ได้อย่างสมบูรณ์แบบ
  - ขา Output จ่ายสัญญาณ Swing เต็มที่ **0V ถึง 5.0V** สลับสัญญาณความเร็วสูง 100+ MHz (สัญญาณ WS2812B วิ่งที่ 800 kHz)
  - ขา `~OE` (Pin 1) ต่อลง GND เพื่อ Enable เอาต์พุตไว้ตลอดเวลา

```text
  ESP32 (GPIO18) --------> [ Pin 2 (A) ]
                              74AHCT1G125  --------> [ Pin 4 (Y) ] ---> [ R1: 330R ] ---> DIN (เข้า MOD1)
  +5V -------------------> [ Pin 5 (VCC) ]
  GND -------------------> [ Pin 1 (~OE), Pin 3 (GND) ]
```

---

### 3.2 วงจรแสดงผล LED Matrix 4x4 (MOD1 - MOD16)
บอร์ดแม่ประกอบด้วยโมดูล **WS2812 4-in-1 (MOD1 - MOD16)** จำนวน 16 ตัว เชื่อมต่อกันแบบ **Daisy-Chain**:

```text
 แถว 1:  DIN ---> [MOD1] ---> [MOD2] ---> [MOD3] ---> [MOD4]
                                                         |
 แถว 2:  +------> [MOD5] ---> [MOD6] ---> [MOD7] ---> [MOD8]
         |                                               |
 แถว 3:  +------> [MOD9] ---> [MOD10] --> [MOD11] --> [MOD12]
         |                                               |
 แถว 4:  +------> [MOD13] --> [MOD14] --> [MOD15] --> [MOD16]
                                                         |
                                                         +---> DOUT (ส่งต่อไปยังขั้วต่อแม่เหล็ก North, South, East)
```

* แต่ละช่อง $5 \times 5\text{ cm}$ มีโมดูล LED 4 ดวง วางตรงกึ่งกลาง
* 16 ช่องรวมกัน = 64 หลอด LED
* สัญญาณขาออกสุดท้ายจาก `MOD16 (Pin 4)` จะเป็นสัญญาณ `DOUT` ที่ถูก Re-timing และชดเชยระดับแรงดันโดยอัตโนมัติจากชิป WS2812 ตัวสุดท้าย พร้อมส่งต่อไปยังบอร์ดลูกตัวถัดไป

---

### 3.3 การทำงานของ Magnetic Connectors (ขั้วต่อแม่เหล็ก 3 ทิศ)

ขั้วต่อแม่เหล็ก 4 ขา (Magnetic Pogo Pin Connector 90°) ติดตั้งที่ขอบบอร์ด 3 ด้าน:

```text
  Female Connector (North / East)       Male Connector (South)
      +---------------------+               +---------------------+
      | (1)  (2)  (3)  (4)  |               | (1)  (2)  (3)  (4)  |
      | GND SENSE DATA +5V  |               | +5V DATA SENSE GND  |
      +---------------------+               +---------------------+
```

#### ตารางนิยามขาสัญญาณของขั้วต่อแม่เหล็ก (Pin Assignment):
| Pin # | North Connector (`J_NORTH`) Female | East Connector (`J_EAST`) Female | South Connector (`J_SOUTH`) Male | หน้าที่ของสัญญาณ |
| :---: | :---: | :---: | :---: | :--- |
| **1** | `GND` | `GND` | `+5V` | ไฟเลี้ยงระบบ / กราวด์อ้างอิง |
| **2** | `D01` (SENSE_N) | `D11` (SENSE_E) | `DOUT` (Data Out) | สัญญาณตรวจจับ หรือ สัญญาณข้อมูลสี |
| **3** | `DOUT` (Data Out) | `DOUT` (Data Out) | `D21` (SENSE_S) | สัญญาณข้อมูลสี หรือ สัญญาณตรวจจับ |
| **4** | `+5V` | `+5V` | `GND` | ไฟเลี้ยงระบบ / กราวด์อ้างอิง |

#### กลไกความปลอดภัยและการจับคู่แบบสมมาตร (Poka-Yoke Symmetry):
* **ป้องกันการกลับขั้ว:** ขอบ North/East ใช้หัวตัวเมีย (Female) ขอบ South ใช้หัวตัวผู้ (Male) เมื่อนำบอร์ดสองแผ่นมาประกบกัน พินจะสลับขั้ว 180° พอดี:
  - Pin 1 (GND) จะตรงกับ Pin 4 (GND)
  - Pin 2 (SENSE) จะตรงกับ Pin 3 (SENSE)
  - Pin 3 (DOUT) จะตรงกับ Pin 2 (DATA IN ของบอร์ดลูก)
  - Pin 4 (+5V) จะตรงกับ Pin 1 (+5V)
* **กลไกการตรวจจับเพื่อนบ้าน (Neighbor Auto-Discovery):**
  - ขา `D01` (ESP32 IO26), `D11` (ESP32 IO14), `D21` (ESP32 IO13) ตั้งโหมดเป็น Input with Pull-up ใน ESP32
  - เมื่อบอร์ดลูกมาต่อ ขาสัญญาณ Sense จะถูกดึงแรงดันลง ทำให้ ESP32 ทราบทันทีว่ามีบอร์ดลูกเชื่อมต่ออยู่ที่ทิศใด

---

## 4. แผนผังการต่อขาไมโครคอนโทรลเลอร์ (ESP32 Pinout Mapping)

| ESP32 Pin | Net Name บน Schematic | การเชื่อมต่อไปยัง | คำอธิบายหน้าที่ |
| :--- | :--- | :--- | :--- |
| **Pin 15 (VIN/5V)** | `+5V` | `J_PWR` (Pin 1), Rail 5V | รับไฟเลี้ยง 5.0V จากภายนอกเข้าเรกูเลเตอร์ 3.3V บนตัว DevKit |
| **Pin 14, 29 (GND)**| `GND` | Ground Plane | จุดต่อกราวด์ร่วมของระบบ |
| **Pin 30 (3V3)** | `+3V3` | `C1`, `R2` | ไฟเลี้ยง 3.3V จ่ายโดยออนบอร์ด LDO ของ ESP32 |
| **Pin 22 (IO18)** | - | `U2` Pin 2 (74AHCT1G125 Input A) | เอาต์พุตส่งข้อมูลไฟ LED ความเร็วสูง (WS2812 Data Out 800kHz) |
| **Pin 9 (IO26)** | `D01` | `J_NORTH` Pin 2 | พินตรวจจับการเชื่อมต่อบอร์ดด้านทิศเหนือ (North Detect) |
| **Pin 11 (IO14)** | `D11` | `J_EAST` Pin 2 | พินตรวจจับการเชื่อมต่อบอร์ดด้านทิศตะวันออก (East Detect) |
| **Pin 13 (IO13)** | `D21` | `J_SOUTH` Pin 3 | พินตรวจจับการเชื่อมต่อบอร์ดด้านทิศใต้ (South Detect) |
| **Pin 7 (IO35)** | `NC` | - | สำรอง / ไม่ใช้งาน (No-Connect) |

---

## 5. คุณสมบัติของแผ่นวงจรพิมพ์ (PCB Physical & DFM Specifications)

- **ขนาดบอร์ด (Dimensions):** $192.0\text{ mm} \times 192.0\text{ mm}$ ($19.2 \times 19.2\text{ cm}$) สำหรับเคส 3D Print $19.5\text{ cm}$
- **ความหนาแผ่น (Thickness):** $1.6\text{ mm}$, Copper $1\text{ oz}$ (FR-4, 2 Layers)
- **การลบมุมขอบบอร์ด:** โค้งมนรัศมี $R = 3.0\text{ mm}$ ปลอดภัยต่อการหยิบจับและสไลด์ใส่เคส 3D Print ได้ง่าย
- **การเดินลายวงจร (Routing):**
  - ลายวงจรทำมุม 45 องศาทุกจุด (No sharp 90° bends)
  - รางไฟ $+5\text{V}$ เส้นหลักขนาด $1.5\text{ mm}$ และรางจ่ายตามแถว $1.0\text{ mm}$
  - เส้นสัญญาณ $0.35\text{ mm}$
- **ระนาบกราวด์ (GND Plane):** เทกราวด์เต็มทั้งสองหน้า (`F.Cu` & `B.Cu`) พร้อมจุดเย็บกราวด์ **GND Stitching Vias จำนวน 16 จุด**
- **ความพร้อมในการประกอบ (SMT Ready):** มีมาร์คเกอร์ **Fiducial 1.0mm** บนเลเยอร์หน้าและหลังสำหรับเครื่อง Pick-and-Place อัตโนมัติ

---

## 6. ข้อแนะนำในการประกอบและทดสอบ (Assembly & Testing Tips)

1. **การจ่ายไฟ (Power Delivery & Power Switch):**
   - เมื่อเปิดไฟ LED สีขาวสว่างสูงสุดทั้ง 64 หลอด บอร์ดแม่จะกินกระแสประมาณ $64 \times 60\text{mA} = 3.84\text{A}$
   - จ่ายไฟ 5V ผ่านทาง `J_PWR` (USB Type-C 6P) และควบคุมการเปิด-ปิดไฟเลี้ยงระบบทั้งหมดด้วยสวิตช์ **`SW1` (Main Power Switch)**
   - เมื่อเสียบสาย USB แล้ว ระบบจะยังไม่ได้รับไฟจนกว่าจะเลื่อนเปิดสวิตช์ `SW1` เพื่อความปลอดภัยและสะดวกในการใช้งาน
2. **การอัพโหลดโปรแกรม (Flashing Firmware):**
   - เสียบสาย Micro-USB / Type-C เข้ากับบอร์ด ESP32 ทางขอบซ้ายของบอร์ดได้โดยตรง สามารถ Flash โค้ดและดู Serial Monitor ได้ทันที
3. **การตรวจสอบสัญญาณ:**
   - ทดสอบวัดสัญญาณที่ขา 4 ของ `U2` ด้วย Oscilloscope ควรเห็น Pulse สัญญาณความกว้างประมาณ 0.4µs / 0.8µs ที่แรงดัน 5.0V สวยงามไม่มี Overshoot
