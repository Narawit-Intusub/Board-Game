# Slave Expansion Board (บอร์ดลูก) - Autonomous Smart Tile (ATtiny1604)

เอกสารนี้อธิบายรายละเอียดการออกแบบทางฮาร์ดแวร์ บล็อกการทำงาน หน้าที่ของแต่ละโมดูล และสถาปัตยกรรมของ **Slave Expansion Board** (ไฟล์ KiCad: [Slave_Matrix_4x4.kicad_sch](Slave_Matrix_4x4.kicad_sch), [Slave_Matrix_4x4.kicad_pcb](Slave_Matrix_4x4.kicad_pcb)) ภายใต้สถาปัตยกรรม **Modular Magnetic Smart Tile Grid**

---

## 1. ภาพรวมบอร์ดลูก (Slave Board Overview)

**Slave Expansion Board** ขนาดแผ่น **$192.0 \times 192.0\text{ mm}$ ($19.2 \times 19.2\text{ cm}$)** แบ่งเป็น **16 ช่องตาราง ($4 \times 4$ Cells ช่องละ $48.0 \times 48.0\text{ mm}$)** แต่ละช่องติดตั้งไฟ LED 4 ดวงแบบ $2 \times 2$ (รวม 64 หลอด) ทำหน้าที่เป็นกระดานส่วนขยายอัจฉริยะ สามารถเชื่อมต่อกับ Master หรือ Slave แผ่นอื่นได้ทั้ง 4 ทิศทาง (เหนือ, ใต้, ออก, ตก)

```text
+-----------------------------------------------------------------------------------+
|                           Slave Expansion Board                                   |
|                                                                                   |
|                        [ J_NORTH (Female Mag-4P) ]                                |
|                        (VBUS, Link TX, Link RX, GND)                              |
|                                     |                                             |
|                                     v                                             |
|  [ J_WEST (Male) ]  <--->   +---------------+   <--->  [ J_EAST (Female) ]        |
|  (VBUS, TX, RX, GND)        |  ATtiny1604   |          (VBUS, TX, RX, GND)        |
|                             |  Tile Node    |                                     |
|                             +-------+-------+                                     |
|                                     | Tile ID, Dual Framebuffer, Packet Routing   |
|         +---------------+           v                                             |
|         |  Status LED   |           | DIN_LOCAL (5V Logic from ATtiny)            |
|         +---------------+           v                                             |
|                             +---------------+                                     |
|                             |  LED Matrix   | (MOD1 -> MOD2 -> ... MOD16)         |
|                             |  (64 LEDs)    | (Local Driving Only, 16 Cells)      |
|                             +---------------+                                     |
|                                     |                                             |
|                                     v                                             |
|                        [ J_SOUTH (Male Mag-4P) ]                                  |
|                        (VBUS, Link TX, Link RX, GND)                              |
+-----------------------------------------------------------------------------------+
```

---

## 2. คุณสมบัติและความสามารถหลัก (Smart Tile Capabilities)

1. **มี Tile ID ประจำตัว:** บันทึกรหัสบอร์ดเฉพาะตัวไว้ใน Internal Flash / EEPROM
2. **การตรวจจับเพื่อนบ้านทางทิศทาง (Directional Neighbor Discovery):**
   - มีขา **LINK_RX แยกอิสระ 4 ทิศทาง:** `LINK_RX_N`, `LINK_RX_E`, `LINK_RX_S`, `LINK_RX_W`
   - **รู้ทิศทางเมื่อได้รับข้อความ `HELLO`:** โหนดจะระบุตำแหน่งและทิศทางของเพื่อนบ้านได้อย่างแม่นยำจากขา RX ที่ได้รับแพ็กเก็ต `HELLO` (ไม่ใช่การตรวจจับจากการเสียบเพียงอย่างเดียว)
3. **การประหยัด RAM (16 สีหลัก $\rightarrow$ 64 LEDs):**
   - เก็บข้อมูลสีเฉพาะ **16 ช่องตารางเกม** ($16 \times 3 = 48\text{ bytes}$)
   - **Active Buffer:** 48 bytes (กำลังแสดงผล)
   - **Back Buffer:** 48 bytes (กำลังรับข้อมูลเฟรมใหม่)
   - **รวมใช้ RAM เพียง 96 bytes** จาก SRAM 1024 bytes ของ ATtiny1604
   - ขณะขับสัญญาณออกสู่ WS2812 ชิป ATtiny จะทำการ **ขยาย 1 สีไปยัง 4 หลอดในช่องนั้นโดยอัตโนมัติ**
4. **การแสดงผลพร้อมกัน 60 FPS (Hardware Synchronous Commit):**
   - เมื่อได้รับคำสั่ง `FRAME_COMMIT` ผ่านสาย Link ทุกแผ่นจะสลับ Back Buffer $\rightarrow$ Active Buffer และขับรีเฟรชไฟพร้อมกันในระดับไมโครวินาที
