# สรุปเนื้อหา Docker Compose สำหรับ CI/CD

> **เอกสารสรุปสำหรับการเตรียมทำ Lab CI/CD ด้วย Docker Compose**  
> รวบรวมความรู้ที่จำเป็นสำหรับการใช้งาน Docker Compose ในการพัฒนาและ deploy applications

---

## 📚 สารบัญ

1. [Docker Compose คืออะไร](#section-1)
2. [โครงสร้างไฟล์ Docker Compose](#section-2)
3. [Services Configuration](#section-3)
4. [Networks และ Volumes](#section-4)
5. [Health Checks](#section-5)
6. [Environment Variables](#section-6)
7. [Dockerfile Best Practices](#section-7)
8. [Multi-stage Builds](#section-8)
9. [คำสั่ง Docker Compose](#section-9)
10. [Debugging และ Troubleshooting](#section-10)
11. [Best Practices](#section-11)
12. [ตัวอย่างการใช้งานจริง](#section-12)

---

<a name="section-1"></a>
## 1. Docker Compose คืออะไร?

### คำจำกัดความ

**Docker Compose** เป็นเครื่องมือที่ใช้จัดการแอปพลิเคชันที่ประกอบด้วย **หลาย Docker containers** โดยใช้ไฟล์ YAML เดียวในการกำหนดค่าทั้งหมด

### ความสำคัญสำหรับ CI/CD

- ✅ **จัดการหลาย containers** - รัน app, database, cache พร้อมกัน
- ✅ **ง่ายต่อการใช้งาน** - คำสั่งเดียวสำหรับ start/stop ทุก services
- ✅ **Reproducible** - สามารถ setup environment เดียวกันได้ทุกครั้ง
- ✅ **Development to Production** - ใช้ได้ทั้ง development และ production
- ✅ **Integration กับ CI/CD** - ทำงานร่วมกับ GitHub Actions ได้ดีมาก
- ✅ **Isolation** - แต่ละ service แยกกัน ไม่กระทบกัน

### เปรียบเทียบ: ไม่ใช้ vs ใช้ Docker Compose

**ไม่ใช้ Docker Compose (ยุ่งยาก):**
```bash
# ต้องรัน container ทีละตัว - เสี่ยงพลาด
docker network create app-network

docker run -d --name postgres \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16

docker run -d --name redis \
  --network app-network \
  redis:7

docker run -d --name app \
  --network app-network \
  -p 5000:5000 \
  -e DATABASE_URL=postgresql://postgres:secret@postgres/mydb \
  -e REDIS_URL=redis://redis:6379 \
  myapp
```

**ใช้ Docker Compose (ง่าย, สะดวก):**
```bash
# คำสั่งเดียวจบ!
docker compose up -d

# หยุดทุกอย่าง
docker compose down
```

---

<a name="section-2"></a>
## 2. โครงสร้างไฟล์ Docker Compose

### โครงสร้างพื้นฐาน

```yaml
# Modern Docker Compose - ไม่ต้องระบุ version
# ใช้ Compose Specification V2 อัตโนมัติ

services:           # กำหนด containers ต่างๆ
  service1:
    # configuration
  service2:
    # configuration

networks:           # กำหนดเครือข่าย (optional)
  network1:
    driver: bridge

volumes:            # กำหนด volumes (optional)
  volume1:
    driver: local
```

### ตัวอย่าง: Flask Todo Application

```yaml
services:
  # =============================
  # Database Service
  # =============================
  db:
    image: postgres:16-alpine
    container_name: todo_postgres
    environment:
      POSTGRES_DB: todo_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  # =============================
  # Application Service
  # =============================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: todo_app
    environment:
      FLASK_ENV: ${FLASK_ENV}
      DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@db:5432/todo_dev
      SECRET_KEY: ${SECRET_KEY}
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app  # Development: hot reload
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

---

<a name="section-3"></a>
## 3. Services Configuration

### ตัวเลือกสำคัญทั้งหมด

| Key | คำอธิบาย | ตัวอย่าง |
|-----|---------|---------|
| `image:` | Docker image ที่จะใช้ | `postgres:16-alpine` |
| `build:` | Build จาก Dockerfile | `build: .` |
| `container_name:` | ชื่อ container | `flask_app` |
| `ports:` | Port mapping | `"5000:5000"` |
| `environment:` | Environment variables | `FLASK_ENV: development` |
| `env_file:` | โหลด env จากไฟล์ | `.env` |
| `depends_on:` | Service dependencies | `depends_on: - db` |
| `volumes:` | Mount volumes/directories | `- .:/app` |
| `networks:` | เครือข่าย | `- app-network` |
| `command:` | Override CMD | `flask run --host=0.0.0.0` |
| `restart:` | Restart policy | `unless-stopped` |
| `healthcheck:` | Health check | ดูรายละเอียด section 5 |
| `deploy:` | Resource limits | `limits: cpus: '2'` |

### Build Configuration

```yaml
# แบบง่าย
services:
  app:
    build: .

# แบบละเอียด (แนะนำ)
services:
  app:
    build:
      context: ./app          # ตำแหน่ง build context
      dockerfile: Dockerfile  # ชื่อไฟล์
      target: production      # Multi-stage target
      args:                   # Build arguments
        NODE_VERSION: 18
        PYTHON_VERSION: 3.11
      cache_from:
        - myapp:latest
```

### Ports - 3 วิธี

```yaml
services:
  app:
    ports:
      # 1. Short syntax (แนะนำ)
      - "5000:5000"           # host:container
      - "3000:3000"
      
      # 2. Long syntax (ละเอียด)
      - target: 5000          # Port ใน container
        published: 5000       # Port บน host
        protocol: tcp
        mode: host
      
      # 3. Range
      - "8000-8010:8000-8010"
```

### Depends On - รอให้ Service พร้อม

```yaml
services:
  app:
    depends_on:
      # แบบที่ 1: รอให้ start เท่านั้น
      - db
      - redis
      
      # แบบที่ 2: รอให้ healthy (แนะนำ!)
      db:
        condition: service_healthy
      redis:
        condition: service_started
```

**Conditions มี 3 แบบ:**
- `service_started` - รอให้ container start
- `service_healthy` - รอให้ผ่าน health check ✅
- `service_completed_successfully` - รอให้รันเสร็จ

### Restart Policies

```yaml
services:
  app:
    # no - ไม่ restart (default)
    restart: no
    
    # always - restart เสมอ (แม้ stop manually)
    restart: always
    
    # on-failure - restart เฉพาะเมื่อ exit code != 0
    restart: on-failure
    
    # unless-stopped - restart จนกว่าจะ stop manual (แนะนำ!)
    restart: unless-stopped
```

---

<a name="section-4"></a>
## 4. Networks และ Volumes

### Networks - ทำไมต้องใช้?

**ประโยชน์:**
- ✅ แยก traffic ระหว่าง services
- ✅ ความปลอดภัยสูงขึ้น
- ✅ Service discovery - เรียกกันด้วยชื่อ service
- ✅ Isolation - แยก frontend/backend networks

**ตัวอย่าง: แยก Networks ตาม Security Zones**

```yaml
services:
  # Frontend - เชื่อมต่อ public
  frontend:
    networks:
      - frontend-network
      - backend-network
  
  # Backend - เชื่อมต่อทั้ง frontend และ database
  backend:
    networks:
      - backend-network
      - database-network
  
  # Database - เชื่อมต่อเฉพาะ backend (ปลอดภัย!)
  db:
    networks:
      - database-network

networks:
  frontend-network:
    driver: bridge
  
  backend-network:
    driver: bridge
  
  database-network:
    driver: bridge
    internal: true  # ไม่เชื่อมต่อ internet!
```

**Service Discovery:**
```yaml
# ใน container สามารถเรียกกันด้วยชื่อ service
services:
  app:
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
      #                                     ^^
      #                              ชื่อ service ใช้เป็น hostname
      REDIS_URL: redis://redis:6379
      #                  ^^^^^
      #              ชื่อ service
```

### Volumes - เก็บข้อมูลถาวร

**3 ประเภท:**

**1. Named Volumes (แนะนำสำหรับ Production)**
```yaml
services:
  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data
      #  ^^^^^^^^^^^^ ชื่อ volume

volumes:
  postgres_data:  # กำหนด volume
    driver: local
```

**ข้อดี:**
- เก็บข้อมูลปลอดภัย
- จัดการโดย Docker
- Backup ง่าย

**2. Bind Mounts (แนะนำสำหรับ Development)**
```yaml
services:
  app:
    volumes:
      - ./app:/app              # Source code - hot reload
      - ./config:/etc/config    # Config files
      - ./logs:/var/log/app     # Logs
```

**ข้อดี:**
- แก้ code แล้วเห็นผลทันที
- ไม่ต้อง rebuild image

**3. tmpfs (In-memory, Temporary)**
```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /run
```

**Long Syntax (ละเอียด):**
```yaml
services:
  app:
    volumes:
      # Bind mount
      - type: bind
        source: ./app
        target: /app
        read_only: false
      
      # Named volume
      - type: volume
        source: node_modules
        target: /app/node_modules
      
      # tmpfs
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100000000  # 100MB
```

---

<a name="section-5"></a>
## 5. Health Checks - สำคัญมาก!

### ทำไมต้องมี Health Checks?

- ✅ **รอให้ service พร้อม** - ก่อนเริ่ม services อื่น
- ✅ **Auto-restart** - เมื่อ service unhealthy
- ✅ **Monitoring** - ตรวจสอบสถานะ real-time
- ✅ **Load Balancing** - ส่ง traffic ไปเฉพาะ healthy containers
- ✅ **Prevent Errors** - ไม่ให้ app เชื่อมต่อ DB ที่ยังไม่พร้อม

### Syntax

```yaml
healthcheck:
  test: ["CMD-SHELL", "command"]  # คำสั่งตรวจสอบ
  interval: 30s                   # ตรวจสอบทุก 30 วินาที
  timeout: 10s                    # timeout หลัง 10 วินาที
  retries: 3                      # ลอง 3 ครั้งก่อนถือว่า unhealthy
  start_period: 40s               # รอ 40 วินาทีก่อนเริ่มตรวจสอบ
```

### ตัวอย่าง Health Checks

**PostgreSQL:**
```yaml
db:
  image: postgres:16-alpine
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres -d mydb"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 30s
```

**MySQL:**
```yaml
db:
  image: mysql:8
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Redis:**
```yaml
redis:
  image: redis:7-alpine
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 20s
```

**HTTP Endpoint (Flask/Node.js):**
```yaml
app:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
    # หรือ
    # test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
```

**MongoDB:**
```yaml
mongo:
  image: mongo:7
  healthcheck:
    test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Custom Script:**
```yaml
app:
  healthcheck:
    test: ["CMD-SHELL", "/app/scripts/health-check.sh"]
    interval: 30s
    timeout: 10s
    retries: 3
```

### การใช้กับ depends_on

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy  # รอให้ DB healthy ก่อน!
      redis:
        condition: service_healthy
    # app จะไม่ start จนกว่า db และ redis จะ healthy
```

---

<a name="section-6"></a>
## 6. Environment Variables

### 3 วิธีกำหนด Environment Variables

**1. Inline ใน docker-compose.yml (ไม่แนะนำสำหรับ secrets)**
```yaml
services:
  app:
    environment:
      NODE_ENV: production
      API_URL: https://api.example.com
      DEBUG: false
```

**2. ใช้ไฟล์ .env (แนะนำ!)**

**ไฟล์ `.env`:**
```bash
# Database
POSTGRES_DB=mydb
POSTGRES_USER=postgres
POSTGRES_PASSWORD=SecureP@ssw0rd123

# Application
FLASK_ENV=development
SECRET_KEY=your-secret-key-change-in-production
DEBUG=true

# External APIs
API_KEY=sk-1234567890abcdef
STRIPE_KEY=pk_test_1234567890
```

**docker-compose.yml:**
```yaml
services:
  db:
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  
  app:
    env_file:
      - .env                    # โหลดทั้งไฟล์
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
```

**3. จาก System Environment**
```yaml
services:
  app:
    environment:
      DATABASE_URL: ${DATABASE_URL}  # จาก shell environment
      API_KEY: ${API_KEY}
```

### Security Best Practices

**1. สร้าง `.env.example` เป็น template**
```bash
# .env.example
# Database Configuration
POSTGRES_DB=mydb
POSTGRES_USER=postgres
POSTGRES_PASSWORD=change_this_in_production

# Application
FLASK_ENV=development
SECRET_KEY=generate_random_secret_key
DEBUG=true

# External Services
API_KEY=your_api_key_here
```

**2. เพิ่มใน `.gitignore`**
```bash
# .gitignore
.env
.env.local
.env.*.local
.env.production
.env.staging
```

**3. Generate secure secrets**
```bash
# สร้าง random password
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# หรือใช้ openssl
openssl rand -base64 32
```

### Environment Variable Precedence

```
1. Shell environment (สูงสุด)
2. .env file
3. environment ใน docker-compose.yml
4. Dockerfile ENV (ต่ำสุด)
```

---

<a name="section-7"></a>
## 7. Dockerfile Best Practices

### โครงสร้าง Dockerfile ที่ดี

```dockerfile
# ========================================
# Stage 1: Base Image
# ========================================
FROM python:3.11-slim as base

# Set working directory
WORKDIR /app

# ========================================
# Stage 2: Dependencies (Builder)
# ========================================
FROM base as builder

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    postgresql-client \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements FIRST (for layer caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --user -r requirements.txt

# ========================================
# Stage 3: Production
# ========================================
FROM base as production

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    apt-get update && apt-get install -y --no-install-recommends \
    postgresql-client \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy Python packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Environment variables
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP=run.py

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:5000/api/health || exit 1

# Run with Gunicorn (production server)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "--timeout", "120", "run:app"]
```

### 10 Best Practices

**1. ใช้ Official Images + Alpine/Slim**
```dockerfile
# ✅ Good - เล็ก, ปลอดภัย, เร็ว
FROM python:3.11-slim       # ~50MB
FROM node:18-alpine         # ~40MB
FROM postgres:16-alpine     # ~80MB

# ❌ Bad - ใหญ่, ช้า
FROM ubuntu:latest          # ~77MB + ต้อง install ทุกอย่าง
FROM python:3.11            # ~300MB
```

**2. Layer Caching - เรียงจากน้อยเปลี่ยนไปมากเปลี่ยน**
```dockerfile
# ✅ Good Order
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .      # 1. Dependencies (เปลี่ยนน้อย)
RUN pip install -r requirements.txt
COPY . .                     # 2. Source code (เปลี่ยนบ่อย)

# ❌ Bad Order - ทุก build ต้อง install ใหม่
FROM python:3.11-slim
WORKDIR /app
COPY . .                     # Source code ก่อน - cache พัง!
RUN pip install -r requirements.txt
```

**3. Minimize Layers - รวมคำสั่งด้วย &&**
```dockerfile
# ✅ Good - 1 layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        curl \
    && rm -rf /var/lib/apt/lists/*

# ❌ Bad - 3 layers
RUN apt-get update
RUN apt-get install -y build-essential curl
RUN rm -rf /var/lib/apt/lists/*
```

**4. Cleanup ใน Layer เดียวกัน**
```dockerfile
# ✅ Good - ลบใน layer เดียวกัน
RUN apt-get update && \
    apt-get install -y build-essential && \
    rm -rf /var/lib/apt/lists/*      # ลบทันที

# ❌ Bad - ขยะยังอยู่ใน layer
RUN apt-get update
RUN apt-get install -y build-essential
RUN rm -rf /var/lib/apt/lists/*      # ลบทีหลัง - ไม่ได้ผล!
```

**5. ใช้ .dockerignore**
```
# .dockerignore
.git
.gitignore
.env
.env.*
node_modules
npm-debug.log
README.md
*.md
.vscode
.idea
__pycache__
*.pyc
*.pyo
.pytest_cache
htmlcov
.coverage
dist
build
*.egg-info
.DS_Store
Thumbs.db
```

**6. Multi-stage Builds (ลดขนาด)**
```dockerfile
# Build stage - มี build tools
FROM node:18 as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage - เล็ก, ปลอดภัย
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
```

**7. Non-root User (Security)**
```dockerfile
# สร้าง user
RUN useradd -m -u 1000 appuser

# เปลี่ยนเจ้าของไฟล์
COPY --chown=appuser:appuser . .

# Switch user
USER appuser

# ❌ อย่ารัน container ด้วย root!
```

**8. Health Checks**
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1
```

**9. Set Environment Variables**
```dockerfile
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH=/home/appuser/.local/bin:$PATH
```

**10. Use --no-cache-dir**
```dockerfile
# Python
RUN pip install --no-cache-dir -r requirements.txt

# Node.js
RUN npm ci --only=production && npm cache clean --force
```

---

<a name="section-8"></a>
## 8. Multi-stage Builds

### ทำไมต้องใช้?

- ✅ **ลดขนาด image มาก** - ไม่เก็บ build tools
- ✅ **ความปลอดภัยสูงขึ้น** - ลด attack surface
- ✅ **แยก concerns** - build ≠ runtime
- ✅ **Optimize caching** - แยก dependencies แต่ละ stage

### ตัวอย่าง 1: Node.js Application

```dockerfile
# =============================================
# Stage 1: Dependencies
# =============================================
FROM node:18-alpine AS deps
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# =============================================
# Stage 2: Builder
# =============================================
FROM node:18-alpine AS builder
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (including devDependencies)
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# =============================================
# Stage 3: Runner (Production)
# =============================================
FROM node:18-alpine AS runner
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy production dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy built application from builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD node healthcheck.js || exit 1

# Start application
CMD ["node", "dist/index.js"]
```

**ผลลัพธ์:**
- Builder stage: ~400MB (มี build tools)
- Final image: ~80MB (เฉพาะ runtime) 🎉

### ตัวอย่าง 2: Python Flask Application

```dockerfile
# =============================================
# Stage 1: Builder
# =============================================
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python packages to user directory
RUN pip install --no-cache-dir --user -r requirements.txt

# =============================================
# Stage 2: Runtime (Production)
# =============================================
FROM python:3.11-slim AS runtime

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Install runtime dependencies ONLY
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy Python packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Environment variables
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:5000/api/health || exit 1

# Run with Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "run:app"]
```

### Build Specific Stage

```bash
# Build production (default)
docker build -t myapp:prod .

# Build specific stage
docker build --target builder -t myapp:builder .
docker build --target runtime -t myapp:runtime .

# ใช้กับ docker-compose
docker compose build --target production
```

---


<a name="section-9"></a>
## 9. คำสั่ง Docker Compose

### คำสั่งพื้นฐาน

```bash
# ========================================
# เริ่ม Services
# ========================================

# เริ่มทั้งหมด (foreground - เห็น logs)
docker compose up

# เริ่มแบบ background (detached mode)
docker compose up -d

# Build และเริ่ม
docker compose up --build

# Force recreate containers
docker compose up --force-recreate

# เริ่มเฉพาะ service
docker compose up app
docker compose up db redis

# Scale services
docker compose up -d --scale app=3


# ========================================
# หยุด Services
# ========================================

# หยุดและลบ containers
docker compose down

# หยุด + ลบ volumes (⚠️ ข้อมูลหาย!)
docker compose down -v

# หยุด + ลบ images
docker compose down --rmi all

# หยุด + ลบทุกอย่าง
docker compose down -v --rmi all --remove-orphans


# ========================================
# จัดการ Services
# ========================================

# Restart services
docker compose restart
docker compose restart app

# Stop (ไม่ลบ containers)
docker compose stop
docker compose stop app

# Start (containers ที่ stop ไว้)
docker compose start
docker compose start app

# Pause/Unpause
docker compose pause app
docker compose unpause app

# ลบ stopped containers
docker compose rm
docker compose rm -f  # force
```

### การดู Status และ Logs

```bash
# ========================================
# ดู Status
# ========================================

# ดู status ทั้งหมด
docker compose ps

# ดู status รวม stopped containers
docker compose ps -a

# ดูแบบ format ตาราง
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# ดู health status
docker compose ps --format "table {{.Name}}\t{{.State}}\t{{.Health}}"


# ========================================
# ดู Logs
# ========================================

# ดู logs ทั้งหมด
docker compose logs

# Follow logs (real-time)
docker compose logs -f

# Logs เฉพาะ service
docker compose logs app
docker compose logs -f db

# Logs หลายservices
docker compose logs app db redis

# จำกัดจำนวนบรรทัด
docker compose logs --tail=100 app
docker compose logs --tail=50 -f app

# แสดง timestamps
docker compose logs -f --timestamps

# Follow logs ตั้งแต่เวลาที่กำหนด
docker compose logs --since 2024-01-01T00:00:00
docker compose logs --since 30m  # 30 minutes ago
```

### เข้าใช้งาน Container

```bash
# ========================================
# Execute Commands
# ========================================

# เข้าไปใน container (interactive shell)
docker compose exec app bash
docker compose exec app sh      # สำหรับ alpine
docker compose exec -it app bash  # explicit interactive + tty

# รันคำสั่งใน container
docker compose exec app python --version
docker compose exec app node --version
docker compose exec app ls -la

# เข้า database
docker compose exec db psql -U postgres
docker compose exec db psql -U postgres -d mydb
docker compose exec db mysql -u root -p

# เข้า Redis
docker compose exec redis redis-cli

# Run as specific user
docker compose exec -u root app bash


# ========================================
# Run (สร้าง container ใหม่)
# ========================================

# รันคำสั่งใน container ใหม่
docker compose run app python manage.py migrate
docker compose run --rm app npm test  # --rm ลบหลังรันเสร็จ

# Run without starting dependencies
docker compose run --no-deps app npm test
```

### Build และ Pull

```bash
# ========================================
# Build
# ========================================

# Build ทั้งหมด
docker compose build

# Build โดยไม่ใช้ cache
docker compose build --no-cache

# Build เฉพาะ service
docker compose build app

# Build parallel
docker compose build --parallel


# ========================================
# Pull
# ========================================

# Pull images ล่าสุด
docker compose pull

# Pull และ build
docker compose up --build --pull always
```

### Validate และ Config

```bash
# ========================================
# Validation
# ========================================

# ตรวจสอบ syntax
docker compose config

# แสดง resolved config (แทนที่ variables)
docker compose config

# แสดง services
docker compose config --services

# แสดง volumes
docker compose config --volumes

# แสดง networks
docker compose config --networks

# Quiet mode (ไม่แสดงผล ถ้าถูกต้อง)
docker compose config --quiet


# ========================================
# Dry Run
# ========================================

# ทดสอบโดยไม่รัน
docker compose up --dry-run
```

### ทำงานกับ Volumes

```bash
# ========================================
# Volume Management
# ========================================

# List volumes
docker volume ls

# Inspect volume
docker volume inspect projectname_postgres_data

# Remove volume
docker volume rm projectname_postgres_data

# Prune unused volumes
docker volume prune


# ========================================
# Backup Volume
# ========================================

# Backup
docker run --rm \
  -v projectname_postgres_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz /data

# Restore
docker run --rm \
  -v projectname_postgres_data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/postgres-backup-20240101.tar.gz -C /
```

### อื่นๆ

```bash
# ดู top processes
docker compose top

# ดู resource usage
docker stats $(docker compose ps -q)

# ดู images
docker compose images

# ดู port mapping
docker compose port app 5000

# ดู events
docker compose events

# Copy files
docker compose cp app:/app/logs/app.log ./local-logs/
```

---

<a name="section-10"></a>
## 10. Debugging และ Troubleshooting

### เครื่องมือ Debug พื้นฐาน

```bash
# ========================================
# 1. ตรวจสอบ Status
# ========================================

# ดู status พร้อม health
docker compose ps

# ดูว่า container ยัง running อยู่ไหม
docker compose ps -a

# ดู exit code
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.ExitCode}}"


# ========================================
# 2. ตรวจสอบ Logs
# ========================================

# ดู logs error
docker compose logs | grep -i error
docker compose logs | grep -i warning

# ดู logs service ที่มีปัญหา
docker compose logs -f app

# ดู logs ย้อนหลัง
docker compose logs --tail=200 app


# ========================================
# 3. ตรวจสอบ Health Check
# ========================================

# ดู health status
docker compose ps --format "{{.Name}}: {{.Health}}"

# Inspect health check details
docker inspect <container_name> | grep -A 10 Health

# ทดสอบ health check manually
docker compose exec db pg_isready -U postgres
docker compose exec redis redis-cli ping
curl http://localhost:5000/health
```

### ปัญหาที่พบบ่อย + วิธีแก้

**ปัญหา 1: Service ไม่ Healthy**

```bash
# วินิจฉัย
docker compose ps
docker compose logs db

# ตรวจสอบ health check command
docker inspect todo_postgres | grep -A 10 Health

# ทดสอบ health check manually
docker compose exec db pg_isready -U postgres
docker compose exec db psql -U postgres -c "SELECT 1"

# แก้ไข: ปรับ health check ให้เหมาะสม
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres -d mydb"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 30s  # เพิ่มเวลารอ
```

**ปัญหา 2: Container Crash ทันที**

```bash
# ดู exit code และ error
docker compose ps -a
docker compose logs app

# ดู error ที่ระบุ line number
docker compose logs app 2>&1 | grep "Error"

# รัน container แบบ interactive เพื่อ debug
docker compose run --rm app bash
docker compose run --rm app python -c "import app"

# ตรวจสอบ environment variables
docker compose exec app env | grep DATABASE

# ตรวจสอบ permissions
docker compose exec app ls -la /app
```

**ปัญหา 3: Port Already in Use**

```bash
# ดูว่า port ถูกใช้โดยอะไร
# Linux/macOS
lsof -i :5000
sudo lsof -i :5000

# Windows
netstat -ano | findstr :5000

# วิธีแก้:
# 1. หยุดโปรแกรมที่ใช้ port
# 2. เปลี่ยน port ใน docker-compose.yml
ports:
  - "5001:5000"  # ใช้ 5001 แทน
```

**ปัญหา 4: Cannot Connect to Database**

```bash
# ตรวจสอบ network
docker network ls
docker network inspect projectname_app-network

# ตรวจสอบว่า service อยู่ network เดียวกันไหม
docker compose config | grep -A 5 networks

# ทดสอบ connectivity
docker compose exec app ping db
docker compose exec app nc -zv db 5432
docker compose exec app telnet db 5432

# ตรวจสอบ DATABASE_URL
docker compose exec app env | grep DATABASE
docker compose exec app echo $DATABASE_URL

# ทดสอบเชื่อมต่อ
docker compose exec app python -c "
import psycopg2
conn = psycopg2.connect('postgresql://user:pass@db:5432/mydb')
print('Connected!')
"
```

**ปัญหา 5: Volume Permission Denied**

```bash
# ตรวจสอบ permissions
docker compose exec app ls -la /app

# แก้ไข ownership
docker compose exec -u root app chown -R appuser:appuser /app

# หรือกำหนด user ใน docker-compose.yml
services:
  app:
    user: "1000:1000"  # UID:GID
```

**ปัญหา 6: Out of Memory/Disk Space**

```bash
# ตรวจสอบ disk usage
docker system df
docker system df -v

# ตรวจสอบ container resources
docker stats

# Cleanup
docker system prune        # ลบ unused data
docker system prune -a     # ลบทุกอย่างที่ไม่ใช้
docker volume prune        # ลบ unused volumes
docker image prune -a      # ลบ unused images
```

**ปัญหา 7: Build Failed**

```bash
# Build แบบละเอียด
docker compose build --progress=plain --no-cache

# ดู build context
docker compose config

# ตรวจสอบ .dockerignore
cat .dockerignore

# Build specific stage
docker compose build --target builder
```

### Debug Mode

```bash
# ========================================
# Enable Debug Logging
# ========================================

# Verbose output
docker compose --verbose up

# ดู environment variables ทั้งหมด
docker compose config

# ดู resolved config
docker compose config --resolve-image-digests


# ========================================
# Inspect Container
# ========================================

# ดู detailed info
docker inspect <container_name>

# ดู specific field
docker inspect <container_name> | jq '.[0].State'
docker inspect <container_name> | jq '.[0].NetworkSettings'
docker inspect <container_name> | jq '.[0].Mounts'

# ดู environment variables
docker inspect <container_name> | jq '.[0].Config.Env'


# ========================================
# Process Monitoring
# ========================================

# ดู running processes
docker compose exec app ps aux

# ดู resource usage
docker stats $(docker compose ps -q)

# ดู disk usage ใน container
docker compose exec app df -h
docker compose exec app du -sh /*


# ========================================
# Network Debugging
# ========================================

# ติดตั้ง network tools (ใน container)
docker compose exec -u root app apt-get update
docker compose exec -u root app apt-get install -y iputils-ping dnsutils netcat curl

# ทดสอบ DNS
docker compose exec app nslookup db
docker compose exec app dig db

# ทดสอบ connection
docker compose exec app ping -c 3 db
docker compose exec app nc -zv db 5432
docker compose exec app curl -v http://db:5432
```

### Performance Profiling

```bash
# ดู container stats
docker stats --no-stream

# ดู logs performance
time docker compose logs > /dev/null

# Analyze image layers
docker history myimage:latest

# ดูขนาด image
docker images | grep myimage

# วิเคราะห์ image ด้วย dive
dive myimage:latest
```

---

<a name="section-11"></a>
## 11. Best Practices

### Security Best Practices

**1. ไม่ Commit Secrets**

```yaml
# ❌ Bad - hardcoded password
services:
  db:
    environment:
      POSTGRES_PASSWORD: mysecretpassword123

# ✅ Good - ใช้ environment variable
services:
  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

```bash
# .gitignore
.env
.env.*
!.env.example
docker-compose.override.yml
*.pem
*.key
secrets/
```

**2. ใช้ Docker Secrets (Swarm mode)**

```yaml
services:
  app:
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt
```

**3. Non-root User**

```dockerfile
# ใน Dockerfile
RUN useradd -m -u 1000 appuser
USER appuser
```

```yaml
# ใน docker-compose.yml
services:
  app:
    user: "1000:1000"
```

**4. Read-only Filesystem**

```yaml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

**5. Drop Capabilities**

```yaml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

**6. Network Isolation**

```yaml
services:
  frontend:
    networks:
      - frontend-net
  
  backend:
    networks:
      - frontend-net
      - backend-net
  
  db:
    networks:
      - backend-net  # ไม่ expose ไปยัง frontend

networks:
  frontend-net:
  backend-net:
    internal: true  # ไม่มี internet access
```

### Performance Best Practices

**1. ใช้ Alpine/Slim Images**

```yaml
services:
  app:
    image: python:3.11-slim    # ~50MB
    # image: python:3.11       # ~300MB ❌
  
  db:
    image: postgres:16-alpine  # ~80MB
    # image: postgres:16       # ~150MB ❌
```

**2. Resource Limits**

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '1'
          memory: 512M
```

**3. Health Checks (สำคัญ!)**

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

**4. Logging Configuration**

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        compress: "true"
```

**5. Optimize Build Context**

```
# .dockerignore
.git
.gitignore
node_modules
__pycache__
*.pyc
.env
.env.*
README.md
*.md
.vscode
.idea
.pytest_cache
htmlcov
dist
build
```

### Development Best Practices

**1. docker-compose.override.yml**

**docker-compose.yml (base):**
```yaml
services:
  app:
    build: .
    ports:
      - "5000:5000"
```

**docker-compose.override.yml (development - auto-loaded):**
```yaml
services:
  app:
    build:
      target: development
    volumes:
      - ./app:/app  # Hot reload
    environment:
      FLASK_ENV: development
      DEBUG: "true"
    command: flask run --reload --host=0.0.0.0
```

**docker-compose.prod.yml (production):**
```yaml
services:
  app:
    build:
      target: production
    restart: always
    environment:
      FLASK_ENV: production
      DEBUG: "false"
    command: gunicorn -w 4 -b 0.0.0.0:5000 run:app
```

```bash
# Development (auto-load override)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

**2. ใช้ Profiles**

```yaml
services:
  app:
    # Always start
  
  db:
    # Always start
  
  adminer:
    profiles: ["dev", "debug"]
    image: adminer
    ports:
      - "8080:8080"
  
  pgadmin:
    profiles: ["debug"]
    image: dpage/pgadmin4
    ports:
      - "5050:80"
```

```bash
# Start without debug tools
docker compose up

# Start with dev profile
docker compose --profile dev up

# Start with debug profile
docker compose --profile debug up
```

**3. Hot Reload ใน Development**

```yaml
services:
  # Python/Flask
  app:
    volumes:
      - ./app:/app
    environment:
      FLASK_ENV: development
    command: flask run --reload --host=0.0.0.0
  
  # Node.js
  frontend:
    volumes:
      - ./frontend:/app
      - /app/node_modules  # ไม่ override node_modules
    command: npm run dev
```

### Maintenance Best Practices

**1. Naming Conventions**

```yaml
# ✅ Good - มีความหมาย
services:
  todo-api:
    container_name: todo_api_prod
    image: todo-api:1.0.0
  
  todo-db:
    container_name: todo_postgres_prod

# ❌ Bad - ไม่ชัดเจน
services:
  app:
    container_name: app1
  db:
    container_name: db1
```

**2. Labels สำหรับ Metadata**

```yaml
services:
  app:
    labels:
      com.example.description: "Todo API Backend"
      com.example.version: "1.0.0"
      com.example.team: "backend-team"
      com.example.environment: "production"
```

**3. Documentation**

```yaml
# ใส่ comments อธิบาย
services:
  app:
    # Main Flask application
    # Runs with Gunicorn in production
    # Connects to PostgreSQL and Redis
    build:
      context: .
      target: production
    depends_on:
      db:
        condition: service_healthy  # รอให้ DB พร้อมก่อน
```

**4. Version Control**

```bash
# Git tags สำหรับ releases
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# Docker image tags
docker tag myapp:latest myapp:1.0.0
docker tag myapp:latest myapp:stable
```

---

<a name="section-12"></a>
## 12. ตัวอย่างการใช้งานจริง

### ตัวอย่างที่ 1: Flask Todo Application (Complete)

**โครงสร้างโปรเจกต์:**
```
todo-app/
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── routes.py
│   └── config.py
├── tests/
│   └── test_app.py
├── Dockerfile
├── docker-compose.yml
├── docker-compose.prod.yml
├── requirements.txt
├── .env.example
├── .dockerignore
├── .gitignore
└── README.md
```

**docker-compose.yml:**
```yaml
services:
  # =============================
  # PostgreSQL Database
  # =============================
  db:
    image: postgres:16-alpine
    container_name: todo_postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-todo_dev}
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD required}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # =============================
  # User Service
  # =============================
  user-service:
    build: ./services/user
    environment:
      DB_URL: postgresql://${DB_USER}:${DB_PASSWORD}@user-db:5432/users
      RABBITMQ_URL: amqp://${RABBITMQ_USER}:${RABBITMQ_PASSWORD}@rabbitmq:5672
    depends_on:
      user-db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - microservices
      - user-network
    restart: unless-stopped

  user-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: users
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - user_db_data:/var/lib/postgresql/data
    networks:
      - user-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5

  # =============================
  # Message Queue (RabbitMQ)
  # =============================
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - microservices
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5
    restart: unless-stopped

  # =============================
  # Redis (Shared Cache)
  # =============================
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - microservices
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  auth_db_data:
  user_db_data:
  rabbitmq_data:
  redis_data:

networks:
  microservices:
    driver: bridge
  auth-network:
    driver: bridge
    internal: true
  user-network:
    driver: bridge
    internal: true
```

---

## 📋 Checklist สำหรับ Production

### Pre-deployment Checklist

**Security:**
- [ ] ไม่มี hardcoded secrets ใน docker-compose.yml
- [ ] สร้าง `.env` file และไม่ commit
- [ ] เพิ่ม `.env` ใน `.gitignore`
- [ ] สร้าง `.env.example` เป็น template
- [ ] ใช้ non-root user ใน containers
- [ ] ตั้งค่า read-only filesystem (ถ้าเป็นไปได้)
- [ ] จำกัด capabilities (cap_drop)
- [ ] แยก networks ตาม security zones

**Configuration:**
- [ ] ทุก services มี healthcheck
- [ ] ทุก services มี restart policy
- [ ] ตั้งค่า resource limits (CPU, Memory)
- [ ] กำหนด logging configuration
- [ ] ใช้ Alpine/Slim images
- [ ] Multi-stage builds สำหรับ applications
- [ ] Volume สำหรับ persistent data

**Testing:**
- [ ] `docker compose config` ผ่าน (ไม่มี syntax error)
- [ ] Build ทุก services สำเร็จ
- [ ] Health checks ทำงานถูกต้อง
- [ ] Service dependencies ทำงานถูกต้อง
- [ ] Environment variables ครบถ้วน
- [ ] Volume persistence ทำงาน
- [ ] Network connectivity ระหว่าง services
- [ ] Restart policies ทดสอบแล้ว

**Documentation:**
- [ ] README.md อธิบายการ setup
- [ ] มี comments ใน docker-compose.yml
- [ ] มี architecture diagram
- [ ] มี troubleshooting guide

**Backup:**
- [ ] มี backup strategy สำหรับ volumes
- [ ] ทดสอบ restore procedure
- [ ] มี monitoring/alerting (optional)

---

## 🎯 คำถามสำคัญสำหรับทำข้อสอบ

### พื้นฐาน Docker Compose

**1. Docker Compose คืออะไร?**
- เครื่องมือจัดการ multi-container applications
- ใช้ YAML file กำหนดค่าทุก services
- รัน/หยุด ทุก services ด้วยคำสั่งเดียว
- ใช้ได้ทั้ง development และ production

**2. ไฟล์ docker-compose.yml ต้องอยู่ที่ไหน?**
- อยู่ที่ root ของโปรเจกต์
- ชื่อต้องเป็น `docker-compose.yml` หรือ `docker-compose.yaml`

**3. คำสั่งพื้นฐาน Docker Compose?**
- `docker compose up -d` - เริ่ม services background
- `docker compose down` - หยุดและลบ containers
- `docker compose logs -f` - ดู logs real-time
- `docker compose ps` - ดู status
- `docker compose exec` - รันคำสั่งใน container

### Services Configuration

**4. ความแตกต่างระหว่าง `image:` และ `build:`?**
- `image:` - ใช้ existing image จาก registry (เช่น Docker Hub)
- `build:` - build image จาก Dockerfile ในเครื่อง

**5. `depends_on` ทำงานอย่างไร?**
- กำหนดลำดับการเริ่ม services
- `condition: service_started` - รอให้ container start
- `condition: service_healthy` - รอให้ผ่าน health check (แนะนำ!)
- ไม่รอให้ application พร้อมใช้งาน 100%

**6. Restart policies มีอะไรบ้าง?**
- `no` - ไม่ restart (default)
- `always` - restart เสมอ
- `on-failure` - restart เมื่อ exit code != 0
- `unless-stopped` - restart จนกว่าจะ stop manual (แนะนำ production)

### Networks และ Volumes

**7. Networks มีความสำคัญอย่างไร?**
- แยก traffic ระหว่าง services
- เพิ่มความปลอดภัย (isolation)
- Service discovery - เรียกกันด้วยชื่อ service
- สามารถแยก frontend/backend networks

**8. Named Volumes vs Bind Mounts?**
- **Named volumes:** จัดการโดย Docker, ใช้สำหรับ production, ข้อมูลปลอดภัย
- **Bind mounts:** mount จาก host, ใช้สำหรับ development, hot reload

**9. Volume persist data อย่างไร?**
- เก็บข้อมูลนอก container lifecycle
- ไม่หายเมื่อ container ถูกลบ
- ต้องใช้ `docker compose down -v` เพื่อลบ

**10. ทำไมต้องใช้ internal network?**
- ป้องกัน service เชื่อมต่อ internet
- เหมาะสำหรับ database, internal services
- เพิ่มความปลอดภัย

### Health Checks

**11. Health check สำคัญอย่างไร?**
- รอให้ service พร้อมก่อนเริ่ม services อื่น
- Auto-restart เมื่อ service unhealthy
- ใช้ร่วมกับ `depends_on: condition: service_healthy`
- Monitoring และ load balancing

**12. Health check parameters คืออะไร?**
- `test` - คำสั่งตรวจสอบ
- `interval` - ตรวจสอบทุก X วินาที
- `timeout` - timeout
- `retries` - จำนวนครั้งที่ลองใหม่
- `start_period` - รอก่อนเริ่มตรวจสอบ

### Environment Variables

**13. วิธีจัดการ secrets ที่ปลอดภัย?**
- ใช้ `.env` file
- ไม่ commit `.env` ใน git
- สร้าง `.env.example` เป็น template
- ใช้ `${VARIABLE}` reference ใน compose file
- ใช้ Docker secrets สำหรับ production

**14. Environment variable precedence?**
1. Shell environment (สูงสุด)
2. .env file
3. environment ใน docker-compose.yml
4. Dockerfile ENV (ต่ำสุด)

### Dockerfile

**15. Multi-stage build คืออะไร?**
- แยก build และ runtime stages
- ลดขนาด final image มาก
- เพิ่มความปลอดภัย - ไม่เก็บ build tools
- แยก concerns ชัดเจน

**16. Layer caching ทำงานอย่างไร?**
- Docker cache แต่ละ instruction (layer)
- ถ้า instruction ไม่เปลี่ยน → ใช้ cache
- เรียงจากน้อยเปลี่ยนไปมากเปลี่ยน
- Copy dependencies ก่อน source code

**17. ทำไมต้องใช้ non-root user?**
- เพิ่มความปลอดภัย
- จำกัด damage ถ้า container ถูก compromise
- Best practice สำหรับ production

**18. .dockerignore มีประโยชน์อย่างไร?**
- ลดขนาด build context
- เร็วขึ้น (ไม่ส่งไฟล์ที่ไม่จำเป็น)
- ไม่รวม .git, node_modules, .env

### Commands และ Debugging

**19. ความแตกต่าง `docker compose up` vs `docker compose up -d`?**
- `up` - foreground, เห็น logs, กด Ctrl+C หยุด
- `up -d` - background (detached), ต้องใช้ `docker compose down`

**20. วิธี debug container ที่ crash?**
```bash
# 1. ดู logs
docker compose logs app

# 2. ดู exit code
docker compose ps -a

# 3. เข้าไปใน container
docker compose run --rm app bash

# 4. ตรวจสอบ environment
docker compose exec app env
```

---

## 💡 Tips สำหรับทำ Lab CI/CD

### การเตรียมตัว

**1. ตรวจสอบ syntax ก่อนรัน**
```bash
docker compose config
docker compose config --quiet  # ไม่แสดงผล ถ้าถูกต้อง
```

**2. ใช้ .env.example เป็น template**
```bash
cp .env.example .env
# แก้ไข .env
nano .env
```

**3. Generate secure passwords**
```bash
# Python
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# OpenSSL
openssl rand -base64 32
```

### การทดสอบ

**1. ทดสอบ health checks ทีละ service**
```bash
# PostgreSQL
docker compose exec db pg_isready -U postgres

# Redis
docker compose exec redis redis-cli ping

# Application
curl http://localhost:5000/health
```

**2. ทดสอบ connectivity**
```bash
# ภายใน container
docker compose exec app ping db
docker compose exec app nc -zv db 5432
```

**3. ตรวจสอบ logs ทันที**
```bash
# ไม่ใส่ -d เพื่อดู logs
docker compose up

# หรือ
docker compose up -d
docker compose logs -f
```

### การแก้ปัญหา

**1. Container crash ทันที**
```bash
# ดู error
docker compose logs app

# Run interactive
docker compose run --rm app bash
```

**2. Port already in use**
```bash
# ดูว่าโปรแกรมไหนใช้
lsof -i :5000  # macOS/Linux
netstat -ano | findstr :5000  # Windows

# แก้: เปลี่ยน port
ports:
  - "5001:5000"
```

**3. Database connection failed**
```bash
# ตรวจสอบ DATABASE_URL
docker compose exec app env | grep DATABASE

# ทดสอบเชื่อมต่อ
docker compose exec app ping db
docker compose exec db psql -U postgres
```

### Optimization

**1. ใช้ Alpine images**
```yaml
services:
  db:
    image: postgres:16-alpine    # ~80MB
  app:
    image: python:3.11-slim      # ~50MB
```

**2. Multi-stage builds**
```dockerfile
FROM python:3.11-slim AS builder
# ... build stage

FROM python:3.11-slim AS runtime
COPY --from=builder ...
```

**3. Health checks ทุก service**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/health"]
  interval: 30s
  timeout: 10s
  retries: 3
```

---

## 📚 Resources

### Official Documentation
- [Docker Compose Specification](https://docs.docker.com/compose/compose-file/)
- [Docker Compose CLI Reference](https://docs.docker.com/compose/reference/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Security](https://docs.docker.com/engine/security/)

### Learning Resources
- [Docker Compose Tutorial](https://docs.docker.com/compose/gettingstarted/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [12-Factor App](https://12factor.net/)

### Tools
- [YAML Validator](https://yamllint.com/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Dive - Docker Image Analyzer](https://github.com/wagoodman/dive)
- [Hadolint - Dockerfile Linter](https://github.com/hadolint/hadolint)

### Community
- [Docker Hub](https://hub.docker.com/)
- [Awesome Docker](https://github.com/veggiemonk/awesome-docker)
- [Awesome Compose](https://github.com/docker/awesome-compose)

---

## 🎓 สรุปท้ายสุด

### สิ่งที่ต้องเข้าใจ

**1. Docker Compose Basics**
- Services, Networks, Volumes
- YAML syntax
- คำสั่งพื้นฐาน (up, down, logs, exec)

**2. Configuration**
- Service dependencies (depends_on)
- Health checks (สำคัญมาก!)
- Environment variables
- Resource limits
- Restart policies

**3. Dockerfile**
- Multi-stage builds
- Layer optimization
- Security (non-root user)
- .dockerignore

**4. Networking**
- Service discovery
- Network isolation
- Internal vs external networks

**5. Data Persistence**
- Named volumes (production)
- Bind mounts (development)
- Backup strategies

**6. Best Practices**
- Security (secrets, non-root, read-only)
- Performance (Alpine, caching, limits)
- Development workflow (override files, profiles)
- Debugging techniques

### Key Takeaways สำหรับ CI/CD

✅ **Docker Compose เหมาะสำหรับ:**
- Development environment
- Testing ใน CI/CD pipelines
- Small-scale production
- Quick prototyping

✅ **Health checks เป็น must-have:**
- รอให้ database พร้อมก่อนเริ่ม app
- Auto-restart unhealthy containers
- ใช้กับ `depends_on: condition: service_healthy`

✅ **Security first:**
- ไม่ commit secrets
- ใช้ non-root user
- จำกัด network access
- Resource limits

✅ **Optimize images:**
- Alpine/Slim variants
- Multi-stage builds
- Layer caching
- .dockerignore

### ขั้นตอนต่อไป

1. ✅ ทำ Lab ตามเอกสารที่กำหนด
2. ✅ ทดลองสร้าง docker-compose.yml เอง
3. ✅ ปรับใช้กับโปรเจกต์จริง
4. ✅ ศึกษา Kubernetes สำหรับ large-scale production

---

## 🚀 Quick Reference Card

### คำสั่งที่ใช้บ่อย

```bash
# Start
docker compose up -d

# Stop
docker compose down

# Logs
docker compose logs -f

# Status
docker compose ps

# Execute
docker compose exec app bash

# Restart
docker compose restart app

# Rebuild
docker compose up -d --build

# Validate
docker compose config
```

### Health Check Templates

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 10s
  timeout: 5s
  retries: 5

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5

# HTTP
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
  interval: 30s
  timeout: 10s
  retries: 3
```

### Troubleshooting Checklist

```bash
# 1. ตรวจสอบ syntax
docker compose config

# 2. ดู status
docker compose ps

# 3. ดู logs
docker compose logs -f app

# 4. ตรวจสอบ health
docker compose exec db pg_isready -U postgres

# 5. ทดสอบ connectivity
docker compose exec app ping db

# 6. ดู environment variables
docker compose exec app env
```

---

**Good luck กับการทำ Lab CI/CD! 🐳🚀**POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-todo_dev}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # =============================
  # Flask Application
  # =============================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: ${BUILD_TARGET:-development}
      args:
        PYTHON_VERSION: 3.11
    container_name: todo_app
    environment:
      FLASK_ENV: ${FLASK_ENV:-development}
      DATABASE_URL: postgresql://${POSTGRES_USER:-postgres}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB:-todo_dev}
      SECRET_KEY: ${SECRET_KEY:?SECRET_KEY required}
      REDIS_URL: redis://redis:6379/0
    ports:
      - "${APP_PORT:-5000}:5000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - .:/app
      - /app/__pycache__
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # =============================
  # Redis Cache
  # =============================
  redis:
    image: redis:7-alpine
    container_name: todo_redis
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-}
    volumes:
      - redis_data:/data
    ports:
      - "${REDIS_PORT:-6379}:6379"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

**docker-compose.prod.yml:**
```yaml
services:
  app:
    build:
      target: production
    environment:
      FLASK_ENV: production
    volumes: []  # ไม่ mount source code
    command: gunicorn --bind 0.0.0.0:5000 --workers 4 --timeout 120 run:app
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '1'
          memory: 512M
  
  db:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
```

### ตัวอย่างที่ 2: Full Stack (Frontend + Backend + DB)

```yaml
services:
  # =============================
  # React Frontend
  # =============================
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: production
    container_name: react_frontend
    ports:
      - "3000:80"
    environment:
      REACT_APP_API_URL: ${API_URL:-http://localhost:5000}
    depends_on:
      - backend
    networks:
      - frontend-network
    restart: unless-stopped

  # =============================
  # Flask Backend API
  # =============================
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: production
    container_name: flask_backend
    ports:
      - "5000:5000"
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      SECRET_KEY: ${SECRET_KEY}
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend-network
      - backend-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # =============================
  # PostgreSQL Database
  # =============================
  db:
    image: postgres:16-alpine
    container_name: postgres_db
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # =============================
  # Redis Cache
  # =============================
  redis:
    image: redis:7-alpine
    container_name: redis_cache
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend-network
    healthcheck:
      test: ["CMD", "redis-cli", "--pass", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # =============================
  # Nginx Reverse Proxy
  # =============================
  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - frontend
      - backend
    networks:
      - frontend-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
    internal: true  # Isolated from internet
```

### ตัวอย่างที่ 3: Microservices

```yaml
services:
  # =============================
  # API Gateway
  # =============================
  gateway:
    build: ./services/gateway
    container_name: api_gateway
    ports:
      - "8080:8080"
    environment:
      AUTH_SERVICE_URL: http://auth-service:3001
      USER_SERVICE_URL: http://user-service:3002
      ORDER_SERVICE_URL: http://order-service:3003
      PRODUCT_SERVICE_URL: http://product-service:3004
    networks:
      - microservices
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # =============================
  # Auth Service
  # =============================
  auth-service:
    build: ./services/auth
    environment:
      DB_URL: postgresql://${DB_USER}:${DB_PASSWORD}@auth-db:5432/auth
      JWT_SECRET: ${JWT_SECRET}
      REDIS_URL: redis://redis:6379/0
    depends_on:
      auth-db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - microservices
      - auth-network
    restart: unless-stopped

  auth-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: auth
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - auth_db_data:/var/lib/postgresql/data
    networks:
      - auth-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${

## 📚 สารบัญ

1. [Docker Compose คืออะไร](#section-1)
2. [โครงสร้างไฟล์ Docker Compose](#section-2)
3. [Services Configuration](#section-3)
4. [Networks และ Volumes](#section-4)
5. [Health Checks](#section-5)
6. [Environment Variables](#section-6)
7. [Dockerfile Best Practices](#section-7)
8. [Multi-stage Builds](#section-8)
9. [คำสั่ง Docker Compose](#section-9)
10. [Debugging และ Troubleshooting](#section-10)
11. [Best Practices](#section-11)
12. [ตัวอย่างการใช้งานจริง](#section-12)

---

<a name="section-1"></a>
## 1. Docker Compose คืออะไร?

### คำจำกัดความ

**Docker Compose** เป็นเครื่องมือที่ใช้จัดการแอปพลิเคชันที่ประกอบด้วย **หลาย Docker containers** โดยใช้ไฟล์ YAML เดียวในการกำหนดค่าทั้งหมด

### ความสำคัญสำหรับ CI/CD

- ✅ **จัดการหลาย containers** - รัน app, database, cache พร้อมกัน
- ✅ **ง่ายต่อการใช้งาน** - คำสั่งเดียวสำหรับ start/stop ทุก services
- ✅ **Reproducible** - สามารถ setup environment เดียวกันได้ทุกครั้ง
- ✅ **Development to Production** - ใช้ได้ทั้ง development และ production
- ✅ **Integration กับ CI/CD** - ทำงานร่วมกับ GitHub Actions ได้ดีมาก
- ✅ **Isolation** - แต่ละ service แยกกัน ไม่กระทบกัน

### เปรียบเทียบ: ไม่ใช้ vs ใช้ Docker Compose

**ไม่ใช้ Docker Compose (ยุ่งยาก):**
```bash
# ต้องรัน container ทีละตัว - เสี่ยงพลาด
docker network create app-network

docker run -d --name postgres \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16

docker run -d --name redis \
  --network app-network \
  redis:7

docker run -d --name app \
  --network app-network \
  -p 5000:5000 \
  -e DATABASE_URL=postgresql://postgres:secret@postgres/mydb \
  -e REDIS_URL=redis://redis:6379 \
  myapp
```

**ใช้ Docker Compose (ง่าย, สะดวก):**
```bash
# คำสั่งเดียวจบ!
docker compose up -d

# หยุดทุกอย่าง
docker compose down
```

---

<a name="section-2"></a>
## 2. โครงสร้างไฟล์ Docker Compose

### โครงสร้างพื้นฐาน

```yaml
# Modern Docker Compose - ไม่ต้องระบุ version
# ใช้ Compose Specification V2 อัตโนมัติ

services:           # กำหนด containers ต่างๆ
  service1:
    # configuration
  service2:
    # configuration

networks:           # กำหนดเครือข่าย (optional)
  network1:
    driver: bridge

volumes:            # กำหนด volumes (optional)
  volume1:
    driver: local
```

### ตัวอย่าง: Flask Todo Application

```yaml
services:
  # =============================
  # Database Service
  # =============================
  db:
    image: postgres:16-alpine
    container_name: todo_postgres
    environment:
      POSTGRES_DB: todo_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  # =============================
  # Application Service
  # =============================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: todo_app
    environment:
      FLASK_ENV: ${FLASK_ENV}
      DATABASE_URL: postgresql://postgres:${DB_PASSWORD}@db:5432/todo_dev
      SECRET_KEY: ${SECRET_KEY}
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app  # Development: hot reload
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

---

<a name="section-3"></a>
## 3. Services Configuration

### ตัวเลือกสำคัญทั้งหมด

| Key | คำอธิบาย | ตัวอย่าง |
|-----|---------|---------|
| `image:` | Docker image ที่จะใช้ | `postgres:16-alpine` |
| `build:` | Build จาก Dockerfile | `build: .` |
| `container_name:` | ชื่อ container | `flask_app` |
| `ports:` | Port mapping | `"5000:5000"` |
| `environment:` | Environment variables | `FLASK_ENV: development` |
| `env_file:` | โหลด env จากไฟล์ | `.env` |
| `depends_on:` | Service dependencies | `depends_on: - db` |
| `volumes:` | Mount volumes/directories | `- .:/app` |
| `networks:` | เครือข่าย | `- app-network` |
| `command:` | Override CMD | `flask run --host=0.0.0.0` |
| `restart:` | Restart policy | `unless-stopped` |
| `healthcheck:` | Health check | ดูรายละเอียด section 5 |
| `deploy:` | Resource limits | `limits: cpus: '2'` |

### Build Configuration

```yaml
# แบบง่าย
services:
  app:
    build: .

# แบบละเอียด (แนะนำ)
services:
  app:
    build:
      context: ./app          # ตำแหน่ง build context
      dockerfile: Dockerfile  # ชื่อไฟล์
      target: production      # Multi-stage target
      args:                   # Build arguments
        NODE_VERSION: 18
        PYTHON_VERSION: 3.11
      cache_from:
        - myapp:latest
```

### Ports - 3 วิธี

```yaml
services:
  app:
    ports:
      # 1. Short syntax (แนะนำ)
      - "5000:5000"           # host:container
      - "3000:3000"
      
      # 2. Long syntax (ละเอียด)
      - target: 5000          # Port ใน container
        published: 5000       # Port บน host
        protocol: tcp
        mode: host
      
      # 3. Range
      - "8000-8010:8000-8010"
```

### Depends On - รอให้ Service พร้อม

```yaml
services:
  app:
    depends_on:
      # แบบที่ 1: รอให้ start เท่านั้น
      - db
      - redis
      
      # แบบที่ 2: รอให้ healthy (แนะนำ!)
      db:
        condition: service_healthy
      redis:
        condition: service_started
```

**Conditions มี 3 แบบ:**
- `service_started` - รอให้ container start
- `service_healthy` - รอให้ผ่าน health check ✅
- `service_completed_successfully` - รอให้รันเสร็จ

### Restart Policies

```yaml
services:
  app:
    # no - ไม่ restart (default)
    restart: no
    
    # always - restart เสมอ (แม้ stop manually)
    restart: always
    
    # on-failure - restart เฉพาะเมื่อ exit code != 0
    restart: on-failure
    
    # unless-stopped - restart จนกว่าจะ stop manual (แนะนำ!)
    restart: unless-stopped
```

---

<a name="section-4"></a>
## 4. Networks และ Volumes

### Networks - ทำไมต้องใช้?

**ประโยชน์:**
- ✅ แยก traffic ระหว่าง services
- ✅ ความปลอดภัยสูงขึ้น
- ✅ Service discovery - เรียกกันด้วยชื่อ service
- ✅ Isolation - แยก frontend/backend networks

**ตัวอย่าง: แยก Networks ตาม Security Zones**

```yaml
services:
  # Frontend - เชื่อมต่อ public
  frontend:
    networks:
      - frontend-network
      - backend-network
  
  # Backend - เชื่อมต่อทั้ง frontend และ database
  backend:
    networks:
      - backend-network
      - database-network
  
  # Database - เชื่อมต่อเฉพาะ backend (ปลอดภัย!)
  db:
    networks:
      - database-network

networks:
  frontend-network:
    driver: bridge
  
  backend-network:
    driver: bridge
  
  database-network:
    driver: bridge
    internal: true  # ไม่เชื่อมต่อ internet!
```

**Service Discovery:**
```yaml
# ใน container สามารถเรียกกันด้วยชื่อ service
services:
  app:
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
      #                                     ^^
      #                              ชื่อ service ใช้เป็น hostname
      REDIS_URL: redis://redis:6379
      #                  ^^^^^
      #              ชื่อ service
```

### Volumes - เก็บข้อมูลถาวร

**3 ประเภท:**

**1. Named Volumes (แนะนำสำหรับ Production)**
```yaml
services:
  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data
      #  ^^^^^^^^^^^^ ชื่อ volume

volumes:
  postgres_data:  # กำหนด volume
    driver: local
```

**ข้อดี:**
- เก็บข้อมูลปลอดภัย
- จัดการโดย Docker
- Backup ง่าย

**2. Bind Mounts (แนะนำสำหรับ Development)**
```yaml
services:
  app:
    volumes:
      - ./app:/app              # Source code - hot reload
      - ./config:/etc/config    # Config files
      - ./logs:/var/log/app     # Logs
```

**ข้อดี:**
- แก้ code แล้วเห็นผลทันที
- ไม่ต้อง rebuild image

**3. tmpfs (In-memory, Temporary)**
```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /run
```

**Long Syntax (ละเอียด):**
```yaml
services:
  app:
    volumes:
      # Bind mount
      - type: bind
        source: ./app
        target: /app
        read_only: false
      
      # Named volume
      - type: volume
        source: node_modules
        target: /app/node_modules
      
      # tmpfs
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100000000  # 100MB
```

---

<a name="section-5"></a>
## 5. Health Checks - สำคัญมาก!

### ทำไมต้องมี Health Checks?

- ✅ **รอให้ service พร้อม** - ก่อนเริ่ม services อื่น
- ✅ **Auto-restart** - เมื่อ service unhealthy
- ✅ **Monitoring** - ตรวจสอบสถานะ real-time
- ✅ **Load Balancing** - ส่ง traffic ไปเฉพาะ healthy containers
- ✅ **Prevent Errors** - ไม่ให้ app เชื่อมต่อ DB ที่ยังไม่พร้อม

### Syntax

```yaml
healthcheck:
  test: ["CMD-SHELL", "command"]  # คำสั่งตรวจสอบ
  interval: 30s                   # ตรวจสอบทุก 30 วินาที
  timeout: 10s                    # timeout หลัง 10 วินาที
  retries: 3                      # ลอง 3 ครั้งก่อนถือว่า unhealthy
  start_period: 40s               # รอ 40 วินาทีก่อนเริ่มตรวจสอบ
```

### ตัวอย่าง Health Checks

**PostgreSQL:**
```yaml
db:
  image: postgres:16-alpine
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres -d mydb"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 30s
```

**MySQL:**
```yaml
db:
  image: mysql:8
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Redis:**
```yaml
redis:
  image: redis:7-alpine
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 20s
```

**HTTP Endpoint (Flask/Node.js):**
```yaml
app:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
    # หรือ
    # test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
```

**MongoDB:**
```yaml
mongo:
  image: mongo:7
  healthcheck:
    test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Custom Script:**
```yaml
app:
  healthcheck:
    test: ["CMD-SHELL", "/app/scripts/health-check.sh"]
    interval: 30s
    timeout: 10s
    retries: 3
```

### การใช้กับ depends_on

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy  # รอให้ DB healthy ก่อน!
      redis:
        condition: service_healthy
    # app จะไม่ start จนกว่า db และ redis จะ healthy
```

---

<a name="section-6"></a>
## 6. Environment Variables

### 3 วิธีกำหนด Environment Variables

**1. Inline ใน docker-compose.yml (ไม่แนะนำสำหรับ secrets)**
```yaml
services:
  app:
    environment:
      NODE_ENV: production
      API_URL: https://api.example.com
      DEBUG: false
```

**2. ใช้ไฟล์ .env (แนะนำ!)**

**ไฟล์ `.env`:**
```bash
# Database
POSTGRES_DB=mydb
POSTGRES_USER=postgres
POSTGRES_PASSWORD=SecureP@ssw0rd123

# Application
FLASK_ENV=development
SECRET_KEY=your-secret-key-change-in-production
DEBUG=true

# External APIs
API_KEY=sk-1234567890abcdef
STRIPE_KEY=pk_test_1234567890
```

**docker-compose.yml:**
```yaml
services:
  db:
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  
  app:
    env_file:
      - .env                    # โหลดทั้งไฟล์
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
```

**3. จาก System Environment**
```yaml
services:
  app:
    environment:
      DATABASE_URL: ${DATABASE_URL}  # จาก shell environment
      API_KEY: ${API_KEY}
```

### Security Best Practices

**1. สร้าง `.env.example` เป็น template**
```bash
# .env.example
# Database Configuration
POSTGRES_DB=mydb
POSTGRES_USER=postgres
POSTGRES_PASSWORD=change_this_in_production

# Application
FLASK_ENV=development
SECRET_KEY=generate_random_secret_key
DEBUG=true

# External Services
API_KEY=your_api_key_here
```

**2. เพิ่มใน `.gitignore`**
```bash
# .gitignore
.env
.env.local
.env.*.local
.env.production
.env.staging
```

**3. Generate secure secrets**
```bash
# สร้าง random password
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# หรือใช้ openssl
openssl rand -base64 32
```

### Environment Variable Precedence

```
1. Shell environment (สูงสุด)
2. .env file
3. environment ใน docker-compose.yml
4. Dockerfile ENV (ต่ำสุด)
```

---

<a name="section-7"></a>
## 7. Dockerfile Best Practices

### โครงสร้าง Dockerfile ที่ดี

```dockerfile
# ========================================
# Stage 1: Base Image
# ========================================
FROM python:3.11-slim as base

# Set working directory
WORKDIR /app

# ========================================
# Stage 2: Dependencies (Builder)
# ========================================
FROM base as builder

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    postgresql-client \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements FIRST (for layer caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --user -r requirements.txt

# ========================================
# Stage 3: Production
# ========================================
FROM base as production

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    apt-get update && apt-get install -y --no-install-recommends \
    postgresql-client \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy Python packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Environment variables
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP=run.py

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:5000/api/health || exit 1

# Run with Gunicorn (production server)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "--timeout", "120", "run:app"]
```

### 10 Best Practices

**1. ใช้ Official Images + Alpine/Slim**
```dockerfile
# ✅ Good - เล็ก, ปลอดภัย, เร็ว
FROM python:3.11-slim       # ~50MB
FROM node:18-alpine         # ~40MB
FROM postgres:16-alpine     # ~80MB

# ❌ Bad - ใหญ่, ช้า
FROM ubuntu:latest          # ~77MB + ต้อง install ทุกอย่าง
FROM python:3.11            # ~300MB
```

**2. Layer Caching - เรียงจากน้อยเปลี่ยนไปมากเปลี่ยน**
```dockerfile
# ✅ Good Order
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .      # 1. Dependencies (เปลี่ยนน้อย)
RUN pip install -r requirements.txt
COPY . .                     # 2. Source code (เปลี่ยนบ่อย)

# ❌ Bad Order - ทุก build ต้อง install ใหม่
FROM python:3.11-slim
WORKDIR /app
COPY . .                     # Source code ก่อน - cache พัง!
RUN pip install -r requirements.txt
```

**3. Minimize Layers - รวมคำสั่งด้วย &&**
```dockerfile
# ✅ Good - 1 layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        curl \
    && rm -rf /var/lib/apt/lists/*

# ❌ Bad - 3 layers
RUN apt-get update
RUN apt-get install -y build-essential curl
RUN rm -rf /var/lib/apt/lists/*
```

**4. Cleanup ใน Layer เดียวกัน**
```dockerfile
# ✅ Good - ลบใน layer เดียวกัน
RUN apt-get update && \
    apt-get install -y build-essential && \
    rm -rf /var/lib/apt/lists/*      # ลบทันที

# ❌ Bad - ขยะยังอยู่ใน layer
RUN apt-get update
RUN apt-get install -y build-essential
RUN rm -rf /var/lib/apt/lists/*      # ลบทีหลัง - ไม่ได้ผล!
```

**5. ใช้ .dockerignore**
```
# .dockerignore
.git
.gitignore
.env
.env.*
node_modules
npm-debug.log
README.md
*.md
.vscode
.idea
__pycache__
*.pyc
*.pyo
.pytest_cache
htmlcov
.coverage
dist
build
*.egg-info
.DS_Store
Thumbs.db
```

**6. Multi-stage Builds (ลดขนาด)**
```dockerfile
# Build stage - มี build tools
FROM node:18 as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage - เล็ก, ปลอดภัย
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
```

**7. Non-root User (Security)**
```dockerfile
# สร้าง user
RUN useradd -m -u 1000 appuser

# เปลี่ยนเจ้าของไฟล์
COPY --chown=appuser:appuser . .

# Switch user
USER appuser

# ❌ อย่ารัน container ด้วย root!
```

**8. Health Checks**
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1
```

**9. Set Environment Variables**
```dockerfile
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH=/home/appuser/.local/bin:$PATH
```

**10. Use --no-cache-dir**
```dockerfile
# Python
RUN pip install --no-cache-dir -r requirements.txt

# Node.js
RUN npm ci --only=production && npm cache clean --force
```

---

<a name="section-8"></a>
## 8. Multi-stage Builds

### ทำไมต้องใช้?

- ✅ **ลดขนาด image มาก** - ไม่เก็บ build tools
- ✅ **ความปลอดภัยสูงขึ้น** - ลด attack surface
- ✅ **แยก concerns** - build ≠ runtime
- ✅ **Optimize caching** - แยก dependencies แต่ละ stage

### ตัวอย่าง 1: Node.js Application

```dockerfile
# =============================================
# Stage 1: Dependencies
# =============================================
FROM node:18-alpine AS deps
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# =============================================
# Stage 2: Builder
# =============================================
FROM node:18-alpine AS builder
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (including devDependencies)
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# =============================================
# Stage 3: Runner (Production)
# =============================================
FROM node:18-alpine AS runner
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy production dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy built application from builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD node healthcheck.js || exit 1

# Start application
CMD ["node", "dist/index.js"]
```

**ผลลัพธ์:**
- Builder stage: ~400MB (มี build tools)
- Final image: ~80MB (เฉพาะ runtime) 🎉

### ตัวอย่าง 2: Python Flask Application

```dockerfile
# =============================================
# Stage 1: Builder
# =============================================
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python packages to user directory
RUN pip install --no-cache-dir --user -r requirements.txt

# =============================================
# Stage 2: Runtime (Production)
# =============================================
FROM python:3.11-slim AS runtime

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Install runtime dependencies ONLY
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy Python packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Environment variables
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:5000/api/health || exit 1

# Run with Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "run:app"]
```

### Build Specific Stage

```bash
# Build production (default)
docker build -t myapp:prod .

# Build specific stage
docker build --target builder -t myapp:builder .
docker build --target runtime -t myapp:runtime .

# ใช้กับ docker-compose
docker compose build --target production
```

---