# Master Controller Board (บอร์ดแม่) - Dual-MCU Smart Tile (ESP32 + ATtiny1604)

เอกสารนี้อธิบายรายละเอียดการออกแบบทางฮาร์ดแวร์ บล็อกการทำงาน หน้าที่ของแต่ละโมดูล และโครงสร้างทางวิศวกรรมของ **Master Controller Board** (ไฟล์ KiCad: [Master_Matrix_4x4.kicad_sch](Master_Matrix_4x4.kicad_sch), [Master_Matrix_4x4.kicad_pcb](Master_Matrix_4x4.kicad_pcb)) ภายใต้สถาปัตยกรรม **Dual-MCU Smart Tile Grid**

---

## 1. ภาพรวมบอร์ดแม่ (Master Board Overview)

**Master Controller Board** ขนาดแผ่น **$192.0 \times 192.0\text{ mm}$ ($19.2 \times 19.2\text{ cm}$)** แบ่งเป็น **16 ช่องตาราง ($4 \times 4$ Cells ช่องละ $48.0 \times 48.0\text{ mm}$)** แต่ละช่องติดตั้งไฟ LED 4 ดวงแบบ $2 \times 2$ (รวม 64 หลอด) ทำหน้าที่เป็นทั้ง **พื้นที่เล่นในตัว** และ **เกตเวย์เชื่อมต่อกับคอมพิวเตอร์/Unity**

### สถาปัตยกรรมชิปคู่บนบอร์ดแม่ (Dual-MCU Architecture):
บอร์ดแม่รวมไมโครคอนโทรลเลอร์ 2 ตัวไว้บนแผ่นเดียว เพื่อแยกบทบาทหน้าที่อย่างชัดเจน:

1. **ESP32 Gateway (`U1`):**
   - เชื่อมต่อกับ Unity / PC ผ่านสาย USB ทางขอบด้านซ้าย (West)
   - รับข้อมูลเกม, สถานะ, ข้อมูลสี และค่าความสว่างจาก Unity
   - เก็บโครงสร้างแผนผังความสัมพันธ์ทั้งระบบ (System Topology Map)
   - คำนวณ Global Grid และแปลงพิกัดสากล
   - แยกข้อมูล Packet ตาม Tile ID
   - ตรวจสอบสถานะการทำงานของระบบ
   - **ส่งทุก Packet ให้ ATtiny1604 ของบอร์ดแม่ผ่าน Internal UART**
   - *ESP32 ไม่ต่อเข้าหัวแม่เหล็กภายนอกโดยตรง และไม่ขับ LED โดยตรง*

2. **ATtiny1604 Master Tile Controller (`U_ATTINY`):**
   - ทำหน้าที่เป็น Smart Tile ประจำบอร์ดแม่ โดยใช้ **Firmware สถาปัตยกรรมเดียวกับบอร์ดลูก**
   - รับ Packet ข้อมูลจาก ESP32
   - จัดการเก็บสี 16 ช่องในหน่วยความจำ (Active Buffer และ Back Buffer)
   - ขับไฟ LED 64 ดวงในบอร์ดตัวเอง (1 สี ขยายออกเป็น 4 หลอดต่อช่อง)
   - ขับสัญญาณออกสู่โครงข่ายแม่เหล็ก 3 ทิศทาง (**North, East, South**) ผ่านสาย `LINK_TX`
   - กระจายสัญญาณและส่งต่อแพ็กเก็ตไปยังบอร์ดลูก

