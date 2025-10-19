# สรุปเนื้อหา CI/CD with GitHub Actions

> **เอกสารสรุปสำหรับการเตรียมสอบและทำแลปทดลอง CI/CD**  
> รวบรวมจากไฟล์ 01-introduction.md ถึง 12-examples.md

---

## 📚 สารบัญ

1. [ความรู้พื้นฐาน GitHub Actions](#section-1)
2. [โครงสร้าง Workflow File](#section-2)
3. [Triggers และ Events](#section-3)
4. [Jobs และ Dependencies](#section-4)
5. [Steps และ Actions](#section-5)
6. [Matrix Strategy](#section-6)
7. [Secrets และ Environment Variables](#section-7)
8. [Conditions และ Control Flow](#section-8)
9. [Caching และ Optimization](#section-9)
10. [Advanced Topics](#section-10)
11. [Best Practices](#section-11)
12. [ตัวอย่างการใช้งานจริง](#section-12)

---

<a name="section-1"></a>
## 1. ความรู้พื้นฐาน GitHub Actions

### GitHub Actions คืออะไร?

**GitHub Actions** = **CI/CD Platform** ที่ built-in มากับ GitHub โดยตรง ช่วยให้สามารถ automate workflows ต่างๆ ได้โดยอัตโนมัติ

**คำอธิบาย:**  
GitHub Actions เป็นเครื่องมือที่ช่วยให้เราสามารถทำงานซ้ำๆ แบบอัตโนมัติได้ เช่น ทุกครั้งที่มีการ push code ใหม่ ระบบจะรัน tests, build application, และ deploy ไป server โดยอัตโนมัติ โดยไม่ต้องทำด้วยมือ

**ตัวอย่างการใช้งานจริง:**
- เมื่อ developer push code → ระบบรัน tests อัตโนมัติ
- เมื่อ merge PR → ระบบ build และ deploy ไป staging
- เมื่อ create release → ระบบ deploy ไป production

### ส่วนประกอบหลัก

```
┌─────────────────────────────────────────────┐
│           GitHub Repository                 │
├─────────────────────────────────────────────┤
│  .github/                                   │
│    └── workflows/                           │
│          ├── ci.yml       ← Workflow File   │
│          └── deploy.yml                     │
└─────────────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   Workflow (ci.yml)   │
        └───────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    ┌─────┐    ┌─────┐    ┌─────┐
    │Job 1│    │Job 2│    │Job 3│
    └─────┘    └─────┘    └─────┘
        │          │          │
    ┌───┴──┐   ┌───┴──┐  ┌───┴──┐
    │Step 1│   │Step 1│  │Step 1│
    │Step 2│   │Step 2│  │Step 2│
    │Step 3│   └──────┘  │Step 3│
    └──────┘             └──────┘
```

### องค์ประกอบสำคัญ

| องค์ประกอบ | คำอธิบาย | ตัวอย่าง |
|-----------|---------|---------|
| **Workflow** | ไฟล์ YAML กำหนดขั้นตอนการทำงาน | `.github/workflows/ci.yml` |
| **Events** | เหตุการณ์ที่ trigger workflow | `push`, `pull_request`, `schedule` |
| **Jobs** | ชุดของ steps ที่รันบน runner เดียวกัน | `build`, `test`, `deploy` |
| **Steps** | คำสั่งหรือ action ที่รันเรียงลำดับ | `run: npm test` |
| **Actions** | Reusable unit of code | `actions/checkout@v4` |
| **Runners** | Server ที่รัน workflows | `ubuntu-latest`, `windows-latest` |

**คำอธิบายเพิ่มเติม:**

- **Workflow**: เปรียบเหมือนแผนงานทั้งหมด บอกว่าต้องทำอะไรบ้าง เมื่อไหร่
- **Events**: เหตุการณ์ที่กระตุ้นให้ workflow ทำงาน เช่น มี code ใหม่ ถูก push
- **Jobs**: งานใหญ่ที่แยกกันทำได้ เช่น job test, job build (รันพร้อมกันได้)
- **Steps**: ขั้นตอนย่อยในแต่ละ job ทำเรียงตามลำดับ
- **Actions**: คำสั่งสำเร็จรูปที่คนอื่นเขียนไว้แล้ว เราเอามาใช้ได้เลย
- **Runners**: เครื่อง server ที่ GitHub จัดให้ หรือเราจัดเอง สำหรับรัน jobs

### Workflow Lifecycle

```
1️⃣ Event Occurs (push, PR, schedule)
        ↓
2️⃣ Workflow Triggered
        ↓
3️⃣ Jobs Start (parallel หรือ sequential)
        ↓
4️⃣ Steps Execute (เรียงลำดับภายใน job)
        ↓
5️⃣ Workflow Completes (success/failure)
```

**คำอธิบายแต่ละขั้น:**

1. **Event Occurs**: มีเหตุการณ์เกิดขึ้น เช่น developer push code
2. **Workflow Triggered**: GitHub ตรวจจับและเริ่มรัน workflow ที่เกี่ยวข้อง
3. **Jobs Start**: เริ่มรัน jobs ตามที่กำหนด (อาจรันพร้อมกันหรือรอกัน)
4. **Steps Execute**: รันคำสั่งแต่ละขั้นตอนใน job
5. **Workflow Completes**: เสร็จสิ้น แสดงผลว่าสำเร็จหรือล้มเหลว

### ข้อดีของ GitHub Actions

- ✅ **Integrated กับ GitHub** - ไม่ต้อง setup CI/CD ภายนอก, เข้าถึงง่าย
- ✅ **Free Tier** - Public repos ฟรีไม่จำกัด, Private repos 2,000 min/month
- ✅ **Easy to Use** - YAML syntax เรียนรู้ง่าย, เริ่มต้นได้ไว
- ✅ **Powerful** - รองรับ multi-platform, matrix builds, parallel execution
- ✅ **Flexible** - Customize ได้ทุกอย่าง, สร้าง custom actions ได้
- ✅ **Community** - Actions มากมายจาก Marketplace พร้อมใช้

**ตัวอย่างความสามารถ:**

```yaml
# ตัวอย่าง workflow ง่ายๆ
name: CI

on: [push]  # ทุกครั้งที่ push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # ดึง code
      - run: npm install           # ติดตั้ง dependencies
      - run: npm test              # รัน tests
```

---

<a name="section-2"></a>
## 2. โครงสร้าง Workflow File

### ที่เก็บไฟล์ Workflow

```
my-project/
├── .github/                    ← ต้องชื่อนี้เท่านั้น (ขึ้นต้นด้วยจุด)
│   └── workflows/              ← ต้องชื่อนี้เท่านั้น
│       ├── ci.yml
│       ├── deploy.yml
│       └── test.yaml
├── src/
└── package.json
```

**คำอธิบาย:**
- โฟลเดอร์ **ต้อง** ชื่อ `.github/workflows/` เท่านั้น
- ไฟล์ workflow ลงท้าย `.yml` หรือ `.yaml`
- GitHub จะตรวจจับไฟล์ในโฟลเดอร์นี้อัตโนมัติ
- ถ้าใส่ผิดที่ workflow จะไม่ทำงาน

### โครงสร้าง YAML พื้นฐาน

**คำอธิบาย YAML:**
- YAML เป็นภาษาสำหรับเขียน configuration ที่อ่านง่าย
- ใช้การเว้นวรรค (spaces) แทนวงเล็บปีกกา
- ห้ามใช้ Tab ต้องใช้ Spaces เท่านั้น
- Case-sensitive (แยกตัวพิมพ์เล็ก-ใหญ่)

```yaml
# ==========================================
# 1. WORKFLOW NAME
# ==========================================
name: CI Pipeline
# คำอธิบาย: ชื่อ workflow ที่จะแสดงใน GitHub UI
# ถ้าไม่ใส่จะใช้ชื่อไฟล์แทน

# ==========================================
# 2. TRIGGERS (EVENTS)
# ==========================================
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
# คำอธิบาย: กำหนดว่า workflow นี้จะทำงานเมื่อไหร่
# - push ไป main หรือ develop
# - มี PR เข้ามาที่ main

# ==========================================
# 3. ENVIRONMENT VARIABLES
# ==========================================
env:
  NODE_VERSION: '18'
  DEPLOY_ENV: production
# คำอธิบาย: ตัวแปรที่ใช้ร่วมกันทั้ง workflow
# เรียกใช้ได้ทุก job ด้วย ${{ env.NODE_VERSION }}

# ==========================================
# 4. PERMISSIONS
# ==========================================
permissions:
  contents: read
  packages: write
# คำอธิบาย: กำหนดสิทธิ์ของ GITHUB_TOKEN
# - contents: read = อ่าน repository ได้
# - packages: write = เขียน packages ได้

# ==========================================
# 5. JOBS
# ==========================================
jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
```

**คำอธิบายโครงสร้าง Jobs:**

```yaml
jobs:
  job-name:                    # ID ของ job (ตั้งเองได้)
    name: Display Name         # ชื่อแสดงใน UI
    runs-on: ubuntu-latest     # OS ที่จะรัน
    
    steps:                     # รายการขั้นตอน
      - name: Step Name        # ชื่อ step
        uses: action-name      # ใช้ action สำเร็จรูป
        # หรือ
        run: command           # รันคำสั่ง
```

### Top-level Keys สำคัญ

| Key | คำอธิบาย | ตัวอย่าง |
|-----|---------|---------|
| `name:` | ชื่อ workflow แสดงใน UI | `name: CI Pipeline` |
| `run-name:` | ชื่อ run แบบ dynamic | `run-name: Deploy by @${{ github.actor }}` |
| `on:` | Events ที่ trigger | `on: [push, pull_request]` |
| `env:` | Global environment variables | `env: NODE_VERSION: '18'` |
| `permissions:` | สิทธิ์ GITHUB_TOKEN | `permissions: contents: read` |
| `concurrency:` | ควบคุม concurrent runs | `concurrency: group: ${{ github.ref }}` |
| `defaults:` | ค่า default สำหรับ jobs | `defaults: run: shell: bash` |
| `jobs:` | รายการ jobs | `jobs: test: ...` |

**คำอธิบายเพิ่มเติม:**

**`permissions:`** - ควบคุมว่า workflow มีสิทธิ์อะไรบ้าง
```yaml
permissions:
  contents: read      # อ่าน code ได้
  packages: write     # เขียน packages ได้
  pull-requests: write # comment ใน PR ได้
```

**`concurrency:`** - ป้องกัน workflow ซ้ำซ้อน
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # ยกเลิก run เก่า เมื่อมี push ใหม่
```

**`defaults:`** - ตั้งค่า default สำหรับ steps
```yaml
defaults:
  run:
    shell: bash           # ใช้ bash เป็น shell
    working-directory: ./app  # working directory
```

---

<a name="section-3"></a>
## 3. Triggers และ Events

### Push Event

**คำอธิบาย:**  
Push event เกิดขึ้นเมื่อมีการ push code ไปยัง repository เป็น event ที่ใช้บ่อยที่สุด

```yaml
# Push ทุก branch
on: push
# คำอธิบาย: ทุกครั้งที่ push ไปยัง branch ไหนก็ได้

# Push เฉพาะ branch
on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'  # ทุก branch ที่ขึ้นต้นด้วย feature/
# คำอธิบาย: ทำงานเฉพาะเมื่อ push ไป branches ที่ระบุ

# Push + Path filtering
on:
  push:
    branches: [main]
    paths:
      - 'src/**'      # ไฟล์ใน src/
      - 'tests/**'    # ไฟล์ใน tests/
      - 'package.json'
# คำอธิบาย: ทำงานเฉพาะเมื่อไฟล์ใน paths ที่ระบุมีการเปลี่ยนแปลง
# ถ้าแก้แค่ README.md จะไม่ trigger
```

**ตัวอย่างการใช้งาน:**
```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'
    paths-ignore:
      - '**.md'        # ไม่รวม markdown files
      - 'docs/**'      # ไม่รวม docs/
```

### Pull Request Event

**คำอธิบาย:**  
PR event เกิดขึ้นเมื่อมีการสร้าง, update, หรือจัดการ Pull Request

```yaml
on:
  pull_request:
    branches: [main]
    types:
      - opened          # เมื่อสร้าง PR ใหม่
      - synchronize     # เมื่อ push code ใหม่ใน PR
      - reopened        # เมื่อเปิด PR ที่ปิดไว้อีกครั้ง
```

**Types ที่ใช้ได้:**
- `opened` - สร้าง PR ใหม่
- `synchronize` - push commit ใหม่ใน PR
- `reopened` - เปิด PR ที่ถูกปิด
- `closed` - ปิด PR
- `ready_for_review` - เปลี่ยนจาก draft เป็น ready
- `labeled` - เพิ่ม label

**ตัวอย่างการใช้งาน:**
```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'src/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
      
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Tests passed!'
            })
```

### Schedule Event (Cron)

**คำอธิบาย:**  
Schedule event ใช้สำหรับรัน workflow ตามเวลาที่กำหนด เหมือน cron job

```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM UTC
    - cron: '0 9 * * 1-5'  # Weekdays at 9 AM UTC
    - cron: '*/15 * * * *'  # Every 15 minutes
```

**Cron Format:**
```
* * * * *
│ │ │ │ │
│ │ │ │ └─── Day of Week (0-6, Sunday=0)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of Month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

**ตัวอย่างที่ใช้บ่อย:**

| Description | Cron | ความถี่ |
|------------|------|---------|
| ทุกชั่วโมง | `0 * * * *` | 24 ครั้ง/วัน |
| ทุกวันเวลา 2:00 | `0 2 * * *` | 1 ครั้ง/วัน |
| ทุกวันจันทร์ 9:00 | `0 9 * * 1` | 1 ครั้ง/สัปดาห์ |
| วันแรกของเดือน | `0 0 1 * *` | 12 ครั้ง/ปี |
| ทุก 15 นาที | `*/15 * * * *` | 96 ครั้ง/วัน |

**⚠️ สำคัญ:** Cron รันใน UTC timezone
```
Bangkok (GMT+7) → UTC
09:00 BKK = 02:00 UTC
17:00 BKK = 10:00 UTC

# ต้องการรัน 9:00 เวลาไทย
cron: '0 2 * * *'  # 02:00 UTC = 09:00 Bangkok
```

**ตัวอย่างการใช้งาน:**
```yaml
name: Nightly Build

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC = 9 AM Bangkok

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - run: npm test
      
      - name: Send notification
        if: failure()
        run: echo "Build failed! Send alert"
```

### Manual Trigger (workflow_dispatch)

**คำอธิบาย:**  
workflow_dispatch ให้สามารถรัน workflow ด้วยตัวเองผ่าน GitHub UI พร้อมรับ input parameters

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
        default: 'development'
      
      version:
        description: 'Version to deploy'
        required: false
        type: string
        default: 'latest'
      
      debug_mode:
        description: 'Enable debug logging'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Deploying to: ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
          echo "Debug: ${{ inputs.debug_mode }}"
```

**Input Types:**
- `string` - ข้อความ
- `boolean` - true/false
- `choice` - เลือกจาก options
- `environment` - เลือก environment

**วิธีใช้งาน:**
1. ไปที่ Actions tab
2. เลือก workflow
3. คลิก "Run workflow"
4. กรอก inputs
5. คลิก "Run workflow"

### Combined Triggers

**คำอธิบาย:**  
สามารถกำหนด triggers หลายแบบพร้อมกันได้

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:

# workflow นี้จะทำงานเมื่อ:
# 1. Push ไป main หรือ develop
# 2. สร้าง/update PR ไปที่ main
# 3. ทุกวัน 2:00 AM UTC
# 4. กด manual trigger
```

---───────── Hour (0-23)
└─────────── Minute (0-59)
```

### Manual Trigger (workflow_dispatch)

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
      version:
        description: 'Version to deploy'
        required: false
        type: string
        default: 'latest'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
```

### Release Event

```yaml
on:
  release:
    types:
      - published
      - created

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Release: ${{ github.event.release.tag_name }}"
```

### Combined Triggers

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:
```

---

<a name="section-4"></a>
## 4. Jobs และ Dependencies

### Job Structure พื้นฐาน

```yaml
jobs:
  job-id:
    name: Job Name
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - name: Step 1
        run: echo "Hello"
```

### Runners

| Runner | Use Case | Cost Multiplier |
|--------|----------|----------------|
| `ubuntu-latest` | Web apps, APIs, Docker | 1x (แนะนำ) |
| `ubuntu-22.04` | Ubuntu 22.04 specific | 1x |
| `windows-latest` | .NET, Windows apps | 2x |
| `macos-latest` | iOS, macOS apps | 10x |
| `self-hosted` | Custom hardware | Free |

### Job Dependencies (needs)

```yaml
jobs:
  # Job 1: Build
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
  
  # Job 2: Test (รอ build เสร็จก่อน)
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  
  # Job 3: Deploy (รอทั้ง build และ test)
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

**Flow:**
```
build → test → deploy
```

### Job Outputs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    
    steps:
      - id: version
        run: echo "value=1.2.3" >> $GITHUB_OUTPUT
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

### Service Containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - run: npm test
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test
```

---

<a name="section-5"></a>
## 5. Steps และ Actions

### Run Commands

```yaml
steps:
  # Single line
  - run: npm test
  
  # Multiple lines
  - run: |
      npm ci
      npm run build
      npm test
  
  # With name
  - name: Run tests
    run: npm test
  
  # With environment
  - run: ./deploy.sh
    env:
      API_KEY: ${{ secrets.API_KEY }}
```

### Using Actions

```yaml
steps:
  # Checkout code
  - uses: actions/checkout@v4
  
  # Setup Node.js with caching
  - uses: actions/setup-node@v4
    with:
      node-version: '18'
      cache: 'npm'
  
  # Upload artifacts
  - uses: actions/upload-artifact@v4
    with:
      name: build-output
      path: dist/
  
  # Download artifacts
  - uses: actions/download-artifact@v4
    with:
      name: build-output
```

### Popular Actions

| Action | Purpose | Example |
|--------|---------|---------|
| `actions/checkout@v4` | Clone repository | `uses: actions/checkout@v4` |
| `actions/setup-node@v4` | Setup Node.js | `with: node-version: '18'` |
| `actions/setup-python@v5` | Setup Python | `with: python-version: '3.11'` |
| `actions/cache@v4` | Cache dependencies | `with: path: ~/.npm` |
| `actions/upload-artifact@v4` | Upload files | `with: name: dist` |
| `docker/build-push-action@v5` | Build/push Docker | `with: push: true` |

### Conditional Steps

```yaml
steps:
  - name: Deploy to production
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh
  
  - name: Run on success
    if: success()
    run: echo "Previous steps succeeded"
  
  - name: Run on failure
    if: failure()
    run: echo "Something failed"
  
  - name: Always run
    if: always()
    run: echo "Cleanup"
```

---

<a name="section-6"></a>
## 6. Matrix Strategy

### Basic Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]
    
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

สร้าง 3 jobs: node-16, node-18, node-20 (รัน parallel)

### Multi-dimensional Matrix

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node-version: [18, 20]
```

สร้าง 6 jobs: ubuntu-18, ubuntu-20, windows-18, windows-20, macos-18, macos-20

### Matrix Include/Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node-version: [18, 20]
    
    # เพิ่ม configuration พิเศษ
    include:
      - os: macos-latest
        node-version: 20
    
    # ลบ combination
    exclude:
      - os: windows-latest
        node-version: 18
```

### Matrix Options

```yaml
strategy:
  matrix:
    node-version: [16, 18, 20]
  
  # หยุด jobs อื่นทันทีเมื่อมี job fail
  fail-fast: true
  
  # จำกัดจำนวน parallel jobs
  max-parallel: 2
```

---

<a name="section-7"></a>
## 7. Secrets และ Environment Variables

### Secrets vs Environment Variables

| Aspect | Secrets | Environment Variables |
|--------|---------|----------------------|
| **เก็บที่** | GitHub (encrypted) | Workflow file (plain text) |
| **ความปลอดภัย** | ✅ Encrypted | ❌ Visible |
| **แสดงใน logs** | ❌ Masked (***) | ✅ Visible |
| **Use case** | Passwords, API keys | Config values, URLs |
| **การเข้าถึง** | `${{ secrets.NAME }}` | `${{ env.NAME }}` |

### การสร้าง Secrets

```
Repository → Settings → Secrets and variables → Actions
→ New repository secret
```

### การใช้ Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

### Environment Variables

```yaml
# Workflow level
env:
  NODE_VERSION: '18'
  API_URL: https://api.example.com

jobs:
  test:
    # Job level
    env:
      TEST_ENV: test
    
    steps:
      - name: Test
        # Step level
        env:
          DEBUG: true
        run: npm test
```

### GITHUB_TOKEN

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

---

<a name="section-8"></a>
## 8. Conditions และ Control Flow

### Status Check Functions

```yaml
steps:
  - name: Run on success
    if: success()
    run: echo "All previous steps passed"
  
  - name: Run on failure
    if: failure()
    run: echo "A step failed"
  
  - name: Always run (cleanup)
    if: always()
    run: rm -rf temp/
  
  - name: Run if not cancelled
    if: "!cancelled()"
    run: echo "Workflow not cancelled"
```

### Context Variables

```yaml
jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging
  
  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production
```

### Logical Operators

```yaml
# AND
if: success() && github.ref == 'refs/heads/main'

# OR
if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'

# NOT
if: "!cancelled()"

# Complex
if: |
  (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop') &&
  success() &&
  !cancelled()
```

### Continue on Error

```yaml
steps:
  - name: Optional lint
    continue-on-error: true
    run: npm run lint
  
  - name: Always runs (even if lint fails)
    run: npm test
```

### Timeout

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - name: Long test
        timeout-minutes: 15
        run: npm run test:integration
```

---

<a name="section-9"></a>
## 9. Caching และ Optimization

### Built-in Caching

```yaml
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: '18'
      cache: 'npm'  # ← Auto cache ~/.npm
  
  - run: npm ci
  - run: npm test
```

**รองรับ:** npm, yarn, pnpm, pip, gradle, maven

### Manual Caching

```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-npm-
  
  - run: npm ci
```

### Docker Layer Caching

```yaml
- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Concurrency Control

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

ยกเลิก workflow runs ที่ยังไม่เสร็จเมื่อมี push ใหม่

### Path Filters

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package.json'
```

รันเฉพาะเมื่อไฟล์ที่ระบุมีการเปลี่ยนแปลง

---

<a name="section-10"></a>
## 10. Advanced Topics

### Reusable Workflows

**คำอธิบาย:**  
Reusable workflows คือ workflows ที่สามารถเรียกใช้จาก workflows อื่นได้ ช่วยลด code ซ้ำซ้อนและทำให้ดูแลง่าย

**ประโยชน์:**
- ✅ **ลด duplication** - เขียนครั้งเดียว ใช้ได้หลาย workflows
- ✅ **Centralized maintenance** - แก้ที่เดียว มีผลทุกที่
- ✅ **Consistent standards** - ทีมใช้ standards เดียวกัน
- ✅ **Easy to update** - update ง่าย ไม่ต้องแก้หลายไฟล์

**การสร้าง Reusable Workflow:**

**ไฟล์: `.github/workflows/reusable-deploy.yml`**
```yaml
name: Reusable Deploy Workflow

on:
  workflow_call:  # ← สำคัญ! บอกว่าเป็น reusable
    inputs:
      environment:
        required: true
        type: string
        description: 'Environment to deploy (dev/staging/prod)'
      version:
        required: false
        type: string
        default: 'latest'
    secrets:
      deploy-token:
        required: true
    outputs:
      deployment-url:
        description: "URL ของ deployment"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to ${{ inputs.environment }}
        id: deploy
        run: |
          echo "Deploying version ${{ inputs.version }} to ${{ inputs.environment }}"
          ./deploy.sh --env=${{ inputs.environment }} --version=${{ inputs.version }}
          
          URL="https://${{ inputs.environment }}.myapp.com"
          echo "url=$URL" >> $GITHUB_OUTPUT
      
      - name: Verify deployment
        run: curl -f ${{ steps.deploy.outputs.url }}/health
```

**การเรียกใช้:**

**ไฟล์: `.github/workflows/main.yml`**
```yaml
name: Main CI/CD

on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
  
  # เรียกใช้ reusable workflow สำหรับ staging
  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/develop'
    uses: ./.github/workflows/reusable-deploy.yml  # ← เรียกใช้
    with:
      environment: staging
      version: ${{ github.sha }}
    secrets:
      deploy-token: ${{ secrets.STAGING_DEPLOY_TOKEN }}
  
  # เรียกใช้ reusable workflow สำหรับ production
  deploy-production:
    needs: test
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      version: ${{ github.sha }}
    secrets:
      deploy-token: ${{ secrets.PROD_DEPLOY_TOKEN }}
  
  # ใช้ output จาก reusable workflow
  notify:
    needs: deploy-production
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deployed to ${{ needs.deploy-production.outputs.deployment-url }}"
```

**คำอธิบาย:**
1. Reusable workflow ใช้ `workflow_call` trigger
2. กำหนด inputs, secrets, outputs ที่ต้องการ
3. Workflow อื่นเรียกใช้ด้วย `uses:` แล้วส่ง parameters ผ่าน `with:`

### Composite Actions

**คำอธิบาย:**  
Composite actions คือ custom actions ที่รวมหลาย steps เป็น action เดียว ใช้ซ้ำได้ง่าย

**ความแตกต่าง Reusable Workflow vs Composite Action:**

| Feature | Reusable Workflow | Composite Action |
|---------|-------------------|------------------|
| **Level** | Job-level | Step-level |
| **Can use** | Jobs | Steps only |
| **Where** | `.github/workflows/` | `.github/actions/` |
| **Call with** | `uses:` in jobs | `uses:` in steps |

**การสร้าง Composite Action:**

**ไฟล์: `.github/actions/setup-project/action.yml`**
```yaml
name: 'Setup Project'
description: 'Setup Node.js and install dependencies'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '18'

runs:
  using: 'composite'  # ← สำคัญ!
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      shell: bash
      run: npm ci
    
    - name: Show versions
      shell: bash
      run: |
        node --version
        npm --version
```

**การใช้งาน:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # ใช้ composite action
      - uses: ./.github/actions/setup-project
        with:
          node-version: '20'
      
      - run: npm test
```

**คำอธิบาย:**
- Composite action ใช้ `using: 'composite'`
- แต่ละ step ต้องระบุ `shell:`
- เรียกใช้เหมือน action ปกติ

### Deployment Environments

**คำอธิบาย:**  
Deployment environments ใช้สำหรับจัดการ deployment พร้อม protection rules

**ความสามารถ:**
- ✅ **Required reviewers** - ต้องมีคน approve ก่อน deploy
- ✅ **Wait timer** - รอระยะเวลาก่อน deploy
- ✅ **Deployment branches** - จำกัด branches ที่ deploy ได้
- ✅ **Environment secrets** - secrets เฉพาะ environment

**การตั้งค่า:**
```
Repository → Settings → Environments → New environment
```

**การใช้งาน:**
```yaml
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment:
      name: production  # ชื่อ environment
      url: https://myapp.com  # Deployment URL
    
    steps:
      - name: Wait for approval
        run: echo "Waiting for required reviewers..."
      
      - name: Deploy
        run: ./deploy-prod.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}  # production API_KEY
```

**Timeline:**
```
1. Workflow starts
2. Job ถึง production environment
3. ⏸️ รอ required reviewers approve
4. ⏸️ รอตาม wait timer (ถ้ามี)
5. ✅ ตรวจสอบ deployment branches
6. 🚀 Deploy
```

---

<a name="section-11"></a>
## 11. Best Practices

### Security Best Practices

**1. ใช้ Least Privilege Permissions**

**คำอธิบาย:**  
ให้สิทธิ์แค่พอเท่าที่จำเป็น ไม่ให้เกิน

```yaml
# ❌ Bad - ให้สิทธิ์มากเกินไป
permissions: write-all  # อันตราย!

# ✅ Good - ให้สิทธิ์เฉพาะที่จำเป็น
permissions:
  contents: read      # อ่านเท่านั้น
  packages: write     # เขียนเฉพาะ packages
  pull-requests: write
```

**2. ไม่ Hardcode Secrets**

```yaml
# ❌ Bad
- run: curl -H "Authorization: Bearer abc123xyz..."

# ✅ Good
- run: curl -H "Authorization: Bearer ${{ secrets.API_TOKEN }}"
```

**3. Pin Action Versions**

**คำอธิบาย:**  
กำหนด version ของ actions เพื่อความปลอดภัยและ stability

```yaml
# ❌ Bad - ไม่ stable
- uses: actions/checkout@main

# ✅ Good - Major version (แนะนำ)
- uses: actions/checkout@v4

# ✅ Better - Specific version
- uses: actions/checkout@v4.1.2

# ✅ Best - SHA (สำหรับ security-critical)
- uses: actions/checkout@8f4b7f84864484a7bf31766abe9204da3cbe65b3
```

**4. Validate Inputs**

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice  # ✅ จำกัด choices
        options: [dev, staging, production]

jobs:
  deploy:
    steps:
      # ✅ Validate ก่อนใช้
      - name: Validate input
        run: |
          VALID_ENVS="dev staging production"
          if [[ ! $VALID_ENVS =~ ${{ inputs.environment }} ]]; then
            echo "❌ Invalid environment"
            exit 1
          fi
```

**5. ป้องกัน Script Injection**

```yaml
# ❌ Bad - เสี่ยง injection
- run: echo "Hello ${{ github.event.issue.title }}"

# ✅ Good - ใช้ environment variable
- run: echo "Hello $TITLE"
  env:
    TITLE: ${{ github.event.issue.title }}
```

### Performance Optimization

**1. ใช้ Caching**

```yaml
# ✅ Built-in caching (ง่ายที่สุด)
- uses: actions/setup-node@v4
  with:
    cache: 'npm'

# ✅ Manual caching (ยืดหยุ่นกว่า)
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
```

**2. Path Filters**

```yaml
# ✅ รันเฉพาะเมื่อ code เปลี่ยน
on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
    paths-ignore:
      - '**.md'
```

**3. Concurrency Control**

```yaml
# ✅ ยกเลิก runs ซ้ำซ้อน
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

**4. ใช้ Ubuntu (เร็วสุด, ฟรีมากสุด)**

```yaml
# ✅ Ubuntu - แนะนำ
runs-on: ubuntu-latest

# ⚠️ Windows - ช้า, ใช้ minutes 2x
runs-on: windows-latest

# ⚠️ macOS - ช้ามาก, ใช้ minutes 10x
runs-on: macos-latest
```

### Error Handling

**คำอธิบาย:**  
จัดการ errors อย่างเหมาะสม

```yaml
steps:
  # Optional step - ไม่สำคัญถ้า fail
  - name: Lint
    continue-on-error: true
    run: npm run lint
  
  # Always cleanup
  - name: Cleanup
    if: always()
    run: rm -rf temp/
  
  # Set timeout
  - name: Long test
    timeout-minutes: 30
    run: npm run test:e2e
  
  # Retry on failure
  - name: Deploy with retry
    uses: nick-invision/retry@v2
    with:
      timeout_minutes: 10
      max_attempts: 3
      command: ./deploy.sh
```

---

<a name="section-12"></a>
## 12. ตัวอย่างการใช้งานจริง

### Example 1: Complete Node.js CI/CD

**คำอธิบาย:**  
Workflow สมบูรณ์สำหรับ Node.js application ครอบคลุม quality checks, tests, build, และ deploy

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package*.json'
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'

permissions:
  contents: read
  packages: write
  pull-requests: write

jobs:
  # =============================
  # JOB 1: Code Quality Checks
  # =============================
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Format check
        run: npm run format:check
      
      - name: Type check
        run: npm run type-check
  
  # =============================
  # JOB 2: Tests
  # =============================
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - name: Unit tests
        run: npm run test:unit
      
      - name: Integration tests
        run: npm run test:integration
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: always()
  
  # =============================
  # JOB 3: Build
  # =============================
  build:
    name: Build Application
    needs: [quality, test]  # รอให้ quality และ test ผ่านก่อน
    runs-on: ubuntu-latest
    
    outputs:
      version: ${{ steps.version.outputs.value }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      
      - name: Get version
        id: version
        run: echo "value=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
  
  # =============================
  # JOB 4: Deploy to Staging
  # =============================
  deploy-staging:
    name: Deploy to Staging
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
      
      - name: Deploy
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }}"
          # Deploy commands here
  
  # =============================
  # JOB 5: Deploy to Production
  # =============================
  deploy-production:
    name: Deploy to Production
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
      
      - name: Deploy
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }}"
          # Deploy commands here
      
      - name: Health check
        run: curl -f https://myapp.com/health
```

**คำอธิบายการทำงาน:**
1. **Quality** - ตรวจสอบ code quality (lint, format, type check)
2. **Test** - รัน tests และ upload coverage
3. **Build** - Build application และ upload artifacts
4. **Deploy** - Deploy ไป staging/production ตาม branch

---

## 📋 Checklist สำหรับ Lab CI/CD

### ก่อนเริ่มทำ Lab

- [ ] มี GitHub Account แล้ว
- [ ] ติดตั้ง Git แล้ว
- [ ] ติดตั้ง Docker Desktop แล้ว
- [ ] ติดตั้ง Python/Node.js ตาม Lab
- [ ] เตรียม Code Editor (VS Code แนะนำ)

### การสร้าง Workflow

- [ ] สร้างโฟลเดอร์ `.github/workflows/`
- [ ] สร้างไฟล์ workflow (`.yml` หรือ `.yaml`)
- [ ] กำหนด `name:` ที่ชัดเจน
- [ ] เลือก triggers (`on:`) ที่เหมาะสม
- [ ] ตั้งค่า permissions ถ้าจำเป็น
- [ ] ตรวจสอบ syntax ด้วย `yamllint` หรือ VS Code

### การเขียน Jobs

- [ ] ตั้งชื่อ jobs ที่บอกได้ว่าทำอะไร
- [ ] เลือก `runs-on:` ที่เหมาะสม (แนะนำ ubuntu-latest)
- [ ] กำหนด dependencies (`needs:`) ถ้าจำเป็น
- [ ] ตั้ง `timeout-minutes:` สำหรับ jobs ที่อาจนาน

### การเขียน Steps

- [ ] ใส่ `name:` ทุก step
- [ ] ใช้ actions จาก marketplace (เช่น `actions/checkout@v4`)
- [ ] ใช้ version tags แทน branches
- [ ] เพิ่ม error handling (`continue-on-error`, `if:`)

### Testing & Quality

- [ ] มี unit tests
- [ ] มี linting/formatting checks
- [ ] วัด code coverage
- [ ] ทดสอบใน service containers ถ้าจำเป็น

### Secrets & Security

- [ ] ไม่ hardcode secrets ในโค้ด
- [ ] สร้าง secrets ใน GitHub Settings
- [ ] ใช้ `${{ secrets.NAME }}` เท่านั้น
- [ ] ตั้ง permissions ที่จำเป็นเท่านั้น

### Deployment

- [ ] ใช้ environment protection
- [ ] ตั้ง required reviewers สำหรับ production
- [ ] มี health check หลัง deploy
- [ ] บันทึก deployment URL

### Optimization

- [ ] ใช้ caching สำหรับ dependencies
- [ ] ใช้ path filters ถ้าเหมาะสม
- [ ] ตั้ง concurrency control
- [ ] ใช้ matrix สำหรับ multi-version testing

---

## 🎯 คำถามสำคัญสำหรับทำข้อสอบ

### พื้นฐาน

**1. GitHub Actions คืออะไร?**
- CI/CD Platform ที่ built-in กับ GitHub
- ใช้ automate workflows (build, test, deploy)
- ใช้ YAML files กำหนดค่า

**2. Workflow ต้องเก็บไว้ที่ไหน?**
- `.github/workflows/` เท่านั้น
- ไฟล์ลงท้าย `.yml` หรือ `.yaml`

**3. Jobs, Steps, Actions แตกต่างกันอย่างไร?**
- **Job**: ชุดของ steps ที่รันบน runner เดียวกัน
- **Step**: คำสั่งเดี่ยวที่รันเรียงลำดับ
- **Action**: Reusable unit ใช้ด้วย `uses:`

**4. Jobs รันแบบ parallel หรือ sequential?**
- Default: parallel (พร้อมกัน)
- ใช้ `needs:` เพื่อทำให้รันเรียงลำดับ

**5. Matrix strategy ใช้ทำอะไร?**
- รัน job เดียวกันหลาย configurations พร้อมกัน
- เช่น test หลาย versions ของ Node.js

### Triggers

**6. Events ที่ trigger workflow ได้มีอะไรบ้าง?**
- `push`, `pull_request`, `schedule`, `workflow_dispatch`, `release`

**7. Cron schedule ใช้งานอย่างไร?**
- Format: `* * * * *` (minute hour day month weekday)
- เช่น `0 2 * * *` = ทุกวัน 2:00 AM UTC
- ⚠️ รันใน UTC timezone

**8. workflow_dispatch ใช้ทำอะไร?**
- Manual trigger workflow ผ่าน GitHub UI
- สามารถรับ inputs จากผู้ใช้

### Secrets & Security

**9. ความแตกต่างระหว่าง Secrets และ Environment Variables?**
- **Secrets**: encrypted, masked ใน logs, ใช้สำหรับ passwords/keys
- **Env vars**: plain text, visible, ใช้สำหรับ config values

**10. GITHUB_TOKEN คืออะไร?**
- Token ที่ GitHub สร้างให้อัตโนมัติแต่ละ workflow run
- ใช้ authenticate กับ GitHub API

### Optimization

**11. Caching ช่วยอะไร?**
- ลดเวลา download dependencies
- ประหยัด bandwidth และ minutes

**12. Built-in caching vs Manual caching?**
- **Built-in**: ง่าย แค่เพิ่ม `cache:` parameter
- **Manual**: ยืดหยุ่นกว่า ใช้ `actions/cache@v4`

**13. Concurrency control คืออะไร?**
- ควบคุมจำนวน workflow runs พร้อมกัน
- ยกเลิก runs เก่าเมื่อมี push ใหม่

---

**Good luck กับการทำข้อสอบและ Lab CI/CD! 🚀**# สรุปเนื้อหา CI/CD with GitHub Actions

> **เอกสารสรุปสำหรับการเตรียมสอบและทำแลปทดลอง CI/CD**  
> รวบรวมจากไฟล์ 01-introduction.md ถึง 12-examples.md

---

## 📚 สารบัญ

1. [ความรู้พื้นฐาน GitHub Actions](#section-1)
2. [โครงสร้าง Workflow File](#section-2)
3. [Triggers และ Events](#section-3)
4. [Jobs และ Dependencies](#section-4)
5. [Steps และ Actions](#section-5)
6. [Matrix Strategy](#section-6)
7. [Secrets และ Environment Variables](#section-7)
8. [Conditions และ Control Flow](#section-8)
9. [Caching และ Optimization](#section-9)
10. [Advanced Topics](#section-10)
11. [Best Practices](#section-11)
12. [ตัวอย่างการใช้งานจริง](#section-12)

---

<a name="section-1"></a>
## 1. ความรู้พื้นฐาน GitHub Actions

### GitHub Actions คืออะไร?

**GitHub Actions** = **CI/CD Platform** ที่ built-in มากับ GitHub โดยตรง ช่วยให้สามารถ automate workflows ต่างๆ ได้โดยอัตโนมัติ

**คำอธิบาย:**  
GitHub Actions เป็นเครื่องมือที่ช่วยให้เราสามารถทำงานซ้ำๆ แบบอัตโนมัติได้ เช่น ทุกครั้งที่มีการ push code ใหม่ ระบบจะรัน tests, build application, และ deploy ไป server โดยอัตโนมัติ โดยไม่ต้องทำด้วยมือ

**ตัวอย่างการใช้งานจริง:**
- เมื่อ developer push code → ระบบรัน tests อัตโนมัติ
- เมื่อ merge PR → ระบบ build และ deploy ไป staging
- เมื่อ create release → ระบบ deploy ไป production

### ส่วนประกอบหลัก

```
┌─────────────────────────────────────────────┐
│           GitHub Repository                 │
├─────────────────────────────────────────────┤
│  .github/                                   │
│    └── workflows/                           │
│          ├── ci.yml       ← Workflow File   │
│          └── deploy.yml                     │
└─────────────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   Workflow (ci.yml)   │
        └───────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    ┌─────┐    ┌─────┐    ┌─────┐
    │Job 1│    │Job 2│    │Job 3│
    └─────┘    └─────┘    └─────┘
        │          │          │
    ┌───┴──┐   ┌───┴──┐  ┌───┴──┐
    │Step 1│   │Step 1│  │Step 1│
    │Step 2│   │Step 2│  │Step 2│
    │Step 3│   └──────┘  │Step 3│
    └──────┘             └──────┘
```

### องค์ประกอบสำคัญ

| องค์ประกอบ | คำอธิบาย | ตัวอย่าง |
|-----------|---------|---------|
| **Workflow** | ไฟล์ YAML กำหนดขั้นตอนการทำงาน | `.github/workflows/ci.yml` |
| **Events** | เหตุการณ์ที่ trigger workflow | `push`, `pull_request`, `schedule` |
| **Jobs** | ชุดของ steps ที่รันบน runner เดียวกัน | `build`, `test`, `deploy` |
| **Steps** | คำสั่งหรือ action ที่รันเรียงลำดับ | `run: npm test` |
| **Actions** | Reusable unit of code | `actions/checkout@v4` |
| **Runners** | Server ที่รัน workflows | `ubuntu-latest`, `windows-latest` |

**คำอธิบายเพิ่มเติม:**

- **Workflow**: เปรียบเหมือนแผนงานทั้งหมด บอกว่าต้องทำอะไรบ้าง เมื่อไหร่
- **Events**: เหตุการณ์ที่กระตุ้นให้ workflow ทำงาน เช่น มี code ใหม่ ถูก push
- **Jobs**: งานใหญ่ที่แยกกันทำได้ เช่น job test, job build (รันพร้อมกันได้)
- **Steps**: ขั้นตอนย่อยในแต่ละ job ทำเรียงตามลำดับ
- **Actions**: คำสั่งสำเร็จรูปที่คนอื่นเขียนไว้แล้ว เราเอามาใช้ได้เลย
- **Runners**: เครื่อง server ที่ GitHub จัดให้ หรือเราจัดเอง สำหรับรัน jobs

### Workflow Lifecycle

```
1️⃣ Event Occurs (push, PR, schedule)
        ↓
2️⃣ Workflow Triggered
        ↓
3️⃣ Jobs Start (parallel หรือ sequential)
        ↓
4️⃣ Steps Execute (เรียงลำดับภายใน job)
        ↓
5️⃣ Workflow Completes (success/failure)
```

**คำอธิบายแต่ละขั้น:**

1. **Event Occurs**: มีเหตุการณ์เกิดขึ้น เช่น developer push code
2. **Workflow Triggered**: GitHub ตรวจจับและเริ่มรัน workflow ที่เกี่ยวข้อง
3. **Jobs Start**: เริ่มรัน jobs ตามที่กำหนด (อาจรันพร้อมกันหรือรอกัน)
4. **Steps Execute**: รันคำสั่งแต่ละขั้นตอนใน job
5. **Workflow Completes**: เสร็จสิ้น แสดงผลว่าสำเร็จหรือล้มเหลว

### ข้อดีของ GitHub Actions

- ✅ **Integrated กับ GitHub** - ไม่ต้อง setup CI/CD ภายนอก, เข้าถึงง่าย
- ✅ **Free Tier** - Public repos ฟรีไม่จำกัด, Private repos 2,000 min/month
- ✅ **Easy to Use** - YAML syntax เรียนรู้ง่าย, เริ่มต้นได้ไว
- ✅ **Powerful** - รองรับ multi-platform, matrix builds, parallel execution
- ✅ **Flexible** - Customize ได้ทุกอย่าง, สร้าง custom actions ได้
- ✅ **Community** - Actions มากมายจาก Marketplace พร้อมใช้

**ตัวอย่างความสามารถ:**

```yaml
# ตัวอย่าง workflow ง่ายๆ
name: CI

on: [push]  # ทุกครั้งที่ push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # ดึง code
      - run: npm install           # ติดตั้ง dependencies
      - run: npm test              # รัน tests
```

---

<a name="section-2"></a>
## 2. โครงสร้าง Workflow File

### ที่เก็บไฟล์ Workflow

```
my-project/
├── .github/                    ← ต้องชื่อนี้เท่านั้น (ขึ้นต้นด้วยจุด)
│   └── workflows/              ← ต้องชื่อนี้เท่านั้น
│       ├── ci.yml
│       ├── deploy.yml
│       └── test.yaml
├── src/
└── package.json
```

**คำอธิบาย:**
- โฟลเดอร์ **ต้อง** ชื่อ `.github/workflows/` เท่านั้น
- ไฟล์ workflow ลงท้าย `.yml` หรือ `.yaml`
- GitHub จะตรวจจับไฟล์ในโฟลเดอร์นี้อัตโนมัติ
- ถ้าใส่ผิดที่ workflow จะไม่ทำงาน

### โครงสร้าง YAML พื้นฐาน

**คำอธิบาย YAML:**
- YAML เป็นภาษาสำหรับเขียน configuration ที่อ่านง่าย
- ใช้การเว้นวรรค (spaces) แทนวงเล็บปีกกา
- ห้ามใช้ Tab ต้องใช้ Spaces เท่านั้น
- Case-sensitive (แยกตัวพิมพ์เล็ก-ใหญ่)

```yaml
# ==========================================
# 1. WORKFLOW NAME
# ==========================================
name: CI Pipeline
# คำอธิบาย: ชื่อ workflow ที่จะแสดงใน GitHub UI
# ถ้าไม่ใส่จะใช้ชื่อไฟล์แทน

# ==========================================
# 2. TRIGGERS (EVENTS)
# ==========================================
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
# คำอธิบาย: กำหนดว่า workflow นี้จะทำงานเมื่อไหร่
# - push ไป main หรือ develop
# - มี PR เข้ามาที่ main

# ==========================================
# 3. ENVIRONMENT VARIABLES
# ==========================================
env:
  NODE_VERSION: '18'
  DEPLOY_ENV: production
# คำอธิบาย: ตัวแปรที่ใช้ร่วมกันทั้ง workflow
# เรียกใช้ได้ทุก job ด้วย ${{ env.NODE_VERSION }}

# ==========================================
# 4. PERMISSIONS
# ==========================================
permissions:
  contents: read
  packages: write
# คำอธิบาย: กำหนดสิทธิ์ของ GITHUB_TOKEN
# - contents: read = อ่าน repository ได้
# - packages: write = เขียน packages ได้

# ==========================================
# 5. JOBS
# ==========================================
jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
```

**คำอธิบายโครงสร้าง Jobs:**

```yaml
jobs:
  job-name:                    # ID ของ job (ตั้งเองได้)
    name: Display Name         # ชื่อแสดงใน UI
    runs-on: ubuntu-latest     # OS ที่จะรัน
    
    steps:                     # รายการขั้นตอน
      - name: Step Name        # ชื่อ step
        uses: action-name      # ใช้ action สำเร็จรูป
        # หรือ
        run: command           # รันคำสั่ง
```

### Top-level Keys สำคัญ

| Key | คำอธิบาย | ตัวอย่าง |
|-----|---------|---------|
| `name:` | ชื่อ workflow แสดงใน UI | `name: CI Pipeline` |
| `run-name:` | ชื่อ run แบบ dynamic | `run-name: Deploy by @${{ github.actor }}` |
| `on:` | Events ที่ trigger | `on: [push, pull_request]` |
| `env:` | Global environment variables | `env: NODE_VERSION: '18'` |
| `permissions:` | สิทธิ์ GITHUB_TOKEN | `permissions: contents: read` |
| `concurrency:` | ควบคุม concurrent runs | `concurrency: group: ${{ github.ref }}` |
| `defaults:` | ค่า default สำหรับ jobs | `defaults: run: shell: bash` |
| `jobs:` | รายการ jobs | `jobs: test: ...` |

**คำอธิบายเพิ่มเติม:**

**`permissions:`** - ควบคุมว่า workflow มีสิทธิ์อะไรบ้าง
```yaml
permissions:
  contents: read      # อ่าน code ได้
  packages: write     # เขียน packages ได้
  pull-requests: write # comment ใน PR ได้
```

**`concurrency:`** - ป้องกัน workflow ซ้ำซ้อน
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # ยกเลิก run เก่า เมื่อมี push ใหม่
```

**`defaults:`** - ตั้งค่า default สำหรับ steps
```yaml
defaults:
  run:
    shell: bash           # ใช้ bash เป็น shell
    working-directory: ./app  # working directory
```

---

<a name="section-3"></a>
## 3. Triggers และ Events

### Push Event

**คำอธิบาย:**  
Push event เกิดขึ้นเมื่อมีการ push code ไปยัง repository เป็น event ที่ใช้บ่อยที่สุด

```yaml
# Push ทุก branch
on: push
# คำอธิบาย: ทุกครั้งที่ push ไปยัง branch ไหนก็ได้

# Push เฉพาะ branch
on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'  # ทุก branch ที่ขึ้นต้นด้วย feature/
# คำอธิบาย: ทำงานเฉพาะเมื่อ push ไป branches ที่ระบุ

# Push + Path filtering
on:
  push:
    branches: [main]
    paths:
      - 'src/**'      # ไฟล์ใน src/
      - 'tests/**'    # ไฟล์ใน tests/
      - 'package.json'
# คำอธิบาย: ทำงานเฉพาะเมื่อไฟล์ใน paths ที่ระบุมีการเปลี่ยนแปลง
# ถ้าแก้แค่ README.md จะไม่ trigger
```

**ตัวอย่างการใช้งาน:**
```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'
    paths-ignore:
      - '**.md'        # ไม่รวม markdown files
      - 'docs/**'      # ไม่รวม docs/
```

### Pull Request Event

**คำอธิบาย:**  
PR event เกิดขึ้นเมื่อมีการสร้าง, update, หรือจัดการ Pull Request

```yaml
on:
  pull_request:
    branches: [main]
    types:
      - opened          # เมื่อสร้าง PR ใหม่
      - synchronize     # เมื่อ push code ใหม่ใน PR
      - reopened        # เมื่อเปิด PR ที่ปิดไว้อีกครั้ง
```

**Types ที่ใช้ได้:**
- `opened` - สร้าง PR ใหม่
- `synchronize` - push commit ใหม่ใน PR
- `reopened` - เปิด PR ที่ถูกปิด
- `closed` - ปิด PR
- `ready_for_review` - เปลี่ยนจาก draft เป็น ready
- `labeled` - เพิ่ม label

**ตัวอย่างการใช้งาน:**
```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'src/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
      
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Tests passed!'
            })
```

### Schedule Event (Cron)

**คำอธิบาย:**  
Schedule event ใช้สำหรับรัน workflow ตามเวลาที่กำหนด เหมือน cron job

```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM UTC
    - cron: '0 9 * * 1-5'  # Weekdays at 9 AM UTC
    - cron: '*/15 * * * *'  # Every 15 minutes
```

**Cron Format:**
```
* * * * *
│ │ │ │ │
│ │ │ │ └─── Day of Week (0-6, Sunday=0)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of Month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

**ตัวอย่างที่ใช้บ่อย:**

| Description | Cron | ความถี่ |
|------------|------|---------|
| ทุกชั่วโมง | `0 * * * *` | 24 ครั้ง/วัน |
| ทุกวันเวลา 2:00 | `0 2 * * *` | 1 ครั้ง/วัน |
| ทุกวันจันทร์ 9:00 | `0 9 * * 1` | 1 ครั้ง/สัปดาห์ |
| วันแรกของเดือน | `0 0 1 * *` | 12 ครั้ง/ปี |
| ทุก 15 นาที | `*/15 * * * *` | 96 ครั้ง/วัน |

**⚠️ สำคัญ:** Cron รันใน UTC timezone
```
Bangkok (GMT+7) → UTC
09:00 BKK = 02:00 UTC
17:00 BKK = 10:00 UTC

# ต้องการรัน 9:00 เวลาไทย
cron: '0 2 * * *'  # 02:00 UTC = 09:00 Bangkok
```

**ตัวอย่างการใช้งาน:**
```yaml
name: Nightly Build

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC = 9 AM Bangkok

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - run: npm test
      
      - name: Send notification
        if: failure()
        run: echo "Build failed! Send alert"
```

### Manual Trigger (workflow_dispatch)

**คำอธิบาย:**  
workflow_dispatch ให้สามารถรัน workflow ด้วยตัวเองผ่าน GitHub UI พร้อมรับ input parameters

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
        default: 'development'
      
      version:
        description: 'Version to deploy'
        required: false
        type: string
        default: 'latest'
      
      debug_mode:
        description: 'Enable debug logging'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Deploying to: ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
          echo "Debug: ${{ inputs.debug_mode }}"
```

**Input Types:**
- `string` - ข้อความ
- `boolean` - true/false
- `choice` - เลือกจาก options
- `environment` - เลือก environment

**วิธีใช้งาน:**
1. ไปที่ Actions tab
2. เลือก workflow
3. คลิก "Run workflow"
4. กรอก inputs
5. คลิก "Run workflow"

### Combined Triggers

**คำอธิบาย:**  
สามารถกำหนด triggers หลายแบบพร้อมกันได้

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:

# workflow นี้จะทำงานเมื่อ:
# 1. Push ไป main หรือ develop
# 2. สร้าง/update PR ไปที่ main
# 3. ทุกวัน 2:00 AM UTC
# 4. กด manual trigger
```

---───────── Hour (0-23)
└─────────── Minute (0-59)
```

### Manual Trigger (workflow_dispatch)

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
      version:
        description: 'Version to deploy'
        required: false
        type: string
        default: 'latest'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
```

### Release Event

```yaml
on:
  release:
    types:
      - published
      - created

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Release: ${{ github.event.release.tag_name }}"
```

### Combined Triggers

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:
```

---

<a name="section-4"></a>
## 4. Jobs และ Dependencies

### Job Structure พื้นฐาน

```yaml
jobs:
  job-id:
    name: Job Name
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - name: Step 1
        run: echo "Hello"
```

### Runners

| Runner | Use Case | Cost Multiplier |
|--------|----------|----------------|
| `ubuntu-latest` | Web apps, APIs, Docker | 1x (แนะนำ) |
| `ubuntu-22.04` | Ubuntu 22.04 specific | 1x |
| `windows-latest` | .NET, Windows apps | 2x |
| `macos-latest` | iOS, macOS apps | 10x |
| `self-hosted` | Custom hardware | Free |

### Job Dependencies (needs)

```yaml
jobs:
  # Job 1: Build
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
  
  # Job 2: Test (รอ build เสร็จก่อน)
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  
  # Job 3: Deploy (รอทั้ง build และ test)
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

**Flow:**
```
build → test → deploy
```

### Job Outputs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    
    steps:
      - id: version
        run: echo "value=1.2.3" >> $GITHUB_OUTPUT
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

### Service Containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - run: npm test
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test
```

---

<a name="section-5"></a>
## 5. Steps และ Actions

### Run Commands

```yaml
steps:
  # Single line
  - run: npm test
  
  # Multiple lines
  - run: |
      npm ci
      npm run build
      npm test
  
  # With name
  - name: Run tests
    run: npm test
  
  # With environment
  - run: ./deploy.sh
    env:
      API_KEY: ${{ secrets.API_KEY }}
```

### Using Actions

```yaml
steps:
  # Checkout code
  - uses: actions/checkout@v4
  
  # Setup Node.js with caching
  - uses: actions/setup-node@v4
    with:
      node-version: '18'
      cache: 'npm'
  
  # Upload artifacts
  - uses: actions/upload-artifact@v4
    with:
      name: build-output
      path: dist/
  
  # Download artifacts
  - uses: actions/download-artifact@v4
    with:
      name: build-output
```

### Popular Actions

| Action | Purpose | Example |
|--------|---------|---------|
| `actions/checkout@v4` | Clone repository | `uses: actions/checkout@v4` |
| `actions/setup-node@v4` | Setup Node.js | `with: node-version: '18'` |
| `actions/setup-python@v5` | Setup Python | `with: python-version: '3.11'` |
| `actions/cache@v4` | Cache dependencies | `with: path: ~/.npm` |
| `actions/upload-artifact@v4` | Upload files | `with: name: dist` |
| `docker/build-push-action@v5` | Build/push Docker | `with: push: true` |

### Conditional Steps

```yaml
steps:
  - name: Deploy to production
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh
  
  - name: Run on success
    if: success()
    run: echo "Previous steps succeeded"
  
  - name: Run on failure
    if: failure()
    run: echo "Something failed"
  
  - name: Always run
    if: always()
    run: echo "Cleanup"
```

---

<a name="section-6"></a>
## 6. Matrix Strategy

### Basic Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]
    
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

สร้าง 3 jobs: node-16, node-18, node-20 (รัน parallel)

### Multi-dimensional Matrix

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node-version: [18, 20]
```

สร้าง 6 jobs: ubuntu-18, ubuntu-20, windows-18, windows-20, macos-18, macos-20

### Matrix Include/Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node-version: [18, 20]
    
    # เพิ่ม configuration พิเศษ
    include:
      - os: macos-latest
        node-version: 20
    
    # ลบ combination
    exclude:
      - os: windows-latest
        node-version: 18
```

### Matrix Options

```yaml
strategy:
  matrix:
    node-version: [16, 18, 20]
  
  # หยุด jobs อื่นทันทีเมื่อมี job fail
  fail-fast: true
  
  # จำกัดจำนวน parallel jobs
  max-parallel: 2
```

---

<a name="section-7"></a>
## 7. Secrets และ Environment Variables

### Secrets vs Environment Variables

| Aspect | Secrets | Environment Variables |
|--------|---------|----------------------|
| **เก็บที่** | GitHub (encrypted) | Workflow file (plain text) |
| **ความปลอดภัย** | ✅ Encrypted | ❌ Visible |
| **แสดงใน logs** | ❌ Masked (***) | ✅ Visible |
| **Use case** | Passwords, API keys | Config values, URLs |
| **การเข้าถึง** | `${{ secrets.NAME }}` | `${{ env.NAME }}` |

### การสร้าง Secrets

```
Repository → Settings → Secrets and variables → Actions
→ New repository secret
```

### การใช้ Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

### Environment Variables

```yaml
# Workflow level
env:
  NODE_VERSION: '18'
  API_URL: https://api.example.com

jobs:
  test:
    # Job level
    env:
      TEST_ENV: test
    
    steps:
      - name: Test
        # Step level
        env:
          DEBUG: true
        run: npm test
```

### GITHUB_TOKEN

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

---

<a name="section-8"></a>
## 8. Conditions และ Control Flow

### Status Check Functions

```yaml
steps:
  - name: Run on success
    if: success()
    run: echo "All previous steps passed"
  
  - name: Run on failure
    if: failure()
    run: echo "A step failed"
  
  - name: Always run (cleanup)
    if: always()
    run: rm -rf temp/
  
  - name: Run if not cancelled
    if: "!cancelled()"
    run: echo "Workflow not cancelled"
```

### Context Variables

```yaml
jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging
  
  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production
```

### Logical Operators

```yaml
# AND
if: success() && github.ref == 'refs/heads/main'

# OR
if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'

# NOT
if: "!cancelled()"

# Complex
if: |
  (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop') &&
  success() &&
  !cancelled()
```

### Continue on Error

```yaml
steps:
  - name: Optional lint
    continue-on-error: true
    run: npm run lint
  
  - name: Always runs (even if lint fails)
    run: npm test
```

### Timeout

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - name: Long test
        timeout-minutes: 15
        run: npm run test:integration
```

---

<a name="section-9"></a>
## 9. Caching และ Optimization

### Built-in Caching

```yaml
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: '18'
      cache: 'npm'  # ← Auto cache ~/.npm
  
  - run: npm ci
  - run: npm test
```

**รองรับ:** npm, yarn, pnpm, pip, gradle, maven

### Manual Caching

```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-npm-
  
  - run: npm ci
```

### Docker Layer Caching

```yaml
- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Concurrency Control

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

ยกเลิก workflow runs ที่ยังไม่เสร็จเมื่อมี push ใหม่

### Path Filters

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package.json'
```

รันเฉพาะเมื่อไฟล์ที่ระบุมีการเปลี่ยนแปลง

---

<a name="section-10"></a>
## 10. Advanced Topics

### Reusable Workflows

**ไฟล์: `.github/workflows/reusable-deploy.yml`**
```yaml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy-token:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh ${{ inputs.environment }}
        env:
          TOKEN: ${{ secrets.deploy-token }}
```

**การเรียกใช้:**
```yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
    secrets:
      deploy-token: ${{ secrets.STAGING_TOKEN }}
```

### Composite Actions

**ไฟล์: `.github/actions/setup/action.yml`**
```yaml
name: Setup Project
description: Setup Node.js and install dependencies

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
    
    - run: npm ci
      shell: bash
```

**การใช้งาน:**
```yaml
steps:
  - uses: ./.github/actions/setup
  - run: npm test
```

### Deployment Environments

**สร้าง environment:**
```
Repository → Settings → Environments → New environment
```

**การใช้งาน:**
```yaml
jobs:
  deploy-production:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}  # production secret
```

**Features:**
- ✅ Required reviewers (manual approval)
- ✅ Wait timer
- ✅ Deployment branches
- ✅ Environment-specific secrets

---

<a name="section-11"></a>
## 11. Best Practices

### Security Best Practices

```yaml
# ✅ Use least privilege permissions
permissions:
  contents: read
  packages: write

# ✅ Pin action versions
- uses: actions/checkout@v4  # not @main

# ✅ Never hardcode secrets
env:
  API_KEY: ${{ secrets.API_KEY }}  # not: abc123

# ✅ Validate inputs
- name: Validate input
  if: inputs.environment == 'production' || inputs.environment == 'staging'
  run: echo "Valid environment"
```

### Performance Optimization

```yaml
# ✅ Use caching
- uses: actions/setup-node@v4
  with:
    cache: 'npm'

# ✅ Path filters
on:
  push:
    paths:
      - 'src/**'

# ✅ Concurrency control
concurrency:
  group: ${{ github.ref }}
  cancel-in-progress: true

# ✅ Job dependencies
jobs:
  build:
    runs-on: ubuntu-latest
  
  test:
    needs: build  # รันหลัง build เสร็จ
```

### Error Handling

```yaml
steps:
  # ✅ Continue on error for optional steps
  - name: Optional step
    continue-on-error: true
    run: npm run lint
  
  # ✅ Always cleanup
  - name: Cleanup
    if: always()
    run: rm -rf temp/
  
  # ✅ Set timeout
  - name: Long test
    timeout-minutes: 30
    run: npm run test:e2e
```

### Code Organization

```yaml
# ✅ Clear naming
name: CI/CD Pipeline

jobs:
  lint:
    name: Run Linter
  
  test:
    name: Run Unit Tests
  
  build:
    name: Build Application

# ✅ Separate workflows by purpose
# .github/workflows/
#   ├── ci.yml           # Tests
#   ├── deploy-prod.yml  # Production deployment
#   └── deploy-stg.yml   # Staging deployment
```

---

<a name="section-12"></a>
## 12. ตัวอย่างการใช้งานจริง

### Example 1: Complete Node.js CI/CD

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'

permissions:
  contents: read
  packages: write

jobs:
  # ==========================
  # JOB 1: QUALITY CHECKS
  # ==========================
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Format check
        run: npm run format:check
  
  # ==========================
  # JOB 2: TESTS
  # ==========================
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: password
        options: --health-cmd pg_isready
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run tests
        run: npm test
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: always()
  
  # ==========================
  # JOB 3: BUILD
  # ==========================
  build:
    name: Build Application
    needs: [quality, test]
    runs-on: ubuntu-latest
    
    outputs:
      version: ${{ steps.version.outputs.value }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      
      - name: Get version
        id: version
        run: echo "value=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
  
  # ==========================
  # JOB 4: DOCKER BUILD
  # ==========================
  docker:
    name: Build Docker Image
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ needs.build.outputs.version }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  # ==========================
  # JOB 5: DEPLOY
  # ==========================
  deploy-staging:
    name: Deploy to Staging
    needs: docker
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - run: ./deploy.sh staging
        env:
          API_KEY: ${{ secrets.API_KEY }}
  
  deploy-production:
    name: Deploy to Production
    needs: docker
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - run: ./deploy.sh production
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

---

### Example 2: Python Application with Testing

```yaml
name: Python CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    name: Test Python ${{ matrix.python-version }}
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11']
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        run: pytest --cov=app --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

### Example 3: Docker Multi-platform Build

```yaml
name: Docker Build

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/setup-qemu-action@v3
      
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            myapp:latest
            myapp:${{ github.ref_name }}
```

---

### Example 4: Monorepo with Path Filters

```yaml
name: Monorepo CI

on:
  push:
    branches: [main]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            frontend:
              - 'packages/frontend/**'
            backend:
              - 'packages/backend/**'
  
  frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        working-directory: packages/frontend
  
  backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        working-directory: packages/backend
```

---

### Example 5: Scheduled Cleanup

```yaml
name: Cleanup Old Artifacts

on:
  schedule:
    - cron: '0 2 * * 0'  # Every Sunday at 2 AM
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    
    permissions:
      actions: write
    
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const days = 30;
            const date = new Date();
            date.setDate(date.getDate() - days);
            
            const artifacts = await github.rest.actions.listArtifactsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
            });
            
            for (const artifact of artifacts.data.artifacts) {
              if (new Date(artifact.created_at) < date) {
                await github.rest.actions.deleteArtifact({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  artifact_id: artifact.id,
                });
                console.log(`Deleted ${artifact.name}`);
              }
            }
```

---

## 📋 Checklist สำหรับ Lab CI/CD

### ก่อนเริ่มทำ Lab

- [ ] มี GitHub Account แล้ว
- [ ] ติดตั้ง Git แล้ว
- [ ] ติดตั้ง Docker Desktop แล้ว
- [ ] ติดตั้ง Python/Node.js ตาม Lab
- [ ] เตรียม Code Editor (VS Code)

### การสร้าง Workflow

- [ ] สร้างโฟลเดอร์ `.github/workflows/`
- [ ] สร้างไฟล์ workflow (`.yml` หรือ `.yaml`)
- [ ] กำหนด `name:` ที่ชัดเจน
- [ ] เลือก triggers (`on:`) ที่เหมาะสม
- [ ] ตั้งค่า permissions ถ้าจำเป็น

### การเขียน Jobs

- [ ] ตั้งชื่อ jobs ที่บอกได้ว่าทำอะไร
- [ ] เลือก `runs-on:` ที่เหมาะสม
- [ ] กำหนด dependencies (`needs:`) ถ้าจำเป็น
- [ ] ตั้ง `timeout-minutes:` สำหรับ jobs ที่อาจนานเกินไป

### การเขียน Steps

- [ ] ใส่ `name:` ทุก step
- [ ] ใช้ actions จาก marketplace (เช่น `actions/checkout@v4`)
- [ ] ใช้ version tags แทน branches
- [ ] เพิ่ม error handling (`continue-on-error`, `if:`)

### Testing & Quality

- [ ] มี unit tests
- [ ] มี linting/formatting checks
- [ ] วัด code coverage
- [ ] ทดสอบใน service containers ถ้าจำเป็น

### Secrets & Security

- [ ] ไม่ hardcode secrets ในโค้ด
- [ ] สร้าง secrets ใน GitHub Settings
- [ ] ใช้ `${{ secrets.NAME }}` เท่านั้น
- [ ] ตั้ง permissions ที่จำเป็นเท่านั้น

### Deployment

- [ ] ใช้ environment protection
- [ ] ตั้ง required reviewers สำหรับ production
- [ ] มี health check หลัง deploy
- [ ] บันทึก deployment URL

### Optimization

- [ ] ใช้ caching สำหรับ dependencies
- [ ] ใช้ path filters ถ้าเหมาะสม
- [ ] ตั้ง concurrency control
- [ ] ใช้ matrix สำหรับ multi-version testing

---

## 🎯 คำถามสำคัญสำหรับทำข้อสอบ

### พื้นฐาน GitHub Actions

1. **GitHub Actions คืออะไร และมีข้อดีอย่างไร?**
   - Platform สำหรับ CI/CD ที่ built-in กับ GitHub
   - ข้อดี: ฟรีสำหรับ public repos, ใช้งานง่าย, รองรับหลาย platform

2. **Workflow ต้องเก็บไว้ที่ไหน?**
   - ใน `.github/workflows/` เท่านั้น
   - ชื่อไฟล์ลงท้ายด้วย `.yml` หรือ `.yaml`

3. **Jobs, Steps, Actions แตกต่างกันอย่างไร?**
   - **Job**: ชุดของ steps ที่รันบน runner เดียวกัน
   - **Step**: คำสั่งเดี่ยวที่รันเรียงลำดับ
   - **Action**: Reusable unit ที่เรียกใช้ด้วย `uses:`

### Triggers

4. **Event ที่ trigger workflow ได้มีอะไรบ้าง?**
   - `push`, `pull_request`, `schedule`, `workflow_dispatch`, `release`

5. **Cron schedule ใช้งานอย่างไร?**
   - Format: `* * * * *` (minute hour day month weekday)
   - เช่น `0 2 * * *` = ทุกวัน 2:00 AM UTC

6. **workflow_dispatch ใช้ทำอะไร?**
   - Manual trigger workflow ผ่าน GitHub UI
   - สามารถรับ inputs จากผู้ใช้ได้

### Jobs & Dependencies

7. **Jobs รันแบบ parallel หรือ sequential?**
   - Default: parallel (พร้อมกัน)
   - ใช้ `needs:` เพื่อทำให้รันเรียงลำดับ

8. **Runner คืออะไร? มีประเภทไหนบ้าง?**
   - Server ที่รัน workflow
   - GitHub-hosted (ubuntu, windows, macos) และ self-hosted

9. **Service containers ใช้ทำอะไร?**
   - รัน services เช่น database สำหรับ testing
   - เช่น PostgreSQL, Redis, MongoDB

### Secrets & Environment

10. **ความแตกต่างระหว่าง Secrets และ Environment Variables?**
    - Secrets: เข้ารหัส, ไม่แสดงใน logs
    - Env vars: plain text, แสดงได้

11. **GITHUB_TOKEN คืออะไร?**
    - Token ที่ GitHub สร้างให้อัตโนมัติแต่ละ workflow run
    - ใช้ authenticate กับ GitHub API

12. **วิธีใช้ secrets ที่ถูกต้อง?**
    - สร้างใน Repository Settings
    - เรียกใช้ด้วย `${{ secrets.NAME }}`
    - ห้าม hardcode ในโค้ด

### Matrix Strategy

13. **Matrix strategy ใช้ทำอะไร?**
    - รัน job หลาย configurations พร้อมกัน
    - เช่น test หลาย versions ของ Node.js

14. **include และ exclude ใช้ทำอะไร?**
    - `include`: เพิ่ม configuration พิเศษ
    - `exclude`: ลบ combination ที่ไม่ต้องการ

### Caching & Optimization

15. **Caching ช่วยอะไร?**
    - ลดเวลา download dependencies
    - ประหยัด bandwidth และ minutes

16. **Built-in caching vs Manual caching?**
    - Built-in: ใช้ `cache:` ใน setup actions (ง่ายกว่า)
    - Manual: ใช้ `actions/cache@v4` (ยืดหยุ่นกว่า)

17. **Concurrency control คืออะไร?**
    - ควบคุมจำนวน workflow runs พร้อมกัน
    - ยกเลิก runs เก่าเมื่อมี push ใหม่

### Best Practices

18. **Best practices สำหรับ security?**
    - ใช้ least privilege permissions
    - Pin action versions
    - ไม่ hardcode secrets
    - Validate inputs

19. **วิธี optimize workflow performance?**
    - ใช้ caching
    - Path filters
    - Concurrency control
    - Parallel jobs

20. **Error handling ที่ดีควรทำอย่างไร?**
    - ใช้ `continue-on-error:` สำหรับ optional steps
    - ใช้ `if: always()` สำหรับ cleanup
    - ตั้ง `timeout-minutes:`

---

## 💡 Tips สำหรับทำ Lab

### การ Debug Workflow

```yaml
# เพิ่ม debug logging
- name: Debug
  run: |
    echo "Event: ${{ github.event_name }}"
    echo "Ref: ${{ github.ref }}"
    echo "Actor: ${{ github.actor }}"
    env
```

### การทดสอบ Workflow

1. **ใช้ workflow_dispatch** สำหรับ manual testing
2. **ทดสอบใน branch แยก** ก่อน merge ไป main
3. **ใช้ draft PR** เพื่อ test โดยไม่ trigger workflows อื่น

### การแก้ไข Syntax Errors

1. ใช้ VS Code extension: "GitHub Actions"
2. ตรวจสอบ indentation (ใช้ spaces, ไม่ใช้ tabs)
3. ใช้ YAML validator online

### Resources ที่มีประโยชน์

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Actions Runner Images](https://github.com/actions/runner-images)

---

## 🎓 สรุปท้ายสุด

### สิ่งที่ต้องเข้าใจ

1. **Workflow Structure**: name, on, jobs, steps
2. **Triggers**: push, PR, schedule, manual
3. **Jobs**: parallel/sequential, dependencies, outputs
4. **Steps**: run commands, use actions
5. **Matrix**: multi-version/platform testing
6. **Secrets**: secure credential management
7. **Caching**: optimize performance
8. **Conditions**: control workflow logic
9. **Best Practices**: security, performance, organization

### เป้าหมายการเรียนรู้

- ✅ เข้าใจ CI/CD concepts
- ✅ สร้าง workflow พื้นฐานได้
- ✅ ใช้ actions จาก marketplace
- ✅ จัดการ secrets อย่างปลอดภัย
- ✅ Optimize workflow performance
- ✅ Deploy applications อัตโนมัติ
- ✅ Debug และแก้ไขปัญหาได้

### ขั้นตอนต่อไป

1. ✅ ทำ lab ตามไฟล์ที่กำหนด
2. ✅ ทดลองสร้าง workflow เอง
3. ✅ ศึกษา examples จากโปรเจกต์จริง
4. ✅ ปรับใช้กับโปรเจกต์ของตัวเอง

**Good luck กับการทำข้อสอบและ Lab! 🚀**