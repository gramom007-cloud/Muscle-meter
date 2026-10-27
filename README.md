# Muscle Meter Live

เว็บแอปอ่านค่าแรงจากเครื่องวัดแรงกล้ามเนื้อ (Muscle Meter / handheld dynamometer) ผ่าน **Web Bluetooth** แสดงค่าแรงแบบ real-time และวิเคราะห์ **RFD (Rate of Force Development)** สำหรับงานกายภาพบำบัด ฝึกกีฬา และวิจัย

ทั้งแอปอยู่ในไฟล์เดียว: [`index.html`](index.html) (HTML + CSS + JavaScript ล้วน ไม่มี library ภายนอก ไม่ต้อง build)

## ความสามารถ

- เชื่อมต่อเครื่องผ่าน Bluetooth Low Energy แล้วแสดงแรงสด (เฉลี่ยต่อ packet ในโหมด LIVE)
- หน่วยตามเครื่องอัตโนมัติ (kg / lb จาก mode byte) และแสดงค่า **Newton** คู่กันเสมอ
- Peak force + ปุ่ม Reset Peak, ปุ่ม Tare (ตั้งศูนย์)
- กราฟแรงตามเวลาย้อนหลัง 10 วินาที พร้อมเส้น Peak, จุด onset และจุด peak ของการทดสอบล่าสุด
- ตรวจจับการทดสอบอัตโนมัติ และคำนวณ:
  - Peak Force, Time to 80% Peak, Time to Peak (TTP)
  - RFD 0–50 / 0–100 / 0–150 / 0–200 ms, Peak RFD, RFD 20–80%
  - Force @ 50 / 100 / 150 ms, Impulse 0–100 / 0–200 ms
- ตารางประวัติการทดสอบ + Export CSV
- ดูกราฟแรงของแต่ละครั้ง (ทั้งครั้ง หรือขยาย 300 ms แรกพร้อมแถบช่วง RFD) และตั้ง "ไม่นับครั้งนี้" ได้ (ครั้งที่ไม่นับจะไม่ถูกส่งออก CSV)
- ติดป้ายข้างซ้าย/ขวาให้แต่ละครั้ง และเทียบ LSI (Peak Force, RFD 0–50/100/200, Peak RFD) เลือกข้างที่บาดเจ็บและใช้ค่าดีที่สุดหรือค่าเฉลี่ยได้
- โหมดทดสอบแบบมีขั้นตอน: รอแรงนิ่ง นับถอยหลัง 3-2-1 พร้อมเสียง สัญญาณออกแรง บอกให้คลายแรง นับเวลาพัก และสลับข้างให้อัตโนมัติ (ครั้งที่ออกแรงก่อนสัญญาณถูกตั้งเป็นไม่นับ) โหมดนี้แค่อ่านสถานะของตัวตรวจจับเดิม ไม่ได้เปลี่ยนการตรวจจับหรือสูตร
- คู่มือการทดสอบและนิยามของทุกค่า (ภาษาไทย) พร้อมแหล่งอ้างอิง

## วิธีใช้งาน

Web Bluetooth ใช้ได้เฉพาะ **Chrome / Edge** (Windows, macOS, Android) และต้องเปิดผ่าน **HTTPS** หรือ `localhost`

- **บนคอม:** เปิด `index.html` ใน Chrome หรือ Edge แล้วกด Connect Bluetooth
- **บนมือถือ Android:** เปิดผ่านลิงก์ GitHub Pages (HTTPS)
- **iPhone:** Safari ไม่รองรับ Web Bluetooth ต้องใช้เบราว์เซอร์ Bluefy

### เปิดใช้ GitHub Pages

Settings → Pages → Source: *Deploy from a branch* → Branch: `main` / `(root)` แล้วจะได้ลิงก์ `https://<username>.github.io/<repo>/`
(Pages ของ repo แบบ Private ต้องใช้บัญชีแบบเสียเงิน)

---

## โปรโตคอลของเครื่อง (ห้ามเปลี่ยนโดยไม่ทดสอบกับเครื่องจริง)

| รายการ | ค่า |
|---|---|
| Service UUID | `0xFFE0` |
| Characteristic (notify) | `0000ffe1-0000-1000-8000-00805f9b34fb` |
| อัตราข้อมูล | ~24 notifications/s × ~20 เฟรม ≈ **480 Hz** (1 เฟรม ≈ 2 ms) |

แต่ละเฟรมยาว **3 ไบต์**: `[Weight_High, Weight_Low, Mode]`

- **น้ำหนัก:** 16-bit unsigned big-endian ÷ 10 (เช่น `00 32` = 50 → 5.0)
- **Mode byte:** nibble ล่าง = หน่วย, nibble บน = HOLD

| Mode | ความหมาย |
|---|---|
| `0x01` | kg LIVE |
| `0x11` | kg HOLD |
| `0x02` | lb LIVE |
| `0x12` | lb HOLD |

ตัวอย่าง packet (5.0 lb, LIVE): `00-32-02-00-32-02-00-30-02-...`

หมายเหตุ:
- packet อาจเริ่มเยื้อง โค้ดจึงหา offset (0–2) จากตำแหน่ง mode byte ที่ถูกต้อง
- ใน 1 packet อาจมีเฟรม LIVE และ HOLD ปนกัน (ตอนกด HOLD) ใช้โหมดของเฟรม **สุดท้าย**
- ค่า HOLD คือค่าที่เครื่องค้างไว้ ไม่นำไปลงกราฟ ไม่นับ Peak และไม่นำไปคำนวณ RFD

## การคำนวณ RFD (สรุป)