```text
Unity Game Engine
       │ USB DATA
       ▼
+-------------------------------------------------------------------------+
|                        Master Controller Board                          |
|                                                                         |
|                     [ J_NORTH (Female Mag-4P) ]                         |
|                     (VBUS, Link TX, Link RX, GND)                       |
|                                  ^                                      |
|                                  |                                      |
|  [ POWER INPUT ]            +----+----+                                 |
|  [ USB TO UNITY ]           | ATtiny  | <---> [ J_EAST (Female) ]       |
|         │                   |  1604   |       (VBUS, TX, RX, GND)       |
|         ▼                   | Master  |                                 |
|   +-----------+  Internal   +----+----+                                 |
|   |   ESP32   |  UART            | DIN_LOCAL (5V Logic)                 |
|   |  Gateway  | ---------->      v                                      |
|   +-----------+ (Level Shift)+---------------+                          |
|         │                    |  LED Matrix   | (MOD1 -> MOD2 -> .. MOD16|
|  (System Mgmt)               |  (64 LEDs)    | (Local Driving Only)     |
|   (West Edge)                +---------------+                          |
|                                  |                                      |
|                                  v                                      |
|                     [ J_SOUTH (Male Mag-4P) ]                           |
|                     (VBUS, Link TX, Link RX, GND)                       |
+-------------------------------------------------------------------------+
```

---

## 2. ขั้วต่อแม่เหล็ก 3 ทิศทาง (3-Edge Magnetic Link Bus)

บอร์ด Master ติดตั้งขั้วต่อแม่เหล็กที่ขอบบอร์ด **3 ด้านเท่านั้น** ได้แก่ **North, East, South** (เว้นด้าน **West** ไว้สำหรับช่องรับไฟหลักและพอร์ต USB เชื่อมต่อคอมพิวเตอร์):

$$\text{Pin 1: [ VBUS ]} \quad \vert \quad \text{Pin 2: [ LINK_TX ]} \quad \vert \quad \text{Pin 3: [ LINK_RX ]} \quad \vert \quad \text{Pin 4: [ GND ]}$$

```text
  Female Connector (North / East)       Male Connector (South)
      +---------------------+               +---------------------+
      | (1)  (2)  (3)  (4)  |               | (1)  (2)  (3)  (4)  |
      | VBUS LinkTX LinkRX GND|             | GND LinkTX LinkRX VBUS|
      +---------------------+               +---------------------+
```

### การจับคู่สัญญาณเมื่อประกบกัน (Poka-Yoke 180° Inversion):
* **Pin 1 (VBUS) $\longleftrightarrow$ Pin 4 (VBUS):** บัสจ่ายไฟเลี้ยงหลักของโครงข่าย
* **Pin 2 (LINK_TX) $\longleftrightarrow$ Pin 3 (LINK_RX):** สายส่งข้อมูลชนสายรับข้อมูลของเพื่อนบ้าน
* **Pin 3 (LINK_RX) $\longleftrightarrow$ Pin 2 (LINK_TX):** สายรับข้อมูลชนสายส่งข้อมูลของเพื่อนบ้าน
* **Pin 4 (GND) $\longleftrightarrow$ Pin 1 (GND):** ระนาบกราวด์ร่วมของทั้งระบบ

---

## 3. เส้นทางข้อมูลและการกระจายสัญญาณ (Data Routing & Level Shifting)

1. **เส้นทางข้อมูล:**
   * Unity $\rightarrow$ ESP32 (ถอดรหัสและแยก Packet)
   * ESP32 $\rightarrow$ ATtiny Master (ผ่าน Internal UART)
   * ATtiny Master $\rightarrow$ ขับไฟ LED 64 ดวงบนบอร์ด Master
   * ATtiny Master $\rightarrow$ กระจายข้อมูลออกสู่ `LINK_TX` ขั้วต่อแม่เหล็ก N/E/S ไปยังบอร์ดลูก
2. **การแปลงระดับแรงดัน (Level Shifting):**
   * หาก ATtiny1604 ทำงานที่แรงดัน 5V เอาต์พุต GPIO ของ ATtiny จะเป็น 5V Logic อยู่แล้ว จึงขับขา DIN ของ WS2812B ได้โดยตรง
   * จุดที่ต้องแปลงระดับแรงดันคือระหว่าง **ESP32 (3.3V Logic)** กับ **ATtiny1604 (5V Logic)** บนสัญญาณ UART ภายใน เพื่อความเข้ากันได้ทางไฟฟ้าและความปลอดภัยของชิป
   * ทิศ ESP32 → ATtiny ใช้ `74AHCT1G125`; ทิศ ATtiny → ESP32 ใช้ตัวแบ่งแรงดัน 10kΩ/20kΩ เพื่อจำกัดสัญญาณประมาณ 3.3V
