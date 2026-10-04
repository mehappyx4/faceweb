# 🎯 ReconFace: ระบบค้นหาใบหน้าแบบย้อนกลับและระบบรู้จำใบหน้า

ReconFace คือระบบ OSINT แบบครบวงจรและกระจายการทำงาน สำหรับรับรูปภาพจากแหล่งข้อมูลบนเว็บ สกัดเวกเตอร์ใบหน้าขนาด 512 มิติที่ผ่านการทำให้เป็นมาตรฐานด้วย **InsightFace ArcFace** จัดทำดัชนีเวกเตอร์ลงใน **Qdrant Vector DB** โดยใช้ดัชนี HNSW และระยะห่างแบบ Cosine พร้อมให้บริการ REST API ที่รวดเร็วสำหรับการค้นหาใบหน้าแบบย้อนกลับ

---

## 🏗️ สถาปัตยกรรมระบบ

```mermaid
flowchart LR
    A[ผู้ใช้ / นักวิเคราะห์ OSINT] -->|อัปโหลดรูปภาพ| B[FastAPI REST API]
    A -->|ส่ง URL| B
    B -->|งานนำเข้าแบบอะซิงโครนัส| C[Redis Queue]
    C --> D[Celery Workers]
    D -->|ดาวน์โหลดและสกัดใบหน้า| E[InsightFace ArcFace]
    D -->|จัดเก็บไฟล์ต้นฉบับและภาพใบหน้า| F[MinIO S3 Storage]
    D -->|จัดทำดัชนีเวกเตอร์ 512 มิติ| G[Qdrant Vector DB]
    B -->|ค้นหาเวกเตอร์ 512 มิติ| G
    G -->|ผลลัพธ์ Top-K และข้อมูลกำกับ| B
```

## ⚡ เริ่มต้นใช้งานอย่างรวดเร็ว

### 1. รันด้วย Docker Compose

```bash
docker compose up -d --build
```

- เอกสาร API แบบโต้ตอบด้วย Swagger: http://localhost:8000/docs
- หน้าจอค้นหาเว็บและระบบ AI Recon: http://localhost:8000/ui

### 2. การเร่งความเร็วด้วยฮาร์ดแวร์และการตั้งค่าข้ามแพลตฟอร์ม

ReconFace รองรับการเร่งความเร็วด้วยฮาร์ดแวร์โดยอัตโนมัติในหลายแพลตฟอร์ม:

- **Windows / Linux (NVIDIA GPU):** ใช้ CUDA (`CUDAExecutionProvider`) หรือ DirectML (`DmlExecutionProvider`)
- **macOS (Apple Silicon M1/M2/M3/M4):** ใช้ Apple Neural Engine / GPU (`CoreMLExecutionProvider`) หรือ CPU ประสิทธิภาพสูงบนสถาปัตยกรรม ARM64

### 🍏 การตั้งค่า macOS (รูปแบบ Hybrid ที่แนะนำ)

Docker Desktop บน macOS ไม่สามารถส่งต่อ Apple GPU/Metal เข้าไปยัง Linux containers ได้โดยตรง เพื่อประสิทธิภาพสูงสุด ให้รันฐานข้อมูลด้วย Docker และรัน AI engine บนเครื่องโดยตรง:

```bash
# 1. เริ่มต้น Qdrant, Redis และ MinIO ด้วย Docker
docker compose up -d qdrant redis minio

# 2. ติดตั้ง Xcode command line tools และ cmake
#    (จำเป็นสำหรับการ build InsightFace)
xcode-select --install
brew install cmake

# 3. สร้าง virtual environment และติดตั้ง ReconFace
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# 4. เริ่มต้น API Server
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 🪟 Windows / Linux ที่ใช้ NVIDIA GPU

```bash
pip install -e ".[dev,gpu]"
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

## 📡 จุดเชื่อมต่อ API

### 1. การค้นหาใบหน้าแบบย้อนกลับ

- **Endpoint:** `POST /api/v1/search`
- **Content-Type:** `multipart/form-data`
- **พารามิเตอร์:**
  - `file`: ไฟล์รูปภาพประเภท JPG, PNG หรือ WEBP
  - `face_index`: ดัชนีใบหน้าที่ต้องการค้นหา (ค่าเริ่มต้น: `0` = ใบหน้าที่มีขนาดใหญ่ที่สุด)
  - `top_k`: จำนวนผลลัพธ์ที่ต้องการ (ค่าเริ่มต้น: `10`)
  - `score_threshold`: ค่าขีดจำกัดความคล้ายคลึงแบบ Cosine (ค่าเริ่มต้น: `0.6`)

### 2. การนำเข้ารูปภาพจาก URL

- **Endpoint:** `POST /api/v1/ingest`
- **Body:**

```json
{
  "url": "https://target-source.org/photo.jpg",
  "metadata": {
    "case_id": "OSINT-2026-001",
    "source": "web_archive"
  }
}
```

### 3. การตรวจสอบสถานะระบบ

- **Endpoint:** `GET /api/v1/health`
- ส่งกลับสถานะของ API, collection ใน Qdrant Vector DB และระบบจัดเก็บข้อมูล

## 🧪 การทดสอบ

```bash
pytest tests/ -v
```

การทดสอบทั้ง unit tests และ integration tests จะทำงานด้วย Qdrant แบบ in-memory และชุดรูปภาพทดสอบที่สร้างขึ้นสำหรับการทดสอบโดยเฉพาะ