5. **การขับไฟภายในบอร์ดตัวเอง (5V Logic):**
   - หาก ATtiny1604 เลี้ยงด้วยไฟ 5V ขา GPIO จะส่งสัญญาณ 5V CMOS Logic ขับขา DIN ของ WS2812B ได้โดยตรง
   - ขา DOUT ของหลอดไฟตัวสุดท้าย (`MOD16`) ไม่ต่อออกไปยังคอนเนคเตอร์ภายนอก

---

## 3. ขั้วต่อแม่เหล็ก 4 จุด (4-Pin Magnetic Link)

ขั้วต่อแม่เหล็ก 4 ขา ออกแบบสลับเพศตามหลัก **Poka-Yoke 180° Inversion**:
* ขอบ **North** และ **East** ใช้ขั้วต่อ **ตัวเมีย (Female)**
* ขอบ **South** และ **West** ใช้ขั้วต่อ **ตัวผู้ (Male)**

$$\text{Pin 1: [ VBUS ]} \quad \vert \quad \text{Pin 2: [ LINK_TX ]} \quad \vert \quad \text{Pin 3: [ LINK_RX ]} \quad \vert \quad \text{Pin 4: [ GND ]}$$

```text
  Female Connector (North / East)       Male Connector (South / West)
      +---------------------+               +---------------------+
      | (1)  (2)  (3)  (4)  |               | (1)  (2)  (3)  (4)  |
      | VBUS LinkTX LinkRX GND|             | GND LinkTX LinkRX VBUS|
      +---------------------+               +---------------------+
```

### การจับคู่สัญญาณเมื่อประกบกัน:
* $\text{Pin 1 (VBUS)} \longleftrightarrow \text{Pin 4 (VBUS)}$: บัสจ่ายไฟหลักเชื่อมต่อถึงกัน
* $\text{Pin 2 (LINK_TX)} \longleftrightarrow \text{Pin 3 (LINK_RX)}$: สายส่งข้อมูลชนสายรับข้อมูล
* $\text{Pin 3 (LINK_RX)} \longleftrightarrow \text{Pin 2 (LINK_TX)}$: สายรับข้อมูลชนสายส่งข้อมูล
* $\text{Pin 4 (GND)} \longleftrightarrow \text{Pin 1 (GND)}$: ระนาบกราวด์ร่วมกัน

---

## 4. การกระจายแพ็กเก็ตและการป้องกันข้อมูลวนซ้ำ (Shared TX & Deduplication)

* **LINK_TX หนึ่งเส้นใช้ร่วมกันทุกด้าน:** เมื่อบอร์ด Slave ส่งแพ็กเก็ตออกทาง `LINK_TX` สัญญาณจะออกไปยังเพื่อนบ้านทุกด้านที่เชื่อมต่ออยู่พร้อมกัน
* **โครงสร้างแพ็กเก็ต:**
  ```text
  +-----------+----------------+-----------------+--------------+-----+-----+--------+
  | Source ID | Destination ID | Sequence Number | Frame Number | TTL | CRC | Data.. |
  +-----------+----------------+-----------------+--------------+-----+-----+--------+
  ```
* **การป้องกันลูป (Loop Prevention):** แต่ละ Tile จะบันทึก `Sequence Number` ล่าสุดของแต่ละโหนด หากได้รับ Packet เดิมซ้ำ จะเพิกเฉยทันที (Drop duplicate) ช่วยป้องกันปัญหาแพ็กเก็ตสะท้อนวนในโครงข่าย

---

## 5. สถาปัตยกรรมระบบไฟ (Power Architecture)

```text
VBUS (รับจากหัวแม่เหล็ก)
   └──► Local Protection
          └──► Local Power Converter
                 └──► 5V คงที่สำหรับขับ LED และเลี้ยง ATtiny1604
```

> [!NOTE]
> **หลักการภาคจ่ายไฟของบอร์ดลูก:**
> * รับไฟผ่านบัสกลาง **`VBUS`** โดยแรงดันและรูปแบบของ Local Power Converter จะถูกสรุปหลังทราบพิกัดกระแสจริงของหัวแม่เหล็กและกำลังวัตต์สูงสุดของ LED ทั้งระบบ
> * ไม่ล็อกเบอร์ TVS (เช่น SMAJ5.0A) ในสเปกหลัก เพื่อรองรับการปรับระดับแรงดัน VBUS ให้เหมาะสมกับการใช้งานจริง
> * `R_BUS` เป็น **DNP โดยปริยาย** ห้ามประกอบเป็น 0Ω จนกว่าจะยืนยันการใช้ VBUS 5V แบบจำกัดกระแส หากใช้บัสแรงดันสูงให้แทนด้วย Local Power Converter ที่มีพิกัดเหมาะกับ LED 64 ดวงต่อ Tile
