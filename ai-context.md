# AI Context: Rhombus Camera Player Implementation Guide

**Purpose:** Complete guide for AI agents to implement Rhombus security camera video players in any project.

**Last Updated:** 2025-11-19

---

## Overview

### What This Implements
A web-based video player streaming live footage from Rhombus security cameras using MPEG-DASH protocol via DashJS.

### Key Components
1. **Rhombus API** - RESTful API for camera management and media access
2. **Federated Session Tokens** - Short-lived tokens for secure media streaming
3. **MPEG-DASH** - Adaptive streaming protocol
4. **DashJS** - JavaScript DASH player library
5. **Proxy Server** - Backend protects API credentials

### Critical Security Rule
**NEVER call Rhombus API directly from frontend.** Always use a proxy server to protect API tokens.

---

## Architecture

```
Browser (Frontend)
  ↓ HTTP (localhost:8000)
Proxy Server (Backend)
  ↓ HTTPS + API Token
Rhombus API
  ↓
Media Server (DASH Stream)
```

### Data Flow
1. Frontend → Proxy: Request federated token
2. Proxy → Rhombus: Authenticate with API token, get federated token
3. Proxy → Frontend: Return federated token
4. Frontend → Proxy: Request media URIs
5. Proxy → Rhombus: Get streaming URLs
6. Proxy → Frontend: Return media URIs
7. Frontend → Media Server: Stream video (federated token in query params)

---

## Prerequisites

### Required Credentials
1. **Rhombus API Token** - From dashboard → Settings → API (keep secret, backend only)
2. **Camera UUID** - From dashboard → Camera details (safe for frontend)

### Required Libraries
- **Frontend:** DashJS 4.7.2+ (`https://cdn.dashjs.org/v4.7.2/dash.all.min.js`)
- **Backend:** express, cors, dotenv (Node.js) or equivalent

---

## Step-by-Step Implementation

### Step 1: Create Proxy Server

**File:** `proxy-server.js`

```javascript
const express = require('express');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors({ origin: ['http://localhost:8000'], credentials: true }));
app.use(express.json());

const RHOMBUS_API_URL = 'https://api2.rhombussystems.com/api';
const RHOMBUS_API_TOKEN = process.env.RHOMBUS_API_TOKEN;

// Generate federated token
app.post('/api/federated-token', async (req, res) => {
    const { durationSec } = req.body;
    const response = await fetch(`${RHOMBUS_API_URL}/org/generateFederatedSessionToken`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'x-auth-scheme': 'api-token',
            'x-auth-apikey': RHOMBUS_API_TOKEN
        },
        body: JSON.stringify({ durationSec })
    });
    res.json(await response.json());
});

// Get media URIs
app.post('/api/media-uris', async (req, res) => {
    const { cameraUuid } = req.body;
    const response = await fetch(`${RHOMBUS_API_URL}/camera/getMediaUris`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'x-auth-scheme': 'api-token',
            'x-auth-apikey': RHOMBUS_API_TOKEN
        },
        body: JSON.stringify({ cameraUuid })
    });
    res.json(await response.json());
});

app.listen(3000, () => console.log('Proxy running on http://localhost:3000'));
```

### Step 2: Create Environment File

**File:** `.env` (add to .gitignore!)

```bash
RHOMBUS_API_TOKEN=your-api-token-here
PORT=3000
```

### Step 3: Create Frontend

**File:** `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Rhombus Camera Stream</title>
    <script src="https://cdn.dashjs.org/v4.7.2/dash.all.min.js"></script>
    <script>
        const CAMERA_UUID = "your-camera-uuid-here";
        const BASE_URL = "http://localhost:3000";

        async function getFederatedToken(durationSec) {
            const res = await fetch(`${BASE_URL}/api/federated-token`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ durationSec })
            });
            const data = await res.json();
            return data.federatedSessionToken;
        }

        async function getMediaUri(cameraUuid) {
            const res = await fetch(`${BASE_URL}/api/media-uris`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ cameraUuid })
            });
            const data = await res.json();
            return data.wanLiveMpdUri;
        }

        async function streamVideo(cameraUuid) {
            // Get federated token (24 hours)
            const token = await getFederatedToken(86400);
            
            // Get media URI
            const mediaUri = await getMediaUri(cameraUuid);
            
            // Create player
            const player = dashjs.MediaPlayer().create();
            
            // Add token to all media requests
            player.extend("RequestModifier", function() {
                return {
                    modifyRequestURL: (url) => 
                        url + `?x-auth-scheme=federated-token&x-auth-ft=${token}`
                }
            }, true);
            
            // Configure for live streaming
            player.updateSettings({
                streaming: {
                    delay: { liveDelayFragmentCount: 4 },
                    liveCatchup: { enabled: true, mode: "fast" },
                    abr: { autoSwitchBitrate: { video: false, audio: false } }
                }
            });
            
            // Start playback
            player.initialize(document.querySelector("#video"), mediaUri, true);
        }

        // Start on page load
        streamVideo(CAMERA_UUID);
    </script>
</head>
<body>
    <h1>Rhombus Camera Stream</h1>
    <video id="video" width="640" height="360" muted></video>
</body>
</html>
```

