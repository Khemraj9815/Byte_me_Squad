# Docker Image Build Location & Process

## 🏗️ WHERE is Docker Image Built?

**Answer: In GitHub Actions Runner (Ubuntu Server in the Cloud)**

Not on your local machine, not on Render. It happens in GitHub's servers!

---

## 📍 The Build Location Breakdown:

```
┌─────────────────────────────────────────────────────────────────┐
│                      YOUR COMPUTER                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ git push origin main                                    │    │
│  │ (Sends code to GitHub)                                  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                                ⬇️
┌─────────────────────────────────────────────────────────────────┐
│                    GITHUB SERVERS                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ GitHub Actions Runners (Ubuntu Linux VMs)              │    │
│  │                                                         │    │
│  │ JOB 1: Lint & Type Check (ubuntu-latest)              │    │
│  │ ✅ 2-3 minutes                                          │    │
│  │                                                         │    │
│  │ JOB 2: Build Application (ubuntu-latest)              │    │
│  │ ✅ 3-4 minutes                                          │    │
│  │ Creates: dist/ folder                                 │    │
│  │                                                         │    │
│  │ JOB 3: Docker Build & Trivy Scan (ubuntu-latest) ⬅️   │    │
│  │ ✅ 5-10 minutes                                         │    │
│  │ 🐳 DOCKER IMAGE BUILT HERE!                           │    │
│  │ 🔐 Trivy scans it                                      │    │
│  │                                                         │    │
│  │ JOB 4: Deploy to Production (ubuntu-latest)          │    │
│  │ ✅ 2-5 minutes                                          │    │
│  │ Tells Render: "Pull latest code and deploy"          │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                                ⬇️
┌─────────────────────────────────────────────────────────────────┐
│                    RENDER SERVERS                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Receives deployment request from GitHub Actions        │    │
│  │                                                         │    │
│  │ 1. Pulls your code from GitHub                         │    │
│  │ 2. Builds Docker image again (using your Dockerfile)  │    │
│  │ 3. Runs the container on port 3000                    │    │
│  │ 4. Makes app live at: https://druknest.onrender.com   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🐳 Docker Image Build Details

### Job 3: "Docker Build & Security Scan" - Step by Step

This happens in GitHub Actions, NOT on your computer:

```yaml
# Step 1: Check out your code
- name: Checkout code
  # Downloads your repo to GitHub's Ubuntu server

# Step 2: Set up Docker buildx (special Docker tool)
- name: Set up Docker Buildx
  # Installs Docker build tools on GitHub's server

# Step 3: BUILD THE DOCKER IMAGE ⬅️ THIS IS WHERE IT HAPPENS
- name: Build Docker image
  uses: docker/build-push-action@v5
  # Runs: docker build -f Dockerfile -t image:tag .
  # 
  # This:
  # 1. Reads your Dockerfile
  # 2. Executes each line (FROM, COPY, RUN, CMD)
  # 3. Creates a Docker image (~300MB)
  # 4. Saves it to /tmp/image.tar
  #
  # Takes: ~1-2 minutes

# Step 4: Load the image for scanning
- name: Load image for scanning
  # Loads the image so Trivy can scan it

# Step 5: Scan with Trivy for vulnerabilities
- name: Run Trivy vulnerability scanner
  # Trivy scans the image
  # Checks for CVEs, vulnerable packages, etc.
  # Takes: 3-5 minutes

# Step 6: Upload scan results to GitHub
- name: Upload Trivy results to GitHub Security tab
  # Results appear in your GitHub repo's Security tab
```

---

## 📝 The Dockerfile (What Gets Built)

Your `Dockerfile` is in the repo root:

```
/home/khemrajghalley/Documents/YEAR3_II/DSO101/Byte_me_Squad/Dockerfile
```

When GitHub Actions runs "Build Docker image", it:
1. Reads this Dockerfile
2. Executes each instruction:
   ```dockerfile
   FROM node:20-alpine          # Start with Node.js base image
   WORKDIR /app                 # Set working directory
   COPY package*.json ./        # Copy package.json
   RUN npm ci                   # Install dependencies
   COPY src ./src               # Copy your source code
   RUN npm run build            # Build the app (creates dist/)
   FROM node:20-alpine          # Start fresh (multi-stage)
   COPY --from=builder /app/dist ./dist  # Copy dist folder
   CMD ["serve", "-s", "dist", "-l", "3000"]  # Run the app
   ```
3. Creates a Docker image (~300MB)
4. Saves it as `/tmp/image.tar`

---

## 🔐 Trivy Scan (Part of Job 3)

After Docker image is built:

```
GitHub Actions Runner
├─ Docker image built (/tmp/image.tar)
│
├─ Trivy scans the image
│  ├─ Checks Node.js version for CVEs
│  ├─ Checks npm packages for vulnerabilities
│  ├─ Checks Alpine Linux base image
│  └─ Creates report (SARIF format)
│
├─ Report uploaded to GitHub Security tab
│  └─ You see: "CRITICAL: 0, HIGH: 2, MEDIUM: 5"
│
└─ Workflow continues (even if vulns found)
```

---

## 🚀 What Happens After Docker Build

After Job 3 completes, Job 4 runs:

```
Job 4: Deploy to Production
├─ Downloads dist/ folder (from Job 2)
├─ Sends API request to Render:
│  "Hey Render, deploy the latest code"
│
└─ Render does:
   ├─ Pulls your code from GitHub
   ├─ Reads your Dockerfile
   ├─ Builds Docker image (AGAIN, on Render servers)
   ├─ Runs: docker run -p 3000:3000 [image]
   └─ App is LIVE at: https://druknest.onrender.com
