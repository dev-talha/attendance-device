# TIMMY AI05 / AiFace Biometric Hardware Universal HTTP Integration Guide

> **Framework-Agnostic Developer Integration Guide & Protocol Reference**  
> **Target Hardware:** TIMMY AI05, AiFace, and Compatible Biometric JSON Push Terminals  
> **Compatible Tech Stack:** Any Backend (Laravel, Node.js/Express, Python/Django/FastAPI, Raw PHP, ASP.NET, Go, Java Spring Boot, Ruby on Rails)

---

## Table of Contents

1. [Overview & Architecture](#overview--architecture)
2. [Physical Machine Keypad Setup](#physical-machine-keypad-setup)
3. [HTTP Protocol & Payload Schemas](#http-protocol--payload-schemas)
   * [Device Registration & Heartbeat (cmd: reg)](#device-registration--heartbeat-cmd-reg)
   * [Real-Time Attendance Punch (cmd: sendlog)](#real-time-attendance-punch-cmd-sendlog)
   * [Biometric User Sync (cmd: senduser)](#biometric-user-sync-cmd-senduser)
4. [Verification Mode & Direction Enums](#verification-mode--direction-enums)
5. [Multi-Language Code Implementations](#multi-language-code-implementations)
   * [Raw PHP / Any PHP Framework (Laravel/CodeIgniter/Symfony)](#raw-php--any-php-framework)
   * [Node.js (Express.js)](#nodejs-expressjs)
   * [Python (FastAPI / Flask)](#python-fastapi--flask)
   * [Go (Gin Framework)](#go-gin-framework)
6. [Universal Database Schema & Duplicate Protection](#universal-database-schema--duplicate-protection)
7. [Security & Heartbeat Optimization](#security--heartbeat-optimization)
8. [cURL Testing & Verification](#curl-testing--verification)

---

## Overview & Architecture

TIMMY AI05 / AiFace biometric machines communicate directly over standard **HTTP / HTTPS POST** requests using JSON payloads. This eliminates the need for complex background socket servers, proprietar[...] 

> Sample device serial number used in examples: **AXSC19024286** — replace with your device's SN. In the examples below we use the placeholder `<DEVICE_SN>` to avoid repeating the literal SN multiple times.

```
┌─────────────────────────────────┐      Direct HTTP POST (Port 80/443)     ┌─────────────────[...]
│   TIMMY AI05 Physical Device    │ ──────────────────────────────────────> │  Your Web Server / REST API       │
│   (Model: AiFace / AI05)        │                                         │  (Node / Laravel / Python / Go)   │
│   SN: <DEVICE_SN>               │ <────────────────────────────────────── │                                   │
└─────────────────────────────────┘        JSON Acknowledgement             └─────────────────[...]
```

---

## Physical Machine Keypad Setup

Configure the TIMMY AI05 keypad settings:

1. Press **`[M/OK]`** ➔ Navigate to **`Comm.`** ➔ **`Cloud Server Setting`**
2. **Server Mode:** Set to `HTTP` or `HTTP POST`
3. **Server Domain / IP:** Enter your server domain or IP *(e.g., `domain.com` or `103.84.21.81`)*
4. **Server Port:** `80` (for HTTP) or `443` (for HTTPS)
5. **Target Path:** set to your API path *(`/pub/api`)*

---

## HTTP Protocol & Payload Schemas

### Device Registration & Heartbeat (cmd: reg)

Sent by the machine upon boot and periodically to check server availability.

* **Request Payload (`Device -> Server`):**
  ```json
  {
    "cmd": "reg",
    "sn": "<DEVICE_SN>",
    "devinfo": {
      "modelname": "AiFace/TIMMY AI05",
      "firmware": "ai806_fp50h_v4.57",
      "useduser": 45,
      "usedface": 42,
      "usedfp": 12,
      "usedlog": 1250
    }
  }
  ```

* **Required JSON Response (`Server -> Device`):**
  ```json
  {
    "ret": "reg",
    "tryseconds": 30,
    "result": true,
    "cloudtime": "2026-09-02 13:00:00",
    "nosenduser": false,
    "nosendlog": false,
    "nosendimage": true
  }
  ```

---

### Real-Time Attendance Punch (cmd: sendlog)

Sent immediately when an employee scans their Face, Fingerprint, Password, or Card.

* **Request Payload (`Device -> Server`):**
  ```json
  {
    "cmd": "sendlog",
    "sn": "<DEVICE_SN>",
    "count": 1,
    "logindex": 104,
    "record": [
      {
        "enrollid": "101072",
        "name": "Tanzil",
        "time": "2026-09-02 09:56:46",
        "mode": 1,
        "inout": 0,
        "event": 0
      }
    ]
  }
  ```

* **Required JSON Response (`Server -> Device`):**
  ```json
  {
    "ret": "sendlog",
    "result": true,
    "count": 1,
    "logindex": 104,
    "mark": true
  }
  ```
  > **CRITICAL:** Returning `"mark": true` and `"result": true` notifies the hardware that the log was successfully recorded. If omitted, the device will re-send the log repeatedly.

---

### Biometric User Sync (cmd: senduser)

Sent when new user templates are added or updated on the machine.

* **Request Payload (`Device -> Server`):**
  ```json
  {
    "cmd": "senduser",
    "sn": "<DEVICE_SN>",
    "enrollid": 101072,
    "name": "Tanzil"
  }
  ```

* **Required JSON Response (`Server -> Device`):**
  ```json
  {
    "ret": "senduser",
    "result": true,
    "enrollid": 101072,
    "sn": "<DEVICE_SN>"
  }
  ```

---

## Verification Mode & Direction Enums

### Verification Mode (`mode`):
| Code | Verification Method |
| :---: | :--- |
| **`1`** | **Fingerprint** |
| **`2`** | **Password / PIN** |
| **`3`** | **RFID Card** |
| **`8`** | **Face Recognition** |

### Attendance Direction (`inout`):
| Code | Direction | Description |
| :---: | :--- | :--- |
| **`0`** | **Check-In** | Duty Start / Arrival |
| **`1`** | **Check-Out** | Duty End / Departure |
| **`2`** | **Break-Out** | Lunch / Break Start |
| **`3`** | **Break-In** | Lunch / Break Return |

---

## Multi-Language Code Implementations

### Raw PHP / Any PHP Framework

```php
<?php
// Set JSON response header
header('Content-Type: application/json; charset=utf-8');

// Read raw body input stream
$rawInput = file_get_contents('php://input');
$data = json_decode($rawInput, true);

if (!is_array($data)) {
    echo json_encode(['ret' => 'error', 'result' => false]);
    exit;
}

$cmd = $data['cmd'] ?? '';
$sn  = $data['sn'] ?? '<DEVICE_SN>';

// 1. Heartbeat / Handshake
if ($cmd === 'reg') {
    echo json_encode([
        'ret'        => 'reg',
        'tryseconds' => 30,
        'result'     => true,
        'cloudtime'  => date('Y-m-d H:i:s')
    ]);
    exit;
}

// 2. Attendance Punch Stream
if ($cmd === 'sendlog') {
    $records = isset($data['record']) ? $data['record'] : [];
    if (!is_array($records)) $records = [$records];

    foreach ($records as $rec) {
        $enrollId  = $rec['enrollid'] ?? null;
        $punchTime = $rec['time'] ?? null;
        $mode      = (int)($rec['mode'] ?? 1);
        $inout     = (int)($rec['inout'] ?? 0);

        if ($enrollId && $punchTime) {
            // TODO: Execute your database INSERT logic here
            // e.g., INSERT IGNORE INTO attendance_logs (enroll_id, punch_time, mode, inout, sn) VALUES (...)
        }
    }

    echo json_encode([
        'ret'      => 'sendlog',
        'result'   => true,
        'count'    => (int)($data['count'] ?? count($records)),
        'logindex' => (int)($data['logindex'] ?? 0),
        'mark'     => true
    ]);
    exit;
}

// Fallback
echo json_encode(['ret' => !empty($cmd) ? $cmd : 'ack', 'result' => true]);
```

---

### Node.js (Express.js)

```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.post('/api/attendance', (req, res) => {
    const { cmd, sn, count, logindex, record } = req.body;

    if (cmd === 'reg') {
        return res.json({
            ret: 'reg',
            tryseconds: 30,
            result: true,
            cloudtime: new Date().toISOString().replace('T', ' ').substring(0, 19)
        });
    }

    if (cmd === 'sendlog') {
        const records = Array.isArray(record) ? record : [record];
        records.forEach(rec => {
            if (rec && rec.enrollid && rec.time) {
                // TODO: Save to database (PostgreSQL/MySQL/MongoDB)
                console.log(`Punch from ${rec.enrollid} at ${rec.time} via Mode ${rec.mode}`);
            }
        });

        return res.json({
            ret: 'sendlog',
            result: true,
            count: count || records.length,
            logindex: logindex || 0,
            mark: true
        });
    }

    return res.json({ ret: cmd || 'ack', result: true });
});

app.listen(3000, () => console.log('TIMMY Receiver running on port 3000'));
```

---

### Python (FastAPI / Flask)

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.post("/api/attendance")
async def receive_attendance(request: Request):
    data = await request.json()
    cmd = data.get("cmd", "")
    sn = data.get("sn", "<DEVICE_SN>")

    if cmd == "reg":
        return {
            "ret": "reg",
            "tryseconds": 30,
            "result": True,
            "cloudtime": "2026-09-02 13:00:00"
        }

    if cmd == "sendlog":
        records = data.get("record", [])
        if not isinstance(records, list):
            records = [records]

        for rec in records:
            enrollid = rec.get("enrollid")
            punch_time = rec.get("time")
            mode = rec.get("mode", 1)
            inout = rec.get("inout", 0)
            # TODO: Save to DB

        return {
            "ret": "sendlog",
            "result": True,
            "count": data.get("count", len(records)),
            "logindex": data.get("logindex", 0),
            "mark": True
        }

    return {"ret": cmd or "ack", "result": True}
```

---

### Go (Gin Framework)

```go
package main

import (
	"net/http"
	"time"
	"github.com/gin-gonic/gin"
)

type Record struct {
	EnrollID string `json:"enrollid"`
	Time     string `json:"time"`
	Mode     int    `json:"mode"`
	InOut    int    `json:"inout"`
}

type TimmyPayload struct {
	Cmd      string   `json:"cmd"`
	SN       string   `json:"sn"`
	Count    int      `json:"count"`
	LogIndex int      `json:"logindex"`
	Record   []Record `json:"record"`
}

func main() {
	r := gin.Default()

	r.POST("/api/attendance", func(c *gin.Context) {
		var payload TimmyPayload
		if err := c.ShouldBindJSON(&payload); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"ret": "error", "result": false})
			return
		}

		if payload.Cmd == "reg" {
			c.JSON(http.StatusOK, gin.H{
				"ret":        "reg",
				"tryseconds": 30,
				"result":     true,
				"cloudtime":  time.Now().Format("2006-01-02 15:04:05"),
			})
			return
		}

		if payload.Cmd == "sendlog" {
			for _, rec := range payload.Record {
				if rec.EnrollID != "" && rec.Time != "" {
					// TODO: Save to DB
				}
			}

			c.JSON(http.StatusOK, gin.H{
				"ret":      "sendlog",
				"result":   true,
				"count":    payload.Count,
				"logindex": payload.LogIndex,
				"mark":     true,
			})
			return
		}

		c.JSON(http.StatusOK, gin.H{"ret": payload.Cmd, "result": true})
	})

	r.Run(":8080")
}
```

---

## Universal Database Schema & Duplicate Protection

To guarantee zero duplicate attendance entries during network retry attempts, define a **Composite Unique Key** on `(enroll_id, punch_time)`.

### Standard MySQL DDL:
```sql
CREATE TABLE IF NOT EXISTS `attendance_logs` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `device_sn` VARCHAR(64) NOT NULL DEFAULT '<DEVICE_SN>',
  `enroll_id` VARCHAR(64) NOT NULL,
  `punch_time` DATETIME NOT NULL,
  `verify_mode` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '1=Fingerprint, 2=Password, 3=Card, 8=Face',
  `inout_mode` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '0=In, 1=Out, 2=BreakOut, 3=BreakIn',
  `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY `idx_enroll_punch` (`enroll_id`, `punch_time`),
  INDEX `idx_search` (`enroll_id`, `punch_time`, `device_sn`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## Security & Heartbeat Optimization

1. **Heartbeat Efficiency:**  
   `cmd: reg` and `cmd: checklive` requests are handshakes. Return immediate JSON acknowledgements without making database queries to preserve database CPU cycles.
2. **Device SN Whitelisting:**  
   Validate `sn` against a list of approved Serial Numbers. Reject unauthorized devices with HTTP status `403`.
3. **Strict Input Casting:**  
   Sanitize and cast `enrollid` to string, `time` to datetime format, and `mode`/`inout` to integers to prevent SQL Injection and invalid data types.

---

## cURL Testing & Verification

Simulate a TIMMY AI05 attendance punch from terminal:

```bash
curl -X POST "http://localhost/api/attendance" \
  -H "Content-Type: application/json" \
  -d '{
    "cmd": "sendlog",
    "sn": "<DEVICE_SN>",
    "count": 1,
    "logindex": 100,
    "record": [
      {
        "enrollid": "101072",
        "name": "Tanzil",
        "time": "2026-09-02 13:00:00",
        "mode": 1,
        "inout": 0
      }
    ]
  }'
```

**Expected Response:**
```json
{
  "ret": "sendlog",
  "result": true,
  "count": 1,
  "logindex": 100,
  "mark": true
}
```
