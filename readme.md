# ZKTeco F18 ADMS Push Protocol Integration Guide & Technical Specification

> ℹ️ **Notice:** The codebase in this repository is a **Testing & Inspection Sandbox Environment** for capturing, inspecting, and debugging incoming HTTP payloads from ZKTeco devices. This documentation serves as a standalone **Production Specification & Integration Reference** for connecting the **ZKTeco F18** device directly to any backend application.
> 
> 📌 **ADMS Full Form:** **Automatic Data Master Server** *(ZKTeco's HTTP Push Communication Technology)*

---

## 🏗️ System Architecture & Protocol Flow

The **ZKTeco F18** device communicates directly with your backend server using the **ZKTeco Push ADMS (Automatic Data Master Server) Protocol** over standard HTTP/HTTPS. No proprietary SDKs or DLLs are required on the server.

```
┌────────────────────────────────┐        HTTP Push (ADMS)        ┌───────────────────────────────┐
│  ZKTeco F18 Physical Device    │ ─────────────────────────────> │  Your Production Backend      │
│  (Hardware Terminal at Office) │                                │  (Node.js / PHP / Python)     │
│  Port: 80 / 443 / Custom       │ <───────────────────────────── │  Port: 5959 / 80 / 443        │
└────────────────────────────────┘      Plain-Text Handshake      └───────────────────────────────┘
```

---

## ⚙️ Step-by-Step ZKTeco Hardware Setup Guide (Physical Device Menu)

Follow these exact steps on your physical **ZKTeco F18** keypad to connect the device to your server:

```
[Physical Machine Keypad Setup Flowchart]

Press [M/OK] ➔ Comm. (Communication) ➔ Cloud Server Setting / ADMS ➔ Set Server IP & Port ➔ Save & Restart
```

### 📋 Detailed Menu Steps (F18 Keypad):

1. **Open Main Menu:**
   - Press and hold the **`[M/OK]`** button on the ZKTeco F18 keypad for 3 seconds.
   - (If Admin security is enabled, scan Admin Fingerprint or enter Admin Password).

2. **Open Communication Menu:**
   - Navigate to **`Comm.`** (Communication / যোগাযোগ) and press **`[M/OK]`**.

3. **Open Cloud Server Settings:**
   - Scroll down and select **`Cloud Server Setting`** (or **`ADMS`** / **`Web Server Setting`**) and press **`[M/OK]`**.

4. **Configure Server Address & Parameters:**
   - **Server Mode:** Set to **`ADMS`** *(Automatic Data Master Server)*
   - **Enable Domain Name:** Set to **`ON`** (for HTTPS domain support) or **`OFF`** (for direct IP)
   - **Server Address:** Enter **`<YOUR_SERVER_IP_OR_DOMAIN>`** *(e.g., `zk.yourcompany.com` or VPS IP)*
   - **Server Port:** Enter **`<YOUR_SERVER_PORT>`** *(e.g., 443 for HTTPS or 5959)*
   - **Enable Proxy Server:** Set to **`OFF`**
   - **Request Path:** `/iclock/cdata` (Default)

5. **Save & Restart Device:**
   - Press **`[M/OK]`** or **`[ESC]`** to save changes.
   - Restart/Reboot the F18 machine.

Within 5-10 seconds of restarting, the network icon on the F18 screen will show connected, and the device will automatically initiate its handshake with your backend server.

---

## 📡 ZKTeco F18 ADMS Protocol Specification

When connected, the ZKTeco F18 device communicates directly with the backend via the following HTTP endpoints:

| Endpoint | HTTP Method | Purpose | Server Response Requirement |
| :--- | :--- | :--- | :--- |
| `/iclock/cdata?options=all` | `GET` | Initial Handshake & Delay Negotiation | Plain Text (`Delay=30\nRealtime=1`) |
| `/iclock/cdata?table=ATTLOG` | `POST` | Real-time Attendance Punches Stream | Plain Text (`OK: <line_count>`) |
| `/iclock/getrequest` | `GET` | Device Heartbeat & Command Query | Plain Text (`OK` or `C:<id>:<cmd>`) |
| `/iclock/devicecmd` | `POST/GET` | Device Command Execution Result | Plain Text (`OK`) |

### Server Response Specifications:

#### 1. Handshake Response (`GET /iclock/cdata?options=all`)
Your backend MUST respond with HTTP 200 OK and `Content-Type: text/plain`:
```text
GET OPTION FROM: <DEVICE_SERIAL_NUMBER>
Stamp=9999
OpStamp=9999
ErrorDelay=60
Delay=30
TransTimes=00:00;23:59
TransInterval=1
TransFlag=1111111111
Realtime=1
Encrypt=0
```
- **`Delay=30`**: Instructs the F18 device to poll for commands every 30 seconds.
- **`Realtime=1`**: Instructs the F18 device to stream fingerprint punches immediately (in 1 second) whenever a user scans their finger.

#### 2. Log Acknowledgement (`POST /iclock/cdata?table=ATTLOG`)
Your backend MUST acknowledge receiving the attendance log batch:
```text
OK: <count>
```
*(e.g., `OK: 1` or `OK: 5`). Returning `OK` tells the F18 machine that the logs were saved so it will not resend them.*

---

## 📊 Attendance Log Payload Format (`ATTLOG`)

When a user scans a fingerprint, card, or password on the ZKTeco F18, the machine sends a `POST` request to `/iclock/cdata?table=ATTLOG` containing a tab-separated (`\t`) string body.

### Raw Body Payload Example:
```text
4	2026-08-19 12:51:37	255	1		0	0
37	2026-08-19 12:55:10	255	1		0	0
```

### Tab-Separated Column Schema:

| Column Index | Field Name | Type | Example | Description |
| :---: | :--- | :---: | :--- | :--- |
| **0** | `User ID / Enroll ID` | String | `4` | Employee ID registered on the F18 machine |
| **1** | `Punch Timestamp` | DateTime | `2026-08-19 12:51:37` | Exact time the fingerprint punch occurred |
| **2** | `Attendance State` | Integer | `255` | `255` / `0` = Check-In / General Punch |
| **3** | `Verify Mode` | Integer | `1` | Verification method code |
| **4-6** | `Reserved / WorkCode` | String | `0` | Default internal system flags |

### Verification Modes (`Verify Mode`):
- **`1`**: Fingerprint (ফিংগারপ্রিন্ট)
- **`15`**: Face Recognition (ফেস)
- **`0`** / **`3`**: RFID Card / Badge (কার্ড)
- **`2`**: Password / PIN (পাসওয়ার্ড)

---

## 🔒 Production Security Requirements

When deploying ZKTeco F18 integration to a live Production environment:

### 1. Device Serial Number (SN) Whitelisting
- ZKTeco F18 devices identify themselves using their unique **Device Serial Number (`SN`)** in the HTTP request URL query string:
  ```url
  POST /iclock/cdata?SN=COAW221060606&table=ATTLOG
  ```
- **Security Rule:** Maintain a database table of authorized ZKTeco Device Serial Numbers. On every incoming request, verify that `$_GET['SN']` exists in your database. If an unknown `SN` attempts to push logs, reject the payload immediately with `HTTP 403 Forbidden`.

### 2. Strict User-Agent Signature Verification
- ZKTeco F18 devices send a distinct internal user-agent header:
  ```http
  User-Agent: iClock Proxy/1.09
  ```
- **Security Rule:** Verify that incoming HTTP requests contain valid ZKTeco hardware signatures (`iClock Proxy`, `iClock`, or `ZKTeco`). Reject generic browser or bot user-agents attempting to hit `/iclock/` endpoints.

### 3. API Rate Limiting (Spam & Denial-of-Service Protection)
- Enforce strict Rate Limiting rules on the receiver endpoint to prevent brute-force payload injections or DoS attacks.
- **Rule Limit:** Allow a maximum of **100 requests per minute per IP address** (`100 req/min/IP`).
- Configure Nginx Rate Limiting:
  ```nginx
  limit_req_zone $binary_remote_addr zone=zkteco_limit:10m rate=100r/m;
  
  location /iclock/ {
      limit_req zone=zkteco_limit burst=20 nodelay;
      proxy_pass http://127.0.0.1:5959;
  }
  ```

### 4. HTTPS / SSL Encryption (Domain Name or IP SSL)
- **Domain SSL:** In F18 keypad menu, set **`Enable Domain Name = ON`**, enter domain `zk.yourcompany.com` and Port `443` to use free **Let's Encrypt / Certbot** SSL.
- **IP SSL:** Alternatively, issue a direct **Public IP SSL Certificate** (via ZeroSSL) for your server IP.

---

## 📝 License & Maintainer
Created for **ZKTeco F18 Biometric Hardware ADMS Push Integration**. Free for commercial and private deployment.
