1) แต่ละไฟล์ YAML ทำงานร่วมกันอย่างไร

      **.github/workflows/ci.yml**
      ไฟล์ Workflow ของ GitHub Actions ทำงานอัตโนมัติเมื่อมี push หรือ pull request ไปที่ main
      แบ่งเป็น 3 งานหลัก:

      **1.Lint YAML ด้วย yamllint เพื่อตรวจ syntax/รูปแบบของไฟล์ .yml/.yaml ทั้งโปรเจกต์**
      **2.ตรวจ Docker Compose ด้วย docker compose config -q เพื่อยืนยันว่า docker-compose.yml ถูกต้อง**
      **3.ตรวจ Kubernetes manifests (offline) ด้วย kubeconform (ไม่ต้องต่อ cluster) ให้แน่ใจว่า k8s/*.yml ถูก schema**

      **docker-compose.yml**
      ไฟล์สำหรับการรันแบบเครื่องเดียว (Dev/Test) เช่น web (nginx) เปิดพอร์ต 8080:80 ใช้เพื่อทดสอบแอป/คอนฟิกอย่างรวดเร็วก่อนขึ้น Kubernetes

      **k8s/deployment.yml**
      บอกให้ Kubernetes สร้าง Pod ของ nginx ตาม template และจำนวน replicas ที่ตั้งไว้ (เช่น 2) ถ้า Pod ตายจะถูกสร้างใหม่อัตโนมัติ

      **k8s/service.yml**
      สร้าง Service เพื่อไปยัง Pod ที่ label ตรงกับ selector
      ใช้ชนิด NodePort เพื่อให้เข้าถึงจากนอกคลัสเตอร์ผ่าน <NodeIP>:30080

      **ความสัมพันธ์: Deployment สร้าง Pod พร้อม label app: nginx → Service เลือก Pod เหล่านี้ด้วย selector: app: nginx แล้วทำ load-balance ให้**

2) ลำดับขั้นตอนเมื่อ Push โค้ดไปที่ GitHub (Workflow → Docker → Kubernetes)

      **Trigger Workflow**
      push/PR ไป main → GitHub Actions เรียกใช้ไฟล์ .github/workflows/ci.yml

      **Checkout**
      ดึงซอร์สโค้ดมาให้ runner ใช้งาน

      **Lint YAML**
      รัน yamllint . ตรวจ syntax ของทุก YAML (หากผิดจะเตือน)

      **ตรวจ Docker Compose**
      docker compose config -q รวม/เช็คไฟล์ compose ว่าถูกต้องโดยไม่ต้องขึ้นคอนเทนเนอร์

      **ตรวจ Kubernetes (offline)**
      kubeconform -strict -summary k8s/ ตรวจ schema ของ deployment.yml และ service.yml โดยไม่ต้องมี cluster

      **ขั้นตอนนี้เป็น Pre-flight checks** เพื่อให้มั่นใจก่อน deploy จริง 
3) โครงสร้าง Project

      ci.yml : pipeline ตรวจคุณภาพไฟล์อัตโนมัติ (CI)
      docker-compose.yml : ใช้รัน/ทดสอบบนเครื่องเดียว
      คำสั่งตัวอย่าง: docker compose up -d → เปิดเว็บที่ http://localhost:8080
      k8s/ : ไฟล์สำหรับ deploy บน Kubernetes
      deployment.yml : ระบุ image, containerPort, replicas, labels
      service.yml : ระบุชนิด Service, port/targetPort/nodePort และ selector ให้ตรงกับ Deployment

4) Diagram ความสัมพันธ์ระหว่าง CI/CD Pipeline, Docker Compose และ Kubernetes
   
**4.1 ภาพรวม Pipeline**
flowchart TD
  A[Developer\nPush/PR to main] --> B[GitHub Actions\nci.yml]
  B --> C[Lint YAML\nyamllint]
  B --> D[Check Docker Compose\ncompose config -q]
  B --> E[Validate K8s (offline)\nkubeconform]
  E --> F[Result Summary\n(Artifacts/Logs)]

**4.2 ความสัมพันธ์ Runtime: Compose vs Kubernetes**
flowchart LR
  subgraph Local (Dev/Test)
    DC[docker-compose.yml\nweb: nginx -> 8080:80]
  end

  subgraph Cluster (Prod/Stage)
    DEP[Deployment\nreplicas=2\nlabels: app=nginx]
    SVC[Service NodePort\nselector: app=nginx\nport 80 -> nodePort 30080]
    DEP -->|create| POD1[Pod#1]
    DEP -->|create| POD2[Pod#2]
    SVC --> POD1
    SVC --> POD2
  end

  DC == ใช้ทดสอบโลคัล ==> Developer
  SVC == เข้าใช้งานผ่าน ==> User[(NodeIP:30080)]

**วิธีรัน**

**บนเครื่อง (Docker Compose):**

docker compose config -q   # ตรวจคอนฟิก
docker compose up -d
# เปิด http://localhost:8080


ตรวจ K8s manifests แบบออฟไลน์ (เหมือนใน CI):

# ติดตั้ง kubeconform
curl -L https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz \
| tar -xz kubeconform && sudo mv kubeconform /usr/local/bin/

kubeconform -strict -summary -ignore-missing-schemas k8s/

**หมายเหตุ**

ถ้าใช้ cluster จริงและอยาก dry-run ด้วย kubectl ให้ตั้งค่า KUBECONFIG ให้ต่อคลัสเตอร์ได้ แล้วค่อยใช้
kubectl apply --dry-run=server -f k8s/

**เวอร์ชันนี้ใช้ kubeconform ใน CI เพื่อให้ตรวจได้แม้ไม่มีคลัสเตอร์**
