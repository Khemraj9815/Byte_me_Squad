# 🐳 Docker Image Build - Complete Explanation

## Quick Answer

**Q: Where is Docker image built?**
**A: In GitHub Actions Runner (GitHub's cloud servers) - NOT on your computer**

---

## 📍 Visual: The Complete Process

```
YOUR COMPUTER
└─ git push origin main
   ↓
   ↓ (Code goes to GitHub)
   ↓
GITHUB SERVERS (GitHub Actions Runner = Ubuntu Linux VM)
┌────────────────────────────────────────────────┐
│ Workflow: Build, Test, and Deploy              │
├────────────────────────────────────────────────┤
│                                                │
│ JOB 1: Lint & Type Check (2-3 min)            │
│ ├─ Runs: npm run lint                         │
│ ├─ Runs: npx tsc --noEmit                     │
│ └─ ✅ Done                                     │
│                                                │
│ JOB 2: Build Application (3-4 min)            │
│ ├─ Runs: npm ci (install deps)                │
│ ├─ Runs: npm run build (creates dist/)        │
│ ├─ Uploads dist/ as artifact                  │
│ └─ ✅ Done                                     │
│                                                │
│ JOB 3: Docker Build & Security Scan (5-10 min)│
│ ├─ SET UP DOCKER                              │
│ │  └─ Installs docker/buildx-action           │
│ │                                              │
│ ├─ 🐳 BUILD DOCKER IMAGE ⬅️ HERE!             │
│ │  ├─ Reads: Dockerfile (in repo root)        │
│ │  ├─ Executes: docker build -t image:tag .  │
│ │  │            (builds image from scratch)   │
│ │  ├─ Steps in Dockerfile:                    │
│ │  │  1. FROM node:20-alpine (base image)     │
│ │  │  2. WORKDIR /app (create /app folder)    │
│ │  │  3. COPY package*.json ./ (copy files)   │
│ │  │  4. RUN npm ci (install npm packages)    │
│ │  │  5. COPY src ./src (copy source code)    │
│ │  │  6. RUN npm run build (build React app)  │
│ │  │  7. FROM node:20-alpine (fresh stage)    │
│ │  │  8. COPY dist from builder (copy built app)  │
│ │  │  9. CMD serve -s dist -l 3000 (run app)  │
│ │  └─ Creates: Docker image (~300MB)          │
│ │  └─ Saves to: /tmp/image.tar                │
│ │                                              │
│ ├─ 🔐 SCAN WITH TRIVY ⬅️ SECURITY!            │
│ │  ├─ Load image: docker load /tmp/image.tar  │
│ │  ├─ Runs: trivy scan [image]                │
│ │  ├─ Checks for CVEs, vulnerable packages   │
│ │  ├─ Generates: SARIF report                 │
│ │  └─ Uploads: GitHub Security tab            │
│ │                                              │
│ └─ ✅ Done (even if vulns found)               │
│                                                │
│ JOB 4: Deploy to Production (2-5 min)         │
│ ├─ Download dist/ artifact                    │
│ ├─ Call Render API:                           │
│ │  curl https://api.render.com/deploy/...    │
│ ├─ Tells Render: "Deploy latest code"        │
│ └─ ✅ Done (Render takes it from here)         │
│                                                │
└────────────────────────────────────────────────┘
   ↓
   ↓ (Render gets deployment request)
   ↓
RENDER SERVERS
┌────────────────────────────────────────────────┐
│ 1. Receives deployment webhook from GitHub     │
│ 2. Pulls your code from GitHub repo            │
│ 3. Reads Dockerfile (from repo)                │
│ 4. 🐳 BUILDS DOCKER IMAGE AGAIN               │
│    (This is the PRODUCTION image!)             │
│ 5. Runs: docker run -p 3000:3000 [image]      │
│ 6. App is now LIVE on internet!               │
│    https://druknest.onrender.com               │
└────────────────────────────────────────────────┘
```

---

## 🔧 Your Dockerfile (The Blueprint)

**Location:** `/home/khemrajghalley/Documents/YEAR3_II/DSO101/Byte_me_Squad/Dockerfile`

**What it does:**
1. **Reads the Dockerfile** - Instructions for building image
2. **Executes each line** - Step by step
3. **Creates a Docker image** - A package containing:
   - Node.js runtime
   - npm packages
   - Your React app code
   - Serve utility to run the app
4. **Image size** - ~300MB (compressed)

**How it's used:**
- GitHub Actions: Builds image for scanning
- Render: Builds image for production deployment

---

## 📝 Step-by-Step: What Happens When GitHub Builds Docker

### Step 1: Check out code
```bash
# GitHub Actions downloads your repo
# All files are now available in /github/workspace/
```

### Step 2: Set up Docker
```bash
# GitHub Actions installs Docker buildx (advanced build tool)
# Prepares to build Docker images
```

### Step 3: Build Docker image (THIS IS THE KEY PART!)
```bash
# GitHub runs (internally):
docker build \
  --build-arg VITE_SUPABASE_URL="${{ secrets.VITE_SUPABASE_URL }}" \
  --build-arg VITE_SUPABASE_ANON_KEY="${{ secrets.VITE_SUPABASE_ANON_KEY }}" \
  -t ghcr.io/[user]/byte_me_squad:[git-sha] \
  .

# This reads Dockerfile and builds layer by layer:
```

### Layer-by-layer build process:

```
LAYER 1: FROM node:20-alpine AS builder
         └─ Downloads Node.js base image (~150MB)

LAYER 2: WORKDIR /app
         └─ Creates /app directory in image

LAYER 3: COPY package*.json ./
         └─ Copies package.json to /app/

LAYER 4: RUN npm ci
         └─ Installs dependencies (takes 1-2 min)

LAYER 5: COPY tsconfig*.json ./
         └─ Copies TypeScript config

LAYER 6: COPY src ./src & COPY public ./public
         └─ Copies your source code

LAYER 7: RUN npm run build
         └─ Builds React app with Vite
         └─ Creates dist/ folder with optimized code

LAYER 8: FROM node:20-alpine (NEW STAGE - clean start)
         └─ Starts with fresh Node.js image
         └─ Throws away build tools to save space

LAYER 9: COPY --from=builder /app/dist ./dist
         └─ Copies only the dist/ folder from builder
         └─ (Reduces image size by ~90%)

LAYER 10: COPY --from=builder /app/node_modules ./node_modules
          └─ Copies only needed dependencies

LAYER 11: EXPOSE 3000
          └─ Documents that app uses port 3000

LAYER 12: HEALTHCHECK ...
          └─ Tells Docker how to check if app is healthy

LAYER 13: CMD ["serve", "-s", "dist", "-l", "3000"]
          └─ Default command to run when container starts
          └─ Starts "serve" utility on port 3000
```

### Step 4: Save image
```bash
# After all layers complete:
# Docker creates image (~300MB)
# Saves to: /tmp/image.tar

# This is the Docker image!
# Can be run on any machine with Docker installed
```

---

## 🔐 Trivy Security Scan (Part of Docker Build Job)

**After image is built, Trivy scans it:**

```
1. Load the image
   docker load --input /tmp/image.tar

2. Scan the image
   trivy image /tmp/image.tar
   
3. Checks for vulnerabilities in:
   ├─ Node.js version
   ├─ npm packages (from package.json)
   ├─ Alpine Linux base image
   └─ All installed dependencies

4. Generates report:
   [CRITICAL] 0 vulnerabilities
   [HIGH] 2 vulnerabilities  
   [MEDIUM] 5 vulnerabilities
   
5. Uploads results to GitHub
   └─ Visible in Security → Code scanning tab
```

---

## 🚀 What Happens AFTER GitHub Builds Docker

### GitHub Actions Sends Deploy Signal
```bash
# Job 4 runs:
curl https://api.render.com/deploy/srv-[SERVICE_ID]?key=[API_KEY]
# This tells Render: "Hey, deploy latest code!"
```

### Render Receives Request
```
Render gets: "Deploy latest code from GitHub main branch"

Render does:
1. Clones your GitHub repo
2. Reads Dockerfile
3. Builds Docker image (FRESH - not from GitHub)
   └─ Different image than GitHub built
   └─ This is the one that runs in production!
4. Runs container: docker run -p 3000:3000 [image]
5. App is now live on the internet
```

---

## 🎯 Key Differences: GitHub vs Render Docker Builds

| Aspect | GitHub Actions | Render |
|--------|---|---|
| **Purpose** | Test build & security scan | Production deployment |
| **Image used** | For Trivy scanning only | Runs in production |
| **Network** | GitHub's servers | Render's servers |
| **Preserved** | No (ephemeral) | Yes (runs live) |
| **When** | Every push to main | After GitHub completes |
| **Duration** | 1-2 min build + 3-5 min scan | 2-5 min build & start |

---

## 📊 Complete Timeline

```
TIME     LOCATION              WHAT HAPPENS
────────────────────────────────────────────────────────
0:00     Your computer         git push origin main
         ↓
0:30     GitHub Actions        Lint & type check (2-3 min)
         ↓
3:00     GitHub Actions        Build dist/ folder (3-4 min)
         ↓
6:00     GitHub Actions        Docker builds image ← HERE! (1-2 min)
                               Reads Dockerfile
                               Executes npm ci, npm run build
                               Creates image in /tmp/image.tar
         ↓
7:00     GitHub Actions        Trivy scans image (3-5 min)
                               Checks vulnerabilities
                               Uploads SARIF report
         ↓
10:00    GitHub Actions        Deploy job sends signal (< 1 min)
         ↓
10:05    Render                Receives deployment request
         ↓
10:10    Render                Clones repo & builds Docker image again
         ↓
12:00    Render                Starts container (app running)
         ↓
15:00    Internet              App is LIVE! 🎉
         └─ https://druknest.onrender.com
```

---

## 🔍 How to See Docker Build Happening

### Method 1: Watch GitHub Actions Live
1. Go to: https://github.com/[YOUR-USER]/Byte_me_Squad/actions
2. Click the workflow run
3. Click "Docker Build & Security Scan" job
4. Expand "Build Docker image" step
5. Watch the logs in real-time:
   ```
   #1 [internal] load build definition from Dockerfile
   #1 copying filesystem...
   #1 sha256:abc123...
   
   #2 [builder 1/8] FROM node:20-alpine
   #2 resolving image config for docker.io/library/node:20-alpine
   #2 sha256:def456...
   
   #3 [builder 2/8] WORKDIR /app
   #3 CACHED
   
   #4 [builder 3/8] COPY package*.json ./
   #4 CACHED
   
   #5 [builder 4/8] RUN npm ci
   #5 up 12.34s
   ... (installing packages)
   
   #6 [builder 5/8] COPY src ./src
   #6 CACHED
   
   #7 [builder 6/8] RUN npm run build
   #7 up 34.56s
   ... (building React app)
   
   #8 [builder 7/8] COPY public ./public
   #8 CACHED
   
   #9 [stage-1 1/6] FROM node:20-alpine
   #9 sha256:ghi789...
   
   #10 [stage-1 2/6] COPY --from=builder /app/dist ./dist
   #10 CACHED
   
   #11 [stage-1 3/6] COPY --from=builder /app/node_modules ./node_modules
   #11 up 2.34s
   
   #12 [stage-1 4/6] EXPOSE 3000
   #12 CACHED
   
   #13 [stage-1 5/6] RUN npm install -g serve
   #13 CACHED
   
   #14 [stage-1 6/6] CMD ["serve", "-s", "dist", "-l", "3000"]
   #14 CACHED
   
   #14 writing image sha256:jkl012...
   #14 naming to ghcr.io/[user]/byte_me_squad:[sha]
   ```

### Method 2: Check Trivy Scan Results
1. In same "Docker Build & Security Scan" job
2. Expand "Run Trivy vulnerability scanner" step
3. See scan output:
   ```
   2024-05-16T12:34:56Z INFO Vulnerability DB updated
   2024-05-16T12:34:57Z INFO Scanning image...
   
   Alpine Linux v3.18.0
   ├─ CVE-2024-1234 [HIGH] python3
   │  └─ https://nvd.nist.gov/vuln/detail/CVE-2024-1234
   │
   ├─ CVE-2024-5678 [HIGH] openssl
   │  └─ https://nvd.nist.gov/vuln/detail/CVE-2024-5678
   │
   └─ CVE-2024-9012 [MEDIUM] curl
      └─ https://nvd.nist.gov/vuln/detail/CVE-2024-9012
   
   Total: 3 vulnerabilities
   ```

---

## 💾 Your Dockerfile Location

```
/home/khemrajghalley/Documents/YEAR3_II/DSO101/Byte_me_Squad/Dockerfile
```

**Check it:**
```bash
cat /home/khemrajghalley/Documents/YEAR3_II/DSO101/Byte_me_Squad/Dockerfile
```

**It contains:**
- Multi-stage build (Builder stage + Production stage)
- Node.js alpine base image (lightweight)
- npm install & build commands
- Serve utility to run the app
- Health checks
- Port 3000 exposure

---

## 🎓 Learning Path

1. **Understand Docker basics** ← You are here
2. Watch GitHub Actions build Docker image in real-time
3. Check Trivy scan results
4. Check Render deployment
5. Visit live app at https://druknest.onrender.com
6. Celebrate! 🎉

---

## ❓ FAQ

**Q: Does Docker image get deployed from GitHub?**
A: No. GitHub only builds and scans it. Render builds a fresh image.

**Q: Why build twice (GitHub + Render)?**
A: GitHub builds for security scanning. Render builds for actual deployment.

**Q: Can I build Docker locally?**
A: Yes! `docker build -t druknest . && docker run -p 3000:3000 druknest`

**Q: What if Trivy finds vulnerabilities?**
A: Workflow continues. You can fix later by updating dependencies.

**Q: How big is the Docker image?**
A: ~300MB (acceptable size for Node.js + React app)

**Q: Can I see the image size?**
A: Yes, in GitHub Actions logs: `#14 writing image sha256:xxx (size: 298MB)`

---

## ✅ Next Steps

1. Check GitHub Actions: https://github.com/[USER]/Byte_me_Squad/actions
2. Click your workflow run
3. Click "Docker Build & Security Scan" job
4. Watch the build logs
5. Check Trivy scan results
6. Monitor Render deployment
7. Visit your live app!

**Your Docker image is being built RIGHT NOW in GitHub Actions!** 🐳