### Step 4: Install and Run

```bash
# Install dependencies
npm install express cors dotenv

# Start proxy server
node proxy-server.js

# Start HTML server (separate terminal)
python3 -m http.server 8000

# Open browser
open http://localhost:8000/index.html
```

---

## API Reference

### Rhombus API Endpoints

#### Generate Federated Token
```
POST https://api2.rhombussystems.com/api/org/generateFederatedSessionToken

Headers:
  Content-Type: application/json
  x-auth-scheme: api-token
  x-auth-apikey: YOUR_API_TOKEN

Body:
  { "durationSec": 86400 }

Response:
  { "federatedSessionToken": "5HjBaPSaSEiVtyEv9yHR1A" }
```

#### Get Media URIs
```
POST https://api2.rhombussystems.com/api/camera/getMediaUris

Headers:
  Content-Type: application/json
  x-auth-scheme: api-token
  x-auth-apikey: YOUR_API_TOKEN

Body:
  { "cameraUuid": "CAMERA_UUID" }

Response:
  {
    "wanLiveMpdUri": "https://.../file.mpd",
    "wanLiveM3u8Uri": "https://.../file.m3u8",
    "wanLiveH264Uri": "wss://.../ws",
    ...
  }
```

### Media Stream Authentication

Append to all media requests:
```
?x-auth-scheme=federated-token&x-auth-ft=YOUR_FEDERATED_TOKEN
```

---

## DashJS Configuration

### Essential Settings for Live Cameras

```javascript
player.updateSettings({
    streaming: {
        scheduling: {
            scheduleWhilePaused: false,  // Save bandwidth
            defaultTimeout: 5000
        },
        delay: {
            liveDelayFragmentCount: 4    // 8-12 second latency
        },
        liveCatchup: {
            maxDrift: 10,                 // Max seconds behind live
            playbackRate: { min: -0.5, max: 1.0 },
            playbackBufferMin: 15,
            enabled: true,
            mode: "fast"
        },
        buffer: {
            bufferTimeAtTopQuality: 30
        },
        abr: {
            autoSwitchBitrate: {
                video: false,             // Single quality stream
                audio: false
            }
        }
    }
});
```

### Latency Tuning

**Low Latency (2-4s):**
```javascript
delay: { liveDelayFragmentCount: 1 }
```

**Balanced (8-12s):**
```javascript
delay: { liveDelayFragmentCount: 4 }
```

**Stable/Poor Network (15-20s):**
```javascript
delay: { liveDelayFragmentCount: 8 }
```

---

## Troubleshooting

### CORS Error
```
Access blocked by CORS policy
```
**Fix:** Add origin to proxy CORS config
```javascript
app.use(cors({ origin: ['http://localhost:8000'] }));
```

### 401 Unauthorized
```
Rhombus API error: 401
```
**Fix:** Check API token, verify headers include `x-auth-apikey`

### Video Not Playing
**Causes:**
1. Federated token not appended → Check RequestModifier
2. Wrong URI field → Use `wanLiveMpdUri` not `wanLiveM3u8Uri`
3. DashJS not loaded → Verify CDN script tag
4. Camera offline → Check Rhombus dashboard

### Token Expired (after 24 hours)
**Fix:** Implement token refresh
```javascript
setInterval(async () => {
    const newToken = await getFederatedToken(86400);
    // Reinitialize player with new token
}, 23 * 60 * 60 * 1000); // 23 hours
```

---

## Security Checklist

- [ ] API token in environment variable (not code)
- [ ] .env file in .gitignore
- [ ] Proxy server handles all Rhombus API calls
- [ ] Frontend never sees API token
- [ ] CORS configured with specific origins
- [ ] HTTPS enabled in production
- [ ] Input validation on all endpoints
- [ ] Federated tokens used for media streaming

---

## Quick Reference

### Implementation Order
1. Create proxy server with API token
2. Create .env file (add to .gitignore)
3. Create HTML with DashJS
4. Request federated token from proxy
5. Request media URI from proxy
6. Initialize DashJS with RequestModifier
7. Start playback

### Key Files
- `proxy-server.js` - Backend (has API token)
- `.env` - Secrets (gitignored)
- `index.html` - Frontend (no secrets)
- `package.json` - Dependencies

### Essential Endpoints
- Proxy: `POST /api/federated-token`
- Proxy: `POST /api/media-uris`
- Rhombus: `POST /org/generateFederatedSessionToken`
- Rhombus: `POST /camera/getMediaUris`

---

## Resources

- **Rhombus API Docs:** https://docs.rhombussystems.com/reference
- **DashJS Wiki:** https://github.com/Dash-Industry-Forum/dash.js/wiki
- **Support:** support@rhombus.com