- **เวลา:** คิดจาก *ลำดับเฟรม ÷ ความถี่ของเครื่อง* (ประมาณจากจำนวนเฟรมสะสม) ไม่ใช้เวลาที่ notification มาถึง เพราะ Bluetooth ส่งมาเป็นก้อน
- **ค่าภายในเก็บเป็น Newton** (1 kg = 9.80665 N, 1 lb = 4.4482216 N) แปลงหน่วยตอนแสดงผลเท่านั้น
- **Filter:** Butterworth low-pass อันดับ 2 ที่ 50 Hz กรองไป-กลับ (zero-lag) ใช้เฉพาะตอนวิเคราะห์ กราฟและตัวเลขสดใช้ข้อมูลดิบ
- **Baseline:** ค่าเฉลี่ยและ SD ของแรงช่วง 0.2–0.7 วินาทีก่อนเริ่ม
- **เริ่มบันทึก:** แรง > baseline + 10 N (`TRIGGER_N`)
- **Onset:** เฟรมสุดท้ายที่แรง ≤ baseline + max(3·SD, 2 N) ก่อนจุดเริ่มบันทึก
- **จบการทดสอบ:** แรง < baseline + 5 N, เครื่องเข้า HOLD, หรือนานเกิน 15 วินาที
- **RFD 0–X ms** = (F(X) − F(onset)) / X, **Peak RFD** = ความชันสูงสุดในช่วงละ 20 ms
- **RFD 20–80%** = 0.6 × Peak สุทธิ / (t80 − t20)

ค่าตั้งต้นทั้งหมดอยู่ส่วนบนของ `<script>` (`TRIGGER_N`, `ONSET_MIN_N`, `RFD_WINDOWS_MS`, `FILTER_CUTOFF_HZ` ฯลฯ)

---

## โจทย์สำหรับคนพัฒนา UI

UI ตอนนี้ทำงานครบแต่หน้าตายังเรียบมาก ต้องการปรับให้สวยและใช้งานง่ายขึ้น

**แก้ได้อิสระ**
- HTML/CSS ทั้งหมด: layout, สี, ฟอนต์, การจัดกลุ่มค่า, dark mode, การแสดงผลบนมือถือ
- การวาดกราฟในฟังก์ชัน `draw()` หรือเปลี่ยนไปใช้ library กราฟ
- เปลี่ยนเป็น framework (React, Vue ฯลฯ) ได้ ถ้าย้ายตรรกะส่วน "ห้ามแตะ" ไปแบบไม่เปลี่ยนพฤติกรรม

**ห้ามเปลี่ยนพฤติกรรม** (ถ้าจำเป็นต้องแก้ ให้ทดสอบกับเครื่องจริงหรือข้อมูลจำลองก่อน)
- `parseData()`: การอ่าน packet, offset, mode byte, การแปลงหน่วย
- การประมาณ `fsEst` และการกำหนดเวลาให้แต่ละเฟรม
- `processLiveFrame()`, `finishTrial()`, `analyzeTrial()`, `lowPassZeroLag()`: การตรวจจับการทดสอบและสูตร RFD

**ไอเดียที่อยากได้**
- หน้าจอหลักอ่านง่ายจากระยะไกล (ตัวเลขแรงใหญ่ ชัด)
- โหมดทดสอบแบบมีขั้นตอน: นับถอยหลัง เสียงสัญญาณ เปรียบเทียบซ้าย/ขวา (LSI)
- บันทึกข้อมูลผู้ถูกทดสอบและประวัติ ส่งออกรายงาน PDF

### ทดสอบโดยไม่มีเครื่อง

เปิดหน้าเว็บแล้วรันใน DevTools Console เพื่อป้อนข้อมูลจำลอง (480 Hz, แรงเพิ่ม 1000 N/s นาน 300 ms):

```js
const mk = (fr) => { const b = new Uint8Array(fr.length * 3); fr.forEach(([w, m], i) => { b[i*3] = w >> 8; b[i*3+1] = w & 255; b[i*3+2] = m; }); return new DataView(b.buffer); };
const LB = 4.4482216, FS = 480, t0 = 720;
const f = (i) => { const rest = 2 * LB, t = (i - t0) / FS;
  if (i < t0) return rest; if (t < 0.3) return rest + 1000 * t; if (t < 1.3) return rest + 300; return rest; };
let i = 0;
for (let p = 0; p < 72; p++) setTimeout(() => { const fr = [];
  for (let k = 0; k < 20; k++, i++) fr.push([Math.round(f(i) / LB * 10), 0x02]); parseData(mk(fr)); }, p * 1000 / 24);
```

ผลที่ควรได้: RFD ทุกช่วง ≈ 1000 N/s, Force @ 100 ms ≈ 100 N, Time to 80% Peak ≈ 240 ms

## แหล่งอ้างอิง

- VALD Health (2026). [Understanding Rate of Force Development](https://valdhealth.com/news/understanding-rate-of-force-development)
- Aagaard P. et al. (2002). *J Appl Physiol*, 93(4), 1318–1326.
- Maffiuletti N.A. et al. (2016). Rate of force development: physiological and methodological considerations. *Eur J Appl Physiol*, 116(6), 1091–1116.
- Haff G.G. et al. (2015). *J Strength Cond Res*, 29(2), 386–395.

## โครงสร้างไฟล์

```
index.html                   แอปทั้งหมด
README.md                    ไฟล์นี้
legacy/index-original.html   เวอร์ชันแรก (เก็บไว้อ้างอิง)
legacy/index-v1.html         เวอร์ชันก่อนออกแบบ UI ใหม่ (หน้าตาเดิม ตรรกะเดียวกัน)
```
