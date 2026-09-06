# Modular Smart LED Matrix (กระดานไฟ LED อัจฉริยะต่อขยายได้อิสระสำหรับระบบเกมกระดาน)

---

## 1. ภาพรวมโปรเจกต์ (Project Overview)

**Modular Smart LED Matrix** คือ กระดานแสดงผลไฟ LED อัจฉริยะแบบต่อขยายได้อิสระ (Modular Interactive Tabletop Gaming Grid) ออกแบบมาเพื่อเป็นกระดานเล่นเกมแบบ Interactive ร่วมกับเอนจินเกม (เช่น Unity หรือ PC) และหน้าจอแสดงผลกราฟิกแบบปรับเปลี่ยนรูปทรงได้อย่างอิสระ

ระบบถูกออกแบบภายใต้สถาปัตยกรรม **Smart Tile (Distributed Grid Architecture)** โดยแต่ละแผ่นกระดานมีขนาด **$192.0 \times 192.0\text{ mm}$ ($19.2 \times 19.2\text{ cm}$)** แบ่งเป็น **16 ช่องตาราง ($4 \times 4$ Cells ช่องละ $48.0 \times 48.0\text{ mm}$)** แต่ละช่องมีไฟ 4 หลอด (รวม 64 หลอดต่อแผ่น) ขับแสดงผลเฉพาะในบอร์ดตัวเองอย่างอิสระ ไม่ต่อสายไฟ LED เป็นโซ่อนุกรมข้ามบอร์ด

```text
Unity / PC
    │
    │ USB DATA
    ▼
┌───────────────────────────────┐
│ MASTER                        │
│                               │
│ ESP32 Gateway                 │
│    │ Internal UART            │
│    ▼                          │
│ ATtiny1604 Tile Controller    │
│    ├── LED Matrix ของ Master  │
│    └── Magnetic Bus N/E/S     │
└───────────────┬───────────────┘
                │
        VBUS, TX, RX, GND
                │
       ┌────────┴────────┐
       ▼                 ▼
┌─────────────┐   ┌─────────────┐
│ Slave Tile  │   │ Slave Tile  │
│ ATtiny1604  │   │ ATtiny1604  │
└─────────────┘   └─────────────┘
```

---

## 2. โครงสร้างระบบ (System Structure)

ชุดต้นแบบประกอบด้วยกระดานทั้งหมด 4 แผ่น:

| บอร์ด | ตัวควบคุม | หน้าที่หลัก |
| :--- | :--- | :--- |
| **Master (1 แผ่น)** | **ESP32 Wemos D1 Mini + ATtiny1604** | • **ESP32:** เกตเวย์เชื่อม Unity ผ่าน USB, เก็บ Topology ทั้งระบบ, คำนวณ Global Grid, แยก Packet ตาม Tile ID, ตรวจสอบสถานะระบบ<br>• **ATtiny1604:** ควบคุม Tile ของ Master, ขับ 64 LEDs, จัดการ Magnetic Bus N/E/S |
| **Slave 1 (1 แผ่น)** | **ATtiny1604** | ขับ LED 64 ดวงในตัว, รับ-ส่งต่อ Packet, ตรวจจับเพื่อนบ้าน |
| **Slave 2 (1 แผ่น)** | **ATtiny1604** | ขับ LED 64 ดวงในตัว, รับ-ส่งต่อ Packet, ตรวจจับเพื่อนบ้าน |
| **Slave 3 (1 แผ่น)** | **ATtiny1604** | ขับ LED 64 ดวงในตัว, รับ-ส่งต่อ Packet, ตรวจจับเพื่อนบ้าน |

