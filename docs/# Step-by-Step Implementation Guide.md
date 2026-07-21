# Phase 1 (AWS Version, Beginner-Friendly) — Provision the Cloud VM

**Replaces the original Proxmox-based Phase 1.** This version assumes you've never touched the AWS Console before, so it walks through account creation and every click, not just the commands. Phases 2–6 of your original guide are unchanged — this only covers how your VM gets created and how you reach it.

---

### Step 1.0: Create an AWS account (skip if you already have one) 🖥️ WINDOWS PC (browser)

1. Go to [aws.amazon.com](https://aws.amazon.com) → click **Create an AWS Account** (top right).
2. You'll need:
   - An email address (use one you check regularly — this becomes your account's "root user" login).
   - A **credit or debit card**. AWS requires this even if you end up using free-tier or student credits — think of it as identity verification plus a backstop for anything beyond free usage.
   - A phone number for SMS/call verification.
3. Choose the **Personal** account type (not Business) when asked.
4. You'll pick a **Support Plan** — choose **Basic support - Free**. Don't pay for a support plan for a course project.
5. Once verification finishes, you'll land in the **AWS Management Console** — this is the web dashboard for everything AWS. Bookmark it: `https://console.aws.amazon.com/`.

**A quick orientation, since you'll live in this console for the rest of the project:**
- Top-left has a search bar labeled **"Search"** — this is how you'll get to EC2, S3, etc. Faster than hunting through menus.
- Top-right shows your **Region** (e.g. "US East (N. Virginia)"). This matters — resources you create only exist in the region you're viewing. Pick one close to you and **stick with it for the whole project** (e.g. `us-east-2` or `us-east-1` if you're in Missouri — either has low latency and among the cheapest pricing). If EC2 instances "disappear" later, 9 times out of 10 it's because you switched regions in this dropdown.

---

### Step 1.1: Launch an EC2 instance 🖥️ WINDOWS PC (browser) → creates the ☁️ AWS VM

1. In the console's top search bar, type **EC2** and click the top result. This opens the EC2 Dashboard — the control panel for virtual machines ("instances," in AWS terminology).
2. Click the orange **Launch Instance** button.
3. **Name and tags** → **Name:** type something identifiable, e.g. `splunk-log-project`.
4. **Application and OS Images (Amazon Machine Image):**
   - This picker chooses your VM's operating system. Click **Ubuntu** in the Quick Start list.
   - From the dropdown below it, confirm it says **Ubuntu Server 22.04 LTS** and **64-bit (x86)**.
   - Leave everything else in this section at default.
5. **Instance type** — this is your VM's size (CPU/RAM). Click the dropdown and pick based on this table:

   | Type | vCPU | RAM | Notes |
   |---|---|---|---|
   | `t3.micro` | 2 | 1 GB | **Not enough** — Splunk alone typically wants 2GB+ just to start reliably. This is the free-tier-eligible one, but skip it here. |
   | `t3.medium` | 2 | 4 GB | Minimum comfortable size |
   | `t3.large` | 2 | 8 GB | **Recommended** — comfortable headroom once Splunk + forwarder + sample app + Jenkins are all running together |

   Type `t3.large` into the search box within that dropdown, or scroll to find it.

   ⚠️ **Cost note (important as a first-timer):** `t3.medium`/`t3.large` are **not** free — expect roughly $0.03–0.08/hr depending on region, which is cheap per hour but adds up if left running 24/7. You'll stop the instance between sessions (Step 1.7 below) to control this. AWS will not warn you before charging you — there's no "are you sure" popup for cost, so this is on you to manage.

6. **Key pair (login)** — this is how you'll prove it's really you when connecting later (like a password, but stronger — a cryptographic key file instead of typed text).
   - Click **Create new key pair**.
   - **Key pair name:** `splunk-project-key`.
   - **Key pair type:** RSA.
   - **Private key file format:** `.pem`.
   - Click **Create key pair** — a file named `splunk-project-key.pem` downloads automatically to your Downloads folder.
   - **Move this file somewhere permanent right now** — e.g. create a folder `C:\Users\David\.ssh\` and move it there. AWS will never let you download this again; if you lose it, you lose access to the instance (you'd have to make a new one).

7. **Network settings** — click the **Edit** button on the right side of this section (not the defaults) so you can configure the firewall rules while creating the instance:
   - **VPC / Subnet:** leave at default — AWS auto-creates a default network for new accounts, that's fine for this project.
   - **Auto-assign public IP:** make sure this is set to **Enable** (it usually is by default) — without this, your instance has no internet-reachable address at all.
   - **Firewall (security groups):** choose **Create security group**.
     - Name it something like `splunk-project-sg`.
     - You'll see one default rule already listed for SSH (port 22) with **Source type: Anywhere (0.0.0.0/0)**. Change the source type to **My IP** — the console will auto-fill your current public IP. This restricts SSH to just your current internet connection, safer than leaving it open to the whole internet.
     - Click **Add security group rule** two more times to add:

       | Type | Port range | Source type |
       |---|---|---|
       | Custom TCP | 8000 | My IP |
       | Custom TCP | 8080 | My IP |

       (Port 8000 = Splunk Web UI, port 8080 = Jenkins Web UI, added later in Phase 5. You don't need to open port 9997 — that's forwarder-to-indexer traffic that stays inside the VM, not the internet.)

8. **Configure storage:** default is usually 8GB — click the size field and change it to **40** (GiB). Leave the volume type as `gp3`. This matches your project's original 40GB spec and gives Splunk's index room to grow.
9. Everything else can stay default. Scroll down and click the orange **Launch Instance** button.
10. You'll see a green "Successfully initiated launch" screen — click **View all instances** to go back to the dashboard and watch it boot. **Instance State** will say `Pending`, then flip to `Running` after 30–60 seconds. **Status Check** will take another minute or two to show `2/2 checks passed` — wait for that before trying to connect.

---

### Step 1.2: Find your instance's public IP address 🖥️ WINDOWS PC (browser)

1. Back on the EC2 Dashboard → **Instances** (left sidebar).
2. Click your instance's checkbox (or its name) — a details panel opens below/beside the list.
3. Find and copy the **Public IPv4 address** (looks like `3.15.142.201`). You'll use this to SSH in and later to open Splunk in your browser.

---

### Step 1.3: Connect via SSH from your Windows PC ☁️ your first connection

Windows 10/11 includes a built-in SSH client, so you don't need to install anything extra — just use PowerShell.

1. Open **PowerShell** (search for it in the Start menu).
2. Navigate to wherever you saved the key file, or reference its full path directly. Run:

```powershell
ssh -i "C:\Users\David\.ssh\splunk-project-key.pem" ubuntu@<PUBLIC-IP>
```

   Replace `<PUBLIC-IP>` with the address from Step 1.2. **Note the username is `ubuntu`** — that's the default login for Ubuntu AMIs on AWS, not a course-assigned username like before.

3. **If you get a permissions error** like `Permissions for '...pem' are too open`, Windows needs to lock the file down to just you. Run these once:

```powershell
icacls "C:\Users\David\.ssh\splunk-project-key.pem" /inheritance:r
icacls "C:\Users\David\.ssh\splunk-project-key.pem" /grant:r "%username%:R"
```

   Then retry the `ssh` command from step 2.

4. **First connection only:** you'll see a prompt like:
```
The authenticity of host '3.15.142.201' can't be established.
Are you sure you want to continue connecting (yes/no)?
```
   This is normal and expected the very first time you connect to any new server — type `yes` and press Enter.

**Checkpoint:** you should land at a prompt like `ubuntu@ip-172-31-xx-xx:~$`. That's a real shell on your new VM — no Cloudflare, no proxy, no timeout.

---

### Step 1.4: Install Docker Engine + Docker Compose ☁️ AWS VM

You're now SSH'd into the VM. These commands are identical to your original guide:

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

**Checkpoint:** `docker run hello-world` should succeed (prints a friendly confirmation message).

---

### Step 1.5: (Recommended) Allocate an Elastic IP so your address doesn't change

Right now, if you stop and restart your instance later, AWS assigns you a **brand-new** public IP — breaking your SSH command, your security group's "My IP" rule if your home IP also changed, and any bookmarked Splunk URL. An Elastic IP fixes your address permanently:

1. EC2 Dashboard → left sidebar → **Network & Security → Elastic IPs**.
2. Click **Allocate Elastic IP address** → click **Allocate** (defaults are fine).
3. Select the new address in the list → **Actions → Associate Elastic IP address** → choose your instance → **Associate**.
4. Your instance's public IP is now fixed to this address. Use it going forward for SSH and for the Splunk Web UI.

⚠️ One quirk to know: AWS charges a small hourly fee for an Elastic IP **only if it's allocated but not attached to a running instance** — so as long as it stays associated with your (even stopped) instance, you're fine; just don't allocate a spare you don't use.

---

### Step 1.6: Clone your repo onto the VM ☁️ AWS VM

Still in your SSH session:

```bash
git clone <your-repo-url>
cd splunk-log-project
```

If this is your first time using git from this VM and it asks for GitHub credentials, you may need a [personal access token](https://github.com/settings/tokens) instead of your password (GitHub no longer accepts plain passwords for git operations) — let me know if you hit that and I'll walk you through it.

---

### Step 1.7: Stop the instance when you're done for the session 🖥️ WINDOWS PC (browser)

This is the step that actually controls your cost — get in the habit of doing this every time you're done working:

1. EC2 Dashboard → **Instances** → check the box next to your instance.
2. **Instance State → Stop instance** → confirm.
3. Wait for it to show `Stopped`. While stopped, you are **not** billed for compute time (you are billed a small amount for the storage volume sitting there, a few cents a month for 40GB — negligible).
4. Everything persists — Docker images, Splunk's indexed data, your cloned repo, all still there next time.
5. **To resume work:** select the instance → **Instance State → Start instance** → wait for `Running` + `2/2 status checks` → SSH back in the same way as before. (If you set up the Elastic IP in Step 1.5, your `ssh` command and IP stay identical every time. If you skipped that step, check Step 1.2 again for the new IP.)

---

## What changes downstream in the rest of your guide

Nothing structurally — just terminology:
- Every `<VM-IP>` reference in Phases 2–6 becomes your EC2 public IP (or Elastic IP).
- SSH username is `ubuntu`, not a course-assigned username.
- When you eventually open the Splunk Web UI, it'll be `http://<your-EC2-IP>:8000` directly from your Windows browser — no tunnel needed, since you opened port 8000 in your own security group in Step 1.1.
- In your final README/report, describe the architecture as "deployed on an AWS EC2 instance (Ubuntu 22.04, t3.large)" and note briefly *why* you moved off the course's Proxmox environment (the Cloudflare-proxied SSH connectivity issue) — worth documenting as a real infrastructure decision, not glossing over.

---

## Quick glossary, since some of this vocabulary is new

| Term | What it means here |
|---|---|
| **Instance** | AWS's word for a virtual machine |
| **AMI** | The OS image/template an instance is created from (you picked Ubuntu 22.04) |
| **Key pair** | A cryptographic key used instead of a password to SSH in |
| **Security group** | A virtual firewall attached to your instance — controls which ports/IPs can reach it |
| **Elastic IP** | A public IP address you "reserve" so it doesn't change when you stop/start the instance |
| **Region** | The physical AWS data center location your resources live in (e.g. `us-east-2`) |
| **Console** | The AWS web dashboard at console.aws.amazon.com |




# Step-by-Step Implementation Guide
### Centralized Log Management with Splunk — ITC583/683 Final Project
Based on your approved proposal: Splunk indexer + Universal Forwarder + sample app, containerized with Docker Compose, deployed on the Proxmox course cloud (dev.itccloud.top), automated with Jenkins CI/CD.

**Two machines are involved throughout — every step below is tagged:**
- 🖥️ **WINDOWS PC** — where you write code/config and run Claude Code
- ☁️ **UBUNTU VM** (dev.itccloud.top) — where Docker, Splunk, and Jenkins actually run

**The general workflow:** you author files on your Windows PC (with Claude Code doing the writing), push them to a git repo, then pull/deploy them on the Ubuntu VM over SSH. Jenkins on the VM eventually automates that pull-and-deploy step for you. You are never running Docker or Splunk *on* your Windows PC in this design — everything containerized lives on the VM.

Grading targets this guide is built around:
- Implementation & configuration — 12 pts
- Deployment & cloud usage — 8 pts
- Automation / containerization / CI/CD — 5 pts
- Documentation — 5 pts

---

## Phase 0 — Set Up Your Working Environment

### Step 0.1: Create your project repo 🖥️ WINDOWS PC
Do this in a terminal on your Windows machine (PowerShell, Windows Terminal, or WSL if you have it — either is fine; Claude Code runs in any of them).

```powershell
mkdir splunk-log-project
cd splunk-log-project
git init
mkdir compose, splunk-config, splunk-config\inputs, splunk-config\outputs, jenkins, docs, sample-app
```

**Claude Code prompt** (run inside the new folder, on your Windows PC):
```
Initialize this as a project scaffold for a Dockerized Splunk log-management
stack. Create a .gitignore appropriate for Docker/Splunk/Jenkins projects
(ignore volumes, .env, *.log, node_modules if a Spring Boot/Node sample app
is added later). Create a top-level README.md stub with sections:
Overview, Architecture, Prerequisites, Setup, Deployment, Screenshots,
Team Members. Leave the sections mostly empty for now — just the headers
and one-line placeholders — I'll fill them in as I build.
```

### Step 0.2: Push the repo to GitHub/GitLab 🖥️ WINDOWS PC
You need a remote so the Ubuntu VM (and later, Jenkins) can pull your code without you manually copying files back and forth.

```powershell
git add .
git commit -m "Initial scaffold"
git remote add origin <your-repo-url>
git push -u origin main
```

### Step 0.3: Confirm SSH access to the VM 🖥️ WINDOWS PC → ☁️ UBUNTU VM
From Windows Terminal / PowerShell:
```powershell
ssh <your-user>@dev.itccloud.top
```
Once connected you're now *on* the Ubuntu VM for the rest of that session. Every command below marked ☁️ UBUNTU VM assumes you're SSH'd in.

---

## Phase 1 — Provision the Cloud VM

### Step 1.1: Spin up an Ubuntu VM on Proxmox 🖥️ WINDOWS PC (browser) → creates the ☁️ UBUNTU VM
This is manual, done through a browser on your Windows PC — you can't script the Proxmox web UI with Claude Code.

1. Log into the Proxmox web UI for `dev.itccloud.top` from your Windows browser.
2. Create a new VM: Ubuntu Server 22.04 LTS, at least **2 vCPU / 4GB RAM / 40GB disk** (Splunk is not light — go 4GB+ if your quota allows).
3. Set a static IP or note the DHCP-assigned IP — you'll need it to reach the Splunk Web UI (port 8000) and Jenkins later.
4. Boot it, then SSH in from your Windows PC (Step 0.3) and confirm you have sudo access.

### Step 1.2: Install Docker Engine + Docker Compose ☁️ UBUNTU VM
You're SSH'd into the VM for this — these commands do NOT run on Windows.

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

**Checkpoint (☁️ VM):** `docker run hello-world` should succeed.

### Step 1.3: Clone your repo onto the VM ☁️ UBUNTU VM
```bash
git clone <your-repo-url>
cd splunk-log-project
```
This is how the config files you write on Windows actually get onto the machine that runs them. You'll `git pull` here (or let Jenkins do it) every time you push new changes from Windows.

---

## Phase 2 — Author the Docker Compose Stack

**Where this happens:** you *write* the compose file on 🖥️ WINDOWS PC with Claude Code, then *run* it on ☁️ UBUNTU VM.

### Step 2.1: Write docker-compose.yml 🖥️ WINDOWS PC
Back in your Windows terminal, inside the repo folder.

**Claude Code prompt:**
```
Create a docker-compose.yml in the compose/ folder for a log-management
stack with these services:

1. "splunk" — image splunk/splunk:latest, acting as indexer + search head.
   - Set SPLUNK_START_ARGS=--accept-license and SPLUNK_PASSWORD via a .env
     file (don't hardcode the password in the compose file).
   - Expose port 8000 (Web UI) and 9997 (forwarder receiving port).
   - Mount a named volume for /opt/splunk/var so indexed data persists
     across container recreation.
   - Attach to a custom bridge network called "logging-net".

2. "forwarder" — image splunk/universalforwarder:latest.
   - Configure it via environment variables to forward to splunk:9997.
   - Mount a volume with the app/httpd logs so it has something to tail.
   - Attach to "logging-net", depends_on splunk.

3. "sample-app" — image httpd:latest (Apache).
   - Mount its /usr/local/apache2/logs to a shared volume the forwarder
     can also read.
   - Attach to "logging-net".

Also create a .env.example file (not .env) documenting the required
variables (SPLUNK_PASSWORD, etc.) without real secrets. Add a note at
the top of docker-compose.yml explaining the architecture in a comment
block. Use Docker Compose file format version that doesn't require the
deprecated "version:" key.
```

Review what Claude Code generates — check the volume names, network name, and env var names all match across services.

### Step 2.2: Push and pull the compose file to the VM
🖥️ **WINDOWS PC:**
```powershell
git add compose/ .env.example
git commit -m "Add docker-compose stack"
git push
```
☁️ **UBUNTU VM** (SSH session):
```bash
git pull
cd compose
cp .env.example .env   # edit with a real password: nano .env
docker compose up -d
docker compose ps
```

### Step 2.3: Verify it's running
🖥️ **WINDOWS PC** — open a browser and visit `http://<VM-IP>:8000` (this works from Windows since the VM is reachable over the network; you're just viewing the web UI remotely, nothing installs locally). Log in with `admin` / your password.

**Checkpoint:** ☁️ on the VM, `docker compose logs forwarder` should show it successfully connecting to the indexer on 9997 (no repeated connection-refused errors).

---

## Phase 3 — Configure Splunk (Inputs, Indexes, Source Types)

### Step 3.1: Create a dedicated index 🖥️ WINDOWS PC (browser, pointed at the VM)
In the Splunk Web UI (`http://<VM-IP>:8000`, opened from your Windows browser): **Settings → Indexes → New Index**. Name it something like `webapp_logs`. Set reasonable retention and document your choice.

### Step 3.2: Write inputs.conf / outputs.conf 🖥️ WINDOWS PC
Generate these locally so they're version-controlled, rather than hand-editing inside the container.

**Claude Code prompt:**
```
In splunk-config/, create:

1. outputs.conf — configures the Universal Forwarder to send data to
   the indexer container at splunk:9997, with appropriate
   [tcpout] and [tcpout:default-autolb-group] stanzas.

2. inputs.conf — configures the forwarder to monitor Apache access
   and error logs (e.g. /var/log/httpd or the mounted apache logs path)
   and tag them with sourcetype "access_combined" for access logs and
   "httpd_error" for error logs, indexed into "webapp_logs".

3. props.conf — a minimal source type definition for a custom
   sourcetype if the log format needs custom line-breaking or
   timestamp extraction (assume standard Apache combined log format
   for now).

Add comments explaining each stanza. Then update docker-compose.yml
to mount these three files into the forwarder container's
etc/system/local/ directory as read-only volumes.
```

### Step 3.3: Push, pull, and restart
🖥️ **WINDOWS PC:**
```powershell
git add splunk-config/ compose/docker-compose.yml
git commit -m "Add Splunk input/output config"
git push
```
☁️ **UBUNTU VM:**
```bash
git pull
docker compose restart forwarder
```

### Step 3.4: Confirm ingestion
🖥️ **WINDOWS PC** (browser) — in Splunk Web UI → **Search & Reporting**, run:
```
index=webapp_logs
```
Generate some traffic first so there's something to see — you can do this from either machine since it's just an HTTP request to the VM's Apache port:
```bash
curl http://<VM-IP>:<apache-port>/
```
(run a few times, from Windows PowerShell or the VM, doesn't matter which)

**Checkpoint:** search returns real events with correct sourcetype and timestamps.

---

## Phase 4 — Build Dashboards, Saved Searches, and an Alert

### Step 4.1: Get the SPL written 🖥️ WINDOWS PC
**Claude Code prompt:**
```
Write three Splunk SPL searches for an index called webapp_logs with
sourcetype access_combined (standard Apache combined log format):

1. A search that counts events by HTTP status code over time, suitable
   for a timechart panel (status code trend dashboard).
2. A saved search that finds the top 10 requested URIs by hit count.
3. An alert search that triggers when the count of 5xx status codes
   in the last 5 minutes exceeds a threshold (say, 5).

Output each as plain SPL with a one-line comment above explaining what
it does and what dashboard/alert it's meant to power.
```

### Step 4.2: Build the dashboard, saved search, and alert 🖥️ WINDOWS PC (browser, pointed at the VM)
This part is UI-driven and can't be scripted with Claude Code — you're clicking through the Splunk Web UI in your Windows browser, but everything you build lives on the VM.

- **Dashboards → Create New Dashboard** → paste in the timechart SPL, choose a visualization. Add a second panel for the top-URIs search as a table.
- **Save As → Report** for the top-URIs search.
- For the 5xx spike search: **Save As → Alert**, run every 5 minutes, trigger on "number of results > 0."

**Checkpoint:** trigger a burst of errors (curl a nonexistent path a few times) and confirm the alert fires.

### Step 4.3: Export dashboard XML back into your repo 🖥️ WINDOWS PC
In the Splunk UI, **Edit → Source** on your dashboard to view the raw XML. Copy it into a file on your Windows machine:
```powershell
mkdir splunk-config\dashboards
# paste dashboard XML into splunk-config\dashboards\http_status_dashboard.xml
git add splunk-config/dashboards
git commit -m "Version-control dashboard definition"
git push
```

---

## Phase 5 — Set Up Jenkins CI/CD

### Step 5.1: Run Jenkins as a container ☁️ UBUNTU VM
SSH'd into the VM:
```bash
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts
```
Mounting the Docker socket lets Jenkins run `docker compose` commands on the VM's own Docker engine.

### Step 5.2: Unlock and configure Jenkins 🖥️ WINDOWS PC (browser, pointed at the VM)
Visit `http://<VM-IP>:8080` from your Windows browser. Get the initial admin password by running this ☁️ on the VM: `docker logs jenkins`. Install suggested plugins, create an admin user.

### Step 5.3: Write the Jenkinsfile 🖥️ WINDOWS PC
**Claude Code prompt:**
```
Create a Jenkinsfile (declarative pipeline) in jenkins/ with these stages:

1. Checkout — pull the latest code from the git repo.
2. Validate — run `docker compose config` against compose/docker-compose.yml
   to check the file is syntactically valid before deploying.
3. Deploy — run `docker compose -f compose/docker-compose.yml up -d --build`
   to (re)deploy the stack.
4. Health Check — curl the Splunk Web UI on port 8000 and fail the build
   if it doesn't return HTTP 200 within a reasonable retry/timeout loop
   (e.g. retry every 10s for up to 2 minutes).

Use environment variables for the VM path/working directory rather than
hardcoding it. Add post { success {...} failure {...} } blocks that echo
a clear pass/fail message. Comment each stage briefly.
```

Push it:
```powershell
git add jenkins/Jenkinsfile
git commit -m "Add Jenkins pipeline"
git push
```

### Step 5.4: Create the Jenkins job 🖥️ WINDOWS PC (browser, pointed at the VM)
In Jenkins: **New Item → Pipeline**. Point it at your git repo, set "Pipeline script from SCM," path to `jenkins/Jenkinsfile`. Run it manually first to confirm it works end-to-end. (The pipeline itself executes on the ☁️ VM — Jenkins is just controlled from your browser.)

**Checkpoint:** a full pipeline run should tear down/redeploy the stack and end green, with Splunk reachable afterward and your indexed data still present.

### Step 5.5: Test data persistence across redeploy ☁️ UBUNTU VM + 🖥️ WINDOWS PC (browser)
☁️ On the VM:
```bash
docker compose down          # containers gone, volumes persist
```
Then 🖥️ from Windows, trigger the Jenkins pipeline again via the browser UI. Once it finishes, check in the Splunk Web UI (also from Windows) that your historical search results are still there. This directly verifies a claim from your proposal — screenshot it.

---

## Phase 6 — Documentation and Final Report

Everything in this phase is 🖥️ **WINDOWS PC** — writing docs, taking screenshots of things running on the VM, and packaging the submission.

### Step 6.1: Write the README 🖥️ WINDOWS PC
**Claude Code prompt** (run from the repo root on Windows, after everything above works):
```
Look at the current state of this repo — docker-compose.yml, the Splunk
config files, the Jenkinsfile, and the docs/ folder. Write a complete
README.md with these sections:

- Overview (what the project does, one paragraph)
- Architecture (describe the containers, network, volumes, and data flow
  from log source -> forwarder -> indexer -> dashboard/alert)
- Prerequisites (Docker, Docker Compose, access to the Proxmox VM)
- Setup instructions (clone repo, configure .env, docker compose up)
- Jenkins pipeline (what it does, how to trigger it)
- Dashboards & Alerts (what they show, how to view them)
- Repository structure (brief tree with one-line descriptions)

Write it as if a classmate with no context needs to reproduce this
deployment from scratch. Be specific about ports and file paths actually
used in this repo — don't invent generic placeholders.
```

### Step 6.2: Take screenshots as you go 🖥️ WINDOWS PC
Every screenshot is of something running on the VM, but captured from your Windows browser/terminal as you complete each phase — don't leave this to the end:
- Splunk Web UI home page
- `index=webapp_logs` search returning real events
- Dashboard with both panels populated
- Alert configuration + a triggered-alert instance
- A green Jenkins pipeline run
- `docker compose ps` output (screenshot your SSH terminal) showing all 3 containers healthy

Save into `docs/screenshots/` and commit them.

### Step 6.3: Write the final report 🖥️ WINDOWS PC
**Claude Code prompt** (once README and screenshots are ready):
```
Using the README.md and the docker-compose.yml, Jenkinsfile, and Splunk
config files in this repo as source material, draft a final project
report in Markdown (docs/final_report.md) covering:

1. Project Description & Objectives
2. System Architecture (describe the diagram I'll insert manually)
3. Implementation Details — walk through each config decision
   (indexer setup, forwarder config, index/sourcetype choices) and why
   it was made
4. Deployment & Cloud Usage — how the stack runs on the Proxmox VM
5. Automation — how the Jenkins pipeline works end to end, referencing
   the actual pipeline stages
6. Dashboards & Alerts — what was built and what it demonstrates
7. Challenges & Lessons Learned — leave this as a bullet list with
   placeholders for me to fill in with my own experience
8. Conclusion

Keep it factual and grounded in what's actually in this repo — don't
invent features I didn't build. Target roughly 1500-2500 words.
```

Then convert this to a polished Word doc — I can generate that .docx for you directly once the content is finalized; just paste the markdown back to me or tell me to pull it from your repo.

### Step 6.4: Package the final submission 🖥️ WINDOWS PC
Your three required submission components:
1. **Proposal document** — you already have this.
2. **Final report** — the polished version of Step 6.3, as a Word doc or PDF.
3. **Implementation details** — your git repo: `docker-compose.yml`, all `.conf` files, the `Jenkinsfile`, dashboard XML, and README.

```powershell
git add .
git commit -m "Final project submission"
git archive --format=zip HEAD -o ../submission_implementation.zip
```

---

## Quick Reference: Where You Are at Each Phase

| Phase | Primary machine |
|---|---|
| 0 — Repo scaffold, SSH check | 🖥️ Windows (writing) |
| 1 — VM provisioning, Docker install | 🖥️ Windows (Proxmox browser) → ☁️ VM (Docker install) |
| 2 — Compose stack | 🖥️ Windows (write) → ☁️ VM (run) |
| 3 — Splunk config | 🖥️ Windows (write + browser UI) → ☁️ VM (restart) |
| 4 — Dashboards/alerts | 🖥️ Windows (SPL + browser UI) |
| 5 — Jenkins | ☁️ VM (runs Jenkins) + 🖥️ Windows (browser + Jenkinsfile authoring) |
| 6 — Docs/report | 🖥️ Windows entirely |

**Rule of thumb:** Claude Code always runs on your Windows PC, writing files into your git repo. The VM only ever sees those files after a `git pull` (or a Jenkins-triggered pull) — you're never running Claude Code directly on the VM in this workflow, and you're never running Docker directly on your Windows PC.