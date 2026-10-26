# website_blocker_test_system
PDL mini project
# Remote Lab Website Blocker & Management System

A centralized, cloud-controlled endpoint security and focus tool designed for computer labs. An administrator can open a mobile-friendly dashboard on their phone, view a visual map of all lab PCs, select any computer, and immediately block distracting websites in real time while displaying a custom notification banner on that PC.

---

## Table of Contents
1. [Project Overview & Architecture](#project-overview--architecture)
2. [Tech Stack & Free Cloud Hosting](#tech-stack--free-cloud-hosting)
3. [System Contracts & Data Schemas (The Integration Bridge)](#system-contracts--data-schemas)
4. [Team Division (4 Roles)](#team-division-4-roles)
   - [Person 1: Backend Server & Real-time Communication](#person-1-backend-server--real-time-api)
   - [Person 2: Admin Dashboard & Visual Lab Layout (Frontend)](#person-2-admin-dashboard--visual-lab-layout-frontend)
   - [Person 3: Windows Client Agent & Enforcement Engine](#person-3-windows-client-agent--enforcement-engine)
   - [Person 4: DevOps, Packaging, Integration & Deployment](#person-4-devops-packaging-integration--deployment)
5. [Real-World Gotchas Solved](#real-world-gotchas-solved)
6. [Step-by-Step Merging & Integration Guide](#step-by-step-merging--integration-guide)

---

## Project Overview & Architecture

```
   [ Admin's Phone / Browser ]
                │
                │ HTTPS / WSS
                ▼
      ┌──────────────────┐
      │  Vercel Frontend │ (Visual Grid Map of Computer Lab)
      └─────────┬────────┘
                │ REST API / WebSocket
                ▼
      ┌──────────────────┐
      │  Render Backend  │ (FastAPI + SQLite/PostgreSQL + WebSockets)
      └─────────┬────────┘
                │ Persistent WebSocket (Real-Time Push)
        ┌───────┴────────────────────────┐
        ▼                                ▼
[ Lab PC 01 (Agent) ]            [ Lab PC 02 (Agent) ]
- Windows Service (Admin)        - Windows Service (Admin)
- Modifies Hosts File            - Modifies Hosts File
- Disables Browser DoH           - Disables Browser DoH
- Flushes DNS Cache              - Flushes DNS Cache
- Pops screen message toast      - Pops screen message toast
```

---

## Tech Stack & Free Cloud Hosting

| Component | Technology | Free Hosting Option |
| :--- | :--- | :--- |
| **Backend API & WebSockets** | Python (FastAPI + Uvicorn) | **Render.com** (Free Web Service) or Railway |
| **Database** | SQLite (development) / Supabase PostgreSQL | **Render Disk** or **Supabase Free Tier** |
| **Frontend Web App** | React / Next.js (or Tailwind + Vite) | **Vercel** or **Netlify** |
| **PC Client Agent** | Python 3.10+ (`websockets`, `plyer` or `tkinter`) | Local installation on each Windows PC |
| **Agent Daemonizer** | NSSM (Non-Sucking Service Manager) | Runs agent silently under `SYSTEM` account |

---

## System Contracts & Data Schemas

> **CRITICAL RULE FOR ALL 4 DEVS:** Do not change these JSON structures without discussing with the team. These schemas allow all 4 people to work 100% independently using mock data.

### 1. Device Registration & Heartbeat (Agent -> Backend)
```json
{
  "type": "HEARTBEAT",
  "pc_id": "LAB-PC-01",
  "hostname": "DESKTOP-CHEMLAB-1",
  "ip_address": "192.168.1.101",
  "status": "ONLINE",
  "is_blocked": false
}
```

### 2. Block/Unblock Command (Frontend -> Backend -> Agent)
```json
{
  "type": "COMMAND",
  "pc_id": "LAB-PC-01",
  "action": "BLOCK",            // "BLOCK" or "UNBLOCK"
  "websites": [
    "facebook.com",
    "www.facebook.com",
    "youtube.com",
    "www.youtube.com",
    "instagram.com",
    "tiktok.com"
  ],
  "alert_message": "Browsing is restricted during the practical exam by the Lab Admin."
}
```

### 3. Lab PC State Object (Backend -> Frontend)
```json
[
  {
    "pc_id": "LAB-PC-01",
    "grid_row": 1,
    "grid_col": 1,
    "hostname": "DESKTOP-CHEMLAB-1",
    "status": "ONLINE",         // "ONLINE" | "OFFLINE"
    "is_blocked": true,
    "last_seen": "2026-10-26T12:00:00Z",
    "alert_message": "Lab session active"
  }
]
```

---

## Team Division (4 Roles)

---

### Person 1: Backend Server & Real-time API
**Goal:** Build the cloud API that coordinates all PC agents and the admin frontend.

#### Directory: `/backend`

#### Step-by-Step Instructions:
1. **Setup Project:**
   - Create a FastAPI application (`backend/main.py`).
   - Create a `requirements.txt` with: `fastapi`, `uvicorn[standard]`, `websockets`, `pydantic`.
2. **Implement Connection Manager (`backend/ws_manager.py`):**
   - Maintain two dictionaries of active WebSocket connections:
     - `admin_sockets`: Active browsers viewing the dashboard.
     - `pc_sockets`: Active PC agents mapped by `pc_id` (e.g., `{"LAB-PC-01": WebSocket}`).
3. **API Endpoints (`backend/main.py`):**
   - `GET /api/devices`: Returns the list and status of all registered PCs.
   - `POST /api/block`: Accepts `{ pc_id, action, websites, alert_message }`.
     - Validates payload.
     - Finds the target PC in `pc_sockets`.
     - Forwards the command via WebSocket in under 1 second.
     - Broadcasts updated status to all `admin_sockets`.
4. **Agent WebSocket Endpoint (`backend/main.py`):**
   - `WebSocket /ws/agent/{pc_id}`:
     - When a PC connects, mark status as `ONLINE`.
     - On disconnect, mark as `OFFLINE` and broadcast change to admins.
     - Handle periodic `HEARTBEAT` messages.
5. **Admin WebSocket Endpoint (`backend/main.py`):**
   - `WebSocket /ws/admin`:
     - Emits live state whenever any PC goes online/offline/blocked.
6. **CORS & Environment:**
   - Add CORS middleware (`allow_origins=["*"]`) so the hosted frontend can communicate without browser errors.

#### Output Deliverables:
- Working FastAPI app with interactive docs (`/docs`).
- Ready to deploy on Render or Railway with a single command.

---

### Person 2: Admin Dashboard & Visual Lab Layout (Frontend)
**Goal:** Build a mobile-responsive web app showing the physical layout of the lab with one-tap controls.

#### Directory: `/frontend`

#### Step-by-Step Instructions:
1. **Setup Project:**
   - Initialize with Vite + React + Tailwind CSS (or Next.js).
2. **Visual Lab Grid (`frontend/src/components/LabGrid.jsx`):**
   - Render an interactive floor plan / grid of workstations representing the lab (e.g., a 4x5 grid for 20 PCs).
   - Each PC card should display:
     - Icon (Desktop monitor).
     - Name/ID (e.g., `PC-01`, `PC-02`).
     - Badge/Dot for status:
       - 🟢 Green = Online (Normal)
       - 🔴 Red = Online (Blocked)
       - ⚪ Gray = Offline
3. **PC Detail & Action Modal / Drawer (`frontend/src/components/PCModal.jsx`):**
   - When any PC is clicked:
     - Display the selected **PC ID** prominently at the top.
     - Display current status (`ONLINE` / `OFFLINE` / `BLOCKED`).
     - Input field for **Custom Alert Message** (e.g., *"Focus on task 3 - Social media disabled"*).
     - Preset selection or multi-select of categories to block (e.g. *Social Media*, *Video Streaming*, *Gaming*, or Custom URLs).
     - Prominent action buttons:
       - 🛑 **"Block This PC"** (Red)
       - 🟢 **"Unblock PC"** (Green)
       - ⚡ **"Block All Lab PCs"** (Global emergency switch)
4. **WebSocket Integration (`frontend/src/hooks/useLabSocket.js`):**
   - Connect to `wss://<BACKEND_URL>/ws/admin`.
   - Update PC state dynamically in real time without needing to refresh the page.
5. **Mobile Responsiveness:**
   - Use responsive flex/grid layouts so the admin can comfortably navigate and tap PC nodes from an Android or iPhone screen.

#### Output Deliverables:
- Standalone web app that can run with mock backend data or connect to Person 1's live API URL.

---

### Person 3: Windows Client Agent & Enforcement Engine
**Goal:** Build the background Python script that runs on each lab computer with admin rights to enforce blocking and display notifications.

#### Directory: `/agent`

#### Step-by-Step Instructions:
1. **Configuration (`agent/config.py`):**
   - Store `SERVER_WS_URL = "wss://your-render-app.onrender.com/ws/agent/"`
   - Store local `PC_ID` (read from `pc_config.json` or machine hostname).
2. **Hosts File Modifier (`agent/blocker.py`):**
   - Path on Windows: `C:\Windows\System32\drivers\etc\hosts`.
   - `block_domains(domain_list)`:
     - Read the hosts file.
     - Append lines: `127.0.0.1 <domain>` (both with and without `www.`).
     - Save and flush DNS using `subprocess.run(["ipconfig", "/flushdns"], capture_output=True)`.
   - `unblock_all()`:
     - Remove previously added blocker lines and restore the clean hosts file.
     - Flush DNS again.
3. **Bypass Prevention / DoH Disable (`agent/anti_bypass.py`):**
   - Modern browsers bypass the `hosts` file using DNS-over-HTTPS.
   - Add a Python function modifying Windows Registry to disable DoH in Chrome and Edge:
     ```python
     # Disables DoH in Google Chrome via Registry
     # Path: HKLM\SOFTWARE\Policies\Google\Chrome -> BuiltInDnsClientEnabled = 0
     ```
4. **Desktop Notification Banner (`agent/notifier.py`):**
   - When a block command arrives with `alert_message`, display a non-closable banner or top notification window (using `tkinter` topmost window or Windows toast via `plyer`/`win10toast`):
     - Window Title: *"Lab Notice"*
     - Body: *"{alert_message}"*
     - Shows for 10 seconds or stays sticky until unblocked.
5. **Agent WebSocket Listener (`agent/agent.py`):**
   - Connects to `SERVER_WS_URL + PC_ID`.
   - Sends heartbeat every 30 seconds.
   - On receiving `"action": "BLOCK"`, calls `block_domains(msg["websites"])` and shows `alert_message`.
   - On receiving `"action": "UNBLOCK"`, calls `unblock_all()`.
   - Auto-reconnect with exponential backoff if the internet drops.

#### Output Deliverables:
- A standalone script `agent.py` that can be run with `python agent.py` in Administrator Command Prompt to test live.

---

### Person 4: DevOps, Packaging, Integration & Deployment
**Goal:** Bridge all 3 parts, deploy backend/frontend to free internet hosts, and create a 1-click Windows installer.

#### Directory: `/scripts`, `/deploy`

#### Step-by-Step Instructions:
1. **Deploy Backend to Render (Free):**
   - Create a `render.yaml` or connect GitHub repository to [Render.com](https://render.com).
   - Set Build Command: `pip install -r backend/requirements.txt`
   - Set Start Command: `uvicorn backend.main:app --host 0.0.0.0 --port $PORT`
   - Share public HTTPS/WSS URL with Person 2 and Person 3.
2. **Deploy Frontend to Vercel (Free):**
   - Connect frontend folder to [Vercel](https://vercel.com).
   - Set Environment Variable `VITE_BACKEND_URL = https://your-backend.onrender.com`.
3. **Windows Service Installer (`scripts/install_service.bat`):**
   - Download `nssm.exe` (lightweight service helper).
   - Write a batch script that:
     1. Checks for Administrator privileges.
     2. Installs Python dependencies (`pip install -r agent/requirements.txt`).
     3. Prompts the technician for `PC_ID` (e.g., `LAB-PC-05`) and writes to `agent/pc_config.json`.
     4. Installs the agent as a Windows Background Service via NSSM:
        ```bat
        nssm install LabBlockerAgent "C:\Python310\python.exe" "C:\LabAgent\agent.py"
        nssm set LabBlockerAgent AppDirectory "C:\LabAgent"
        nssm start LabBlockerAgent
        ```
     5. The service will now automatically start on boot with full `SYSTEM` administrative privileges.
4. **End-to-End Test Plan:**
   - Run verification checklist:
     - [ ] PC connects to cloud WebSocket upon boot.
     - [ ] Dashboard shows PC turning Green.
     - [ ] Admin clicks PC on mobile -> selects "Block" -> PC turns Red.
     - [ ] On the PC: Notification pops up -> opening YouTube in Chrome fails.
     - [ ] Admin clicks "Unblock" -> hosts restored -> YouTube works immediately.

---

## Real-World Gotchas Solved

| Problem | Cause | How This Project Solves It |
| :--- | :--- | :--- |
| **Browser ignores hosts file** | Chrome/Edge use DoH (DNS-over-HTTPS) to Cloudflare/Google DNS | Person 3's script pushes registry keys `BuiltInDnsClientEnabled = 0` to force browsers to use local OS DNS. |
| **Site still opens after block** | Local DNS cache holds old IP for several minutes | Agent automatically runs `ipconfig /flushdns` immediately after editing the hosts file. |
| **Permission Denied editing hosts** | Windows protects the drivers file from standard users | Agent is installed as a Windows Service running as `LocalSystem`, which has root Windows access. |
| **Students closing Python terminal** | Student sees Python command window and closes it | Script runs headless as a background Windows Service (`pythonw.exe` / NSSM) with no visible console. |
| **PC restarted after block** | Temporary in-memory blocks reset upon reboot | Hosts file modifications persist across reboots, and the agent re-syncs state with the server immediately on boot. |

---

## Step-by-Step Merging & Integration Guide

### Phase 1: Independent Work (Day 1 - 2)
- **Person 1:** Writes FastAPI endpoints and tests them using Postman/Swagger UI (`/docs`).
- **Person 2:** Builds UI with hardcoded list of 20 lab PCs and tests modal selection and mobile responsiveness.
- **Person 3:** Tests modifying the `hosts` file locally on a Windows test machine and flushing DNS.
- **Person 4:** Prepares GitHub repo, branches (`feat/backend`, `feat/frontend`, `feat/agent`, `feat/devops`), and deployment pipelines.

### Phase 2: Cloud Connection (Day 3)
- Person 4 deploys Person 1's backend to Render.
- Person 2 updates frontend WebSocket URL to connect to live Render backend.
- Person 3 points agent WebSocket URL to Render backend.

### Phase 3: Final Merge & Lab Rollout (Day 4)
1. Merge all 4 feature branches into `main`.
2. Run `scripts/install_service.bat` on test Lab PC.
3. Open admin URL on mobile phone.
4. Select the PC, click **Block**, and confirm live blocking.