### สรุปรวมอุปกรณ์หลักของระบบ:
* **ESP32:** จำนวน 1 ตัว (อยู่บนบอร์ด Master เท่านั้น ทำหน้าที่ Gateway ระดับระบบ)
* **ATtiny1604:** จำนวน 4 ตัว (ประจำทุกแผ่น แผ่นละ 1 ตัว ทำหน้าที่ Tile Controller มาตรฐานเดียวกัน)
* **USB Data:** ต่อคอมพิวเตอร์เฉพาะบอร์ด Master เพียงจุดเดียว
* **Power Input:** จ่ายไฟเข้าที่บอร์ด Master เพียงจุดเดียว แล้วส่งผ่านบัส `VBUS` ไปยังแต่ละแผ่น

---

## 3. หน้าที่ของแต่ละตัวควบคุม

### 3.1 หน้าที่ของ ESP32 (System Gateway)
* ติดต่อกับ Unity ผ่านสาย USB เพียงเส้นเดียว
* รับสถานะเกม ข้อมูลสีพิกัด และคำสั่งเฟรมภาพ
* บันทึกและวิเคราะห์โครงสร้างแผนผังทั้งหมดของระบบ (System Topology)
* คำนวณตำแหน่งสัมพัทธ์และการหมุนของแต่ละ Tile รวมเป็น Global Grid ผืนเดียว
* แยกข้อมูล Packet ตาม Tile ID
* ตรวจสอบสถานะและรายงานความผิดปกติของระบบ
* **ส่ง Packet ทั้งหมดให้ ATtiny1604 ของ Master ผ่าน Internal UART**
* *ESP32 ไม่ต้องต่อเข้าหัวแม่เหล็กภายนอกโดยตรง และไม่ต้องขับ LED โดยตรง*

### 3.2 หน้าที่ของ ATtiny1604 (Tile Controller ประจำทุกแผ่น รวม 4 ตัว)
* มีรหัส **Tile ID** ประจำตัว
* ขับไฟ LED ภายในแผ่นตัวเอง (64 LEDs จากข้อมูลสี 16 ช่อง)
* จัดการหน่วยความจำสี 16 ช่อง (Active Buffer และ Back Buffer)
* รับและส่งต่อ Packet ข้อมูลตามเส้นทางเครือข่าย
* ส่งข้อความ `HELLO` เพื่อค้นหาเพื่อนบ้าน
* **ตรวจจับเพื่อนบ้านและทิศทางการเชื่อมต่อเมื่อได้รับข้อความ `HELLO` ทางขา RX แยกแต่ละทิศ (`LINK_RX_N/E/S/W`)**
* รอรับสัญญาณแสดงผลพร้อมกัน (`FRAME_COMMIT`)
* รายงานสถานะกลับไปยัง ESP32 ผ่านโครงข่ายบัส
* **ATtiny บน Master และ Slave ใช้ Firmware สถาปัตยกรรมพื้นฐานชุดเดียวกัน**

---

## 4. บัสแม่เหล็ก 4 จุด (4-Pin Magnetic Link)

แต่ละขอบที่เชื่อมต่อระหว่างบอร์ดมีพินสัญญาณ 4 พิน:

$$\text{[ VBUS ]} \quad \vert \quad \text{[ LINK_TX ]} \quad \vert \quad \text{[ LINK_RX ]} \quad \vert \quad \text{[ GND ]}$$

```text
  Female Connector (North / East)       Male Connector (South / West)
      +---------------------+               +---------------------+
      | (1)  (2)  (3)  (4)  |               | (1)  (2)  (3)  (4)  |
      | VBUS LinkTX LinkRX GND|             | GND LinkTX LinkRX VBUS|
      +---------------------+               +---------------------+
```