```

---

## 💡 Important Notes

### Why is Docker built TWICE?

1. **GitHub Actions (Job 3):**
   - Builds image to scan with Trivy
   - For security validation
   - This image is NOT deployed

2. **Render:**
   - Receives deployment request
   - Pulls code from GitHub
   - Builds image AGAIN
   - Runs the container
   - This IS the live image

### Why not use the image from GitHub?

GitHub Actions builds are ephemeral (temporary). We use Render's built image because:
- ✅ Render manages the infrastructure
- ✅ Render handles auto-scaling
- ✅ Render handles domain/SSL
- ✅ Easier to manage and monitor

---

## 🔍 How to Monitor Docker Build in GitHub Actions

1. Go to: https://github.com/[USER]/Byte_me_Squad/actions

2. Click the workflow run

3. Click "Docker Build & Security Scan" job

4. Watch the logs:
   ```
   Run docker/setup-buildx-action@v3
   
   Run docker/build-push-action@v5
   #1 [internal] load build definition from Dockerfile
   #1 copying filesystem...
   #2 [builder 1/8] FROM node:20-alpine
   #3 [builder 2/8] WORKDIR /app
   #4 [builder 3/8] COPY package*.json ./
   #5 [builder 4/8] RUN npm ci
   ... (20-30 more lines)
   #10 [stage-1 6/6] RUN /bin/sh -c npm install -g serve
   #10 writing image sha256:abc123...
   #10 naming to ghcr.io/[user]/byte_me_squad:[sha]
   ```

5. After build, Trivy starts scanning:
   ```
   Run Trivy vulnerability scanner
   2024-05-16T12:34:56Z INFO Vulnerability DB updated
   2024-05-16T12:34:57Z INFO Scanning image...
   
   Alpine Linux v3.18
   ├─ CVE-2024-1234 [HIGH]
   ├─ CVE-2024-5678 [HIGH]
   └─ CVE-2024-9012 [MEDIUM]
   
   Total: 3 vulnerabilities
   ```

6. Results uploaded to GitHub Security tab

---

## 🎯 Summary

| Question | Answer |
|----------|--------|
| **Where is Docker built?** | GitHub Actions runner (Ubuntu VM in cloud) |
| **When?** | After "Build Application" job completes |
| **What reads the Dockerfile?** | `docker/build-push-action@v5` |
| **How long?** | ~1-2 minutes for build |
| **Is this the final image?** | No, it's for Trivy scanning only |
| **Where is the final image?** | Built on Render's servers |
| **Can I build it locally?** | Yes: `docker build -t druknest . && docker run -p 3000:3000 druknest` |

---

## 📊 Complete Timeline

```
Time  Event                           Location
────  ─────────────────────────────   ──────────────────
0:00  git push origin main            Your computer
      ↓
0:05  Lint & Type Check starts        GitHub Actions
2:00  Lint check completes            GitHub Actions
      ↓
2:05  Build Application starts        GitHub Actions
5:00  Build completes (dist/ created) GitHub Actions
      ↓
5:05  Docker Build & Scan starts      GitHub Actions
6:00  🐳 Docker image built           GitHub Actions ⬅️ HERE!
7:00  Trivy scan starts               GitHub Actions
10:00 Trivy scan completes            GitHub Actions
      ↓
10:05 Deploy to Production starts     GitHub Actions
10:10 Render receives request         Render servers
10:15 Render builds Docker image      Render servers
15:00 Container starts                Render servers
20:00 App is LIVE! 🎉                https://druknest.onrender.com
```

---

## ✅ Your Deployment is Currently:

Since you pushed code earlier, check the progress:

1. Go to: https://github.com/[YOUR-USERNAME]/Byte_me_Squad/actions
2. Look for the workflow you triggered
3. You should see it running or completed:
   - ✅ Lint check (done)
   - ✅ Build (done)
   - 🔄 Docker build & Trivy scan (in progress or done?)
   - ⏳ Deploy (waiting or in progress?)

Check the "Docker Build & Security Scan" job logs to see the Docker image being built in real-time!