3. **Current monitor:** บัส I²C ของ INA219 มี pull-up 4.7kΩ ไปยัง `+3V3` แยกทั้ง SDA และ SCL

---

## 4. สถาปัตยกรรมระบบไฟ (Power Architecture)

หลักการจ่ายไฟของระบบ:
```text
Power Input
   └──► Master Protection
          └──► VBUS ผ่านขั้วต่อแม่เหล็ก
                 └──► Local Power Converter ในแต่ละ Tile
                        └──► 5V คงที่สำหรับขับ LED และเลี้ยง ATtiny1604
```

> [!NOTE]
> **การกำหนดสเปกภาคจ่ายไฟ:**
> * บัสส่งไฟระหว่างบอร์ดใช้ชื่อกลางว่า **`VBUS`** เพื่อรองรับการทดสอบแรงดันที่เหมาะสม (เช่น 5V ตรง หรือ 9V–12V ร่วมกับ Local Step-Down Converter) เพื่อลดปัญหากระแสสูงและแรงดันตกคร่อมบนพินแม่เหล็ก
> * สเปกของ Power Input, พิกัดแรงดัน VBUS, วงจรตรวจวัดกระแส, และเบอร์ไดโอด TVS จะถูกกำหนดอย่างเป็นทางการหลังทราบพิกัดกระแสจริงของหัวแม่เหล็กและกำลังวัตต์สูงสุดของ LED ทั้งระบบ (ไม่ล็อกเบอร์ SMAJ5.0A หาก VBUS ถูกปรับขึ้นสูงกว่า 5V)
> * ตัวต้านทาน CC 5.1kΩ บนพอร์ต Type-C ทำหน้าที่ระบุสถานะ Sink พื้นฐาน ไม่ได้รับรองการจ่าย 5V/3A และไม่ใช่ระบบ USB-PD
> * `R_BUS` เป็น **DNP โดยปริยาย** ห้ามประกอบเป็น 0Ω จนกว่าจะยืนยันว่าใช้ VBUS 5V แบบจำกัดกระแส หากต้องการขยายหลาย Tile ให้แทนตำแหน่งนี้ด้วย Local Power Converter ที่เลือกตามพิกัดจริง
> * ขา `VCC_(USB)` ของโมดูล ESP32 ไม่ผูกเข้ากับราง `+5V` ภายนอก เพื่อป้องกันการป้อนไฟย้อนเข้าสาย USB Data ของคอมพิวเตอร์

---

## 5. การส่งแพ็กเก็ตและตรวจจับเพื่อนบ้าน

* **การส่งข้อมูล (`LINK_TX`):** ATtiny Master ส่งออกทาง `LINK_TX` ซึ่งเชื่อมขนานไปยังขั้วต่อ N, E, S สัญญาณจะออกไปยังเพื่อนบ้านทุกด้านพร้อมกัน
* **การป้องกันข้อมูลวนซ้ำ (Deduplication):** Packet จะระบุ `Source ID`, `Destination ID`, `Sequence Number`, `Frame Number`, `TTL`, และ `CRC` ทุกโหนดจะไม่ประมวลผล Packet เดิมซ้ำ
* **การค้นหาเพื่อนบ้าน (Neighbor Discovery):** ขา `LINK_RX` แยกตามทิศทาง (`LINK_RX_N`, `LINK_RX_E`, `LINK_RX_S`) เมื่อได้รับข้อความ `HELLO` ทางขาใด จะทำให้ ATtiny ทราบทิศทางและบอร์ดที่เชื่อมต่ออยู่ทันที และส่งรายงานกลับไปยัง ESP32