### 4.1 การจับคู่เมื่อประกบบอร์ด (Poka-Yoke 180° Inversion):
เมื่อนำขอบ Female ประกบกับขอบ Male หลังหมุน 180° พินจะจับคู่ตรงกันอย่างถูกต้อง:
* $\text{Pin 1 (VBUS)} \longleftrightarrow \text{Pin 4 (VBUS)}$: บัสไฟเลี้ยงหลัก
* $\text{Pin 2 (LINK_TX)} \longleftrightarrow \text{Pin 3 (LINK_RX)}$: สายส่งข้อมูลชนสายรับข้อมูล
* $\text{Pin 3 (LINK_RX)} \longleftrightarrow \text{Pin 2 (LINK_TX)}$: สายรับข้อมูลชนสายส่งข้อมูล
* $\text{Pin 4 (GND)} \longleftrightarrow \text{Pin 1 (GND)}$: ระนาบกราวด์ร่วม

### 4.2 ทิศทางขั้วต่อในแต่ละบอร์ด:
* **Master มี 3 ทิศ:** **North, East, South** (เว้นด้าน **West** ไว้สำหรับช่องรับไฟหลักและพอร์ต USB สื่อสารกับคอมพิวเตอร์)
* **Slave มีครบ 4 ทิศ:** **North, East, South, West**

### 4.3 สถาปัตยกรรมการรับ-ส่งสัญญาณ (Directional RX & Shared TX):
บนแต่ละ Tile:
* มีขา **LINK_RX แยกอิสระ 4 ทิศทาง:** `LINK_RX_N`, `LINK_RX_E`, `LINK_RX_S`, `LINK_RX_W` ทำให้ระบุทิศทางของแพ็กเก็ตที่ส่งเข้ามาได้อย่างแม่นยำ
* มีขา **LINK_TX หนึ่งเส้นใช้ร่วมกันทุกด้าน:** เมื่อ Tile สั่งส่งข้อมูล ข้อมูลจะบรอดคาสต์ออกไปยังเพื่อนบ้านทุกด้านพร้อมกัน

---

## 5. รูปแบบแพ็กเก็ตและการป้องกันข้อมูลวนซ้ำ (Packet Architecture & Loop Prevention)

เนื่องจากขา `LINK_TX` เป็นสายร่วมที่กระจายสัญญาณออกทุกด้านพร้อมกัน Packet จึงต้องมีโครงสร้างข้อมูลกำกับ:

```text
+-----------+----------------+-----------------+--------------+-----+-----+--------+
| Source ID | Destination ID | Sequence Number | Frame Number | TTL | CRC | Data.. |
+-----------+----------------+-----------------+--------------+-----+-----+--------+
```

* **Source ID / Destination ID:** ระบุโหนดต้นทางและปลายทาง (หรือ Broadcast ID)
* **Sequence Number:** ลำดับของแพ็กเก็ต
* **Frame Number:** เลขเฟรมภาพสำหรับการซิงก์แสดงผล
* **TTL (Time-To-Live):** ลดค่าลงทีละ 1 ทุกครั้งที่ส่งต่อ เพื่อตัดวงจรแพ็กเก็ตตกค้าง
* **CRC:** ตรวจสอบความถูกต้องของข้อมูล
* **การป้องกันข้อมูลวนซ้ำ (Deduplication):** แต่ละ Tile จะจดจำ Sequence Number ล่าสุด และ**ไม่ประมวลผลแพ็กเก็ตเดิมซ้ำ** ช่วยป้องกันปัญหาข้อมูลวนลูปในกรณีที่มีเส้นทางเชื่อมต่อหลายทาง

---

## 6. เส้นทางข้อมูลที่ถูกต้อง (Data Flow Pipeline)

```text
Unity
  └──► ESP32 (Master)
         └──► ATtiny1604 (Master)
                ├──► ขับไฟ LED 64 ดวงบน Master
                └──► Magnetic Link (LINK_TX)
                       └──► ATtiny1604 (Slave Tile)
                              ├──► ขับไฟ LED 64 ดวงบน Slave
                              └──► ส่งต่อไปยัง Slave ถัดไป
```

