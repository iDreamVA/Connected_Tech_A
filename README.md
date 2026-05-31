# 🚪 Person Door Counter — ระบบนับคนเข้า-ออกประตูด้วย AI

ระบบตรวจจับและนับจำนวนคนเข้า-ออกประตูจากวิดีโอ CCTV อัตโนมัติ
โดยใช้ YOLO ตรวจจับตัวคน + Color Histogram Re-ID จดจำตัวตน + Door Zone สี่เหลี่ยมกรอบประตู

---

## ✨ ฟีเจอร์

- **ตรวจจับคนเข้า-ออกประตู** ด้วยกรอบสี่เหลี่ยม (Door Zone) — นับเฉพาะคนที่เดินผ่านพื้นที่ประตูเท่านั้น ไม่นับคนที่ยืนอยู่ในห้อง
- **จดจำตัวตนคน (Re-ID)** ด้วย Color Histogram — จำคนที่เคยเข้ามาแล้วกลับมาซ้ำได้ แสดงสถานะ `RETURN`
- **แสดงผลแยกสี** — เข้า (น้ำเงิน) / ออก (ส้ม) / กลับมาซ้ำ (ม่วง)
- **เบาและเร็ว** — ประมวลผลวิดีโอ 85 วินาที เสร็จใน ~1-3 นาที ไม่ต้องใช้ GPU สำหรับ Re-ID

---

## 🛠 เทคโนโลยีที่ใช้

| ส่วน | เทคโนโลยี | หน้าที่ |
|---|---|---|
| Detection + Tracking | YOLOv5n (Ultralytics) | ตรวจจับตัวคน + ติดตามการเคลื่อนไหว |
| Feature Extraction | Color Histogram (HSV) | ดึงลักษณะเด่นเพื่อจดจำตัวตน |
| Matching | Cosine Similarity (NumPy) | เปรียบเทียบความคล้ายระหว่างคน |
| Door Detection | Rectangle Zone + Inside/Outside | ตรวจจับการเข้า-ออกพื้นที่ประตู |

---

## 📂 โครงสร้างโปรเจกต์

```
Connected_Tech_A/
├── README.md              ← ไฟล์นี้
├── entrance.mov           ← วิดีโอต้นฉบับ
├── output.mp4             ← วิดีโอผลลัพธ์ (หลังรัน)
└── Untitled30.ipynb       ← Notebook หลัก
    ├── Cell 1: ติดตั้งไลบรารี (ultralytics)
    ├── Cell 2: Feature Extractor — Color Histogram (HSV)
    └── Cell 3: Pipeline หลัก — YOLO + Door Zone + Re-ID
```

---

## 🚀 วิธีใช้

### 1. เตรียมสภาพแวดล้อม
```bash
pip install ultralytics opencv-python numpy
```

### 2. เปิด Notebook
เปิดไฟล์ `Untitled30.ipynb` ใน Jupyter Notebook / VS Code / Google Colab

### 3. รันตามลำดับ
1. **Cell 1** — ติดตั้งไลบรารี
2. **Cell 2** — สร้าง Feature Extractor (Color Histogram)
3. **Cell 3** — รัน Pipeline หลัก → ได้ไฟล์ `output.mp4`

### 4. ดูผลลัพธ์
เปิดไฟล์ `output.mp4` จะเห็น:
- สี่เหลี่ยมกรอบประตูสีแดง (semi-transparent)
- กรอบสีรอบตัวคนแยกตามสถานะ
- ตัวเลข ENTER / EXIT / UNIQUE PEOPLE / RETURN ที่มุมบนซ้าย

---

## ⚙️ ตั้งค่าที่ปรับได้

### ตำแหน่งกรอบประตู (Door Zone)
```python
DOOR_X1, DOOR_Y1 = 100, 50     # มุมซ้ายบน ของกรอบประตู
DOOR_X2, DOOR_Y2 = 550, 950    # มุมขวาล่าง ของกรอบประตู
```
> ปรับพิกัดให้ตรงกับตำแหน่งประตูในวิดีโอของคุณ

### ประสิทธิภาพ
```python
YOLO_IMGSZ = 480       # ขนาดภาพ YOLO (320/416/480/640) — น้อยกว่า = เร็วกว่า
PROCESS_EVERY_N = 3    # ประมวลผลทุก N เฟรม (1=ทุกเฟรม) — มากกว่า = เร็วกว่า
```

### ความแม่นยำ Re-ID
```python
SIMILARITY_THRESHOLD = 0.6   # ค่าความคล้ายขั้นต่ำ (0-1) — สูงกว่า = เข้มงวดกว่า
COOLDOWN_FRAMES = 15         # รอ N เฟรมหลัง crossing ก่อนนับใหม่ (ป้องกันสั่น)
```

---

## 🎨 สีกรอบบนวิดีโอผลลัพธ์

| สี | Label | ความหมาย |
|---|---|---|
| 🟢 เขียว | `T#` | ตรวจจับได้ แต่ยังไม่เข้า Door Zone |
| 🔵 น้ำเงิน | `IN P#` | อยู่ภายใน Door Zone (เข้ามาแล้ว) |
| 🟠 ส้ม | `OUT P#` | ออกจาก Door Zone แล้ว |
| 🟣 ม่วง | `RETURN P#` | คนเดิมที่เคยเข้าแล้วกลับมาอีกครั้ง |

---

## 🔧 วิธีทำงาน (Architecture)

```
วิดีโอ input
    │
    ▼
┌─────────────────┐
│  YOLOv5n Track  │  ← ตรวจจับ + ติดตามตัวคนทุกคนในเฟรม
│  (ทุก 3 เฟรม)   │
└────────┬────────┘
         │ Bounding Box + Track ID ของแต่ละคน
         ▼
┌─────────────────────┐
│  Door Zone Check    │  ← ตรวจจุดศูนย์กลางคน อยู่ใน/นอก สี่เหลี่ยมประตู
│  (สี่เหลี่ยมผืนผ้า) │
└────────┬────────────┘
         │ ถ้าเข้า/ออก zone → crossing event
         ▼
┌─────────────────────┐
│  Re-ID Matching     │  ← ดึง Color Histogram → เทียบกับฐานข้อมูล
│  (Color Histogram)  │     เหมือน = คนเดิม / ไม่เหมือน = คนใหม่
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  นับ + แสดงผล       │  ← ENTER / EXIT / RETURN / UNIQUE
│  เขียน output.mp4   │
└─────────────────────┘
```

---

## 📋 ข้อกำหนด (Requirements)

- Python 3.8+
- ultralytics
- opencv-python
- numpy

### รันบน Google Colab
วางไฟล์ `entrance.mov` ใน `/content/` แล้วรัน Notebook ได้เลย (ใช้ GPU ได้แต่ไม่จำเป็น)

### รันบนเครื่อง Local
วางไฟล์วิดีโอไว้ในโฟลเดอร์เดียวกับ Notebook แล้วแก้ `video_path` เป็นชื่อไฟล์ของคุณ