* **Master Tile:** Unity $\rightarrow$ ESP32 $\rightarrow$ ATtiny Master $\rightarrow$ LED Master
* **Slave Tiles:** Unity $\rightarrow$ ESP32 $\rightarrow$ ATtiny Master $\rightarrow$ Magnetic Link $\rightarrow$ ATtiny Slave $\rightarrow$ ส่งต่อไปยัง Slave ถัดไป
* **ESP32 ไม่ส่งข้าม ATtiny Master ไปยังบอร์ดลูกโดยตรง**

---

## 7. สถาปัตยกรรมระบบไฟ (Power Architecture)

หลักการจ่ายไฟของระบบกำหนดไว้ดังนี้:

```text
Power Input
   └──► Master Protection
          └──► VBUS (ส่งผ่านขั้วต่อแม่เหล็กไปยังทุกแผ่น)
                 └──► Local Power Converter ในแต่ละ Tile
                        └──► 5V คงที่สำหรับขับ LED และเลี้ยง ATtiny1604
```

> [!IMPORTANT]
> **ข้อกำหนดในการออกแบบภาคจ่ายไฟ:**
> 1. **ใช้ชื่อบัสกลางว่า `VBUS`:** ไม่ส่งตรง 5V ผ่านทุกบอร์ดในระยะยาว เพื่อรองรับการจ่ายแรงดันที่สูงขึ้น (เช่น 9V–12V) แล้วใช้ Local Step-Down Converter ลดเป็น 5V ภายในแต่ละแผ่น เพื่อลดแรงดันตกคร่อม (Voltage Drop) และลดกระแสบนพินแม่เหล็ก
> 2. **ยังไม่ล็อกเบอร์อุปกรณ์และแรงดัน VBUS ในสเปกหลัก:** รายละเอียดของแรงดัน VBUS, ชนิดคอนเนคเตอร์ไฟเข้า, ชนิดวงจรแปลงไฟ, อุปกรณ์วัดกระแส, และพิกัด TVS (ห้ามล็อก SMAJ5.0A หาก VBUS มีโอกาสสูงกว่า 5V) จะสรุปหลังจากการทดสอบพิกัดกระแสจริงของหัวแม่เหล็กและกำลังวัตต์สูงสุดของ LED ทั้งระบบ
> 3. **ตัวต้านทาน CC 5.1kΩ:** ทำหน้าที่ระบุสถานะ Sink บนพอร์ต USB Type-C พื้นฐานเท่านั้น ไม่ได้รับรองการจ่ายกระแส 5V/3A และไม่ใช่ระบบเจรจาแรงดัน USB-PD

---

## 8. การจัดการระดับแรงดันสัญญาณ (Logic Level Translation)

* **หาก ATtiny1604 เลี้ยงด้วยไฟ 5V:** สัญญาณ GPIO เอาต์พุต (รวมถึงขาขับ LED `LED_DATA`) จะมีระดับแรงดันเป็น 5V CMOS Logic อยู่แล้ว จึงสามารถขับขา DIN ของโมดูล WS2812B ได้โดยตรงโดยไม่ต้องใช้ Level Shifter 3.3V$\rightarrow$5V หน้า LED
* **จุดที่ต้องแปลงระดับแรงดัน (Level Shifting):** คือส่วนเชื่อมต่อระหว่าง **ESP32 (แรงดันสัญญาณ 3.3V)** กับ **ATtiny1604 (แรงดันสัญญาณ 5V)** บนบอร์ด Master เพื่อให้การส่งข้อมูล UART ภายในทำงานได้อย่างถูกต้องและปลอดภัย

---

## 9. เอกสารอ้างอิงเฉพาะบอร์ด (Detailed Documentation Links)

* 📖 **[เอกสารวงจรและฮาร์ดแวร์บอร์ดแม่ (Master Controller Board README)](hardware/master/README.md)**
* 📖 **[เอกสารวงจรและฮาร์ดแวร์บอร์ดลูก (Slave Expansion Board README)](hardware/slave/README.md)**