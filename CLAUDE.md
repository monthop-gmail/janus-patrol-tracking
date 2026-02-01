# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ระบบติดตามการลาดตระเวน Real-time — ทหารส่ง Live Video (WebRTC) + GPS จากมือถือ มายังศูนย์บัญชาการ ผ่าน Janus Gateway + Node.js API + PostgreSQL

## Architecture

```
[ทหาร A/B/C มือถือ] ──WebRTC publish──▶ [Janus VideoRoom]
        │                                       │
        │ GPS (Socket.IO)               subscribe│
        ▼                                       ▼
   [Node.js API] ──save──▶ [PostgreSQL]    [Center Dashboard]
        │                                  ├── แผนที่ Leaflet + markers
        │ Socket.IO broadcast             └── Live Video popup
        └──────────────────────────────────────▶
```

- **Janus Gateway** (host network) — WebRTC media routing via VideoRoom plugin, 1 shared room (ID 1234), soldiers = publishers, center = subscriber
- **Node.js API** — Express + Socket.IO รับ GPS, จัดการ soldier sessions, บันทึกลง PostgreSQL
- **PostgreSQL** — เก็บข้อมูลทหาร + GPS history ย้อนหลัง
- **Nginx** — reverse proxy /api, /socket.io, /janus, /ws + serve static HTML
- **Coturn** — TURN server สำหรับ NAT traversal

## Commands

```bash
# Start all services
docker compose up -d --build

# Restart single service
docker compose restart janus

# View logs
docker compose logs -f api
docker compose logs -f janus

# Test API
curl -s http://localhost/api/soldiers | python3 -m json.tool
curl -X POST http://localhost/api/soldiers -H "Content-Type: application/json" -d '{"callsign":"Alpha-1"}'
curl http://localhost/api/soldiers/1/track

# Test Janus
curl -X POST http://localhost:8088/janus -H "Content-Type: application/json" -d '{"janus":"info","transaction":"test"}'
```

## Key Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | 5 services: janus (host network), api, postgres, nginx, coturn |
| `api/server.js` | REST API + Socket.IO server (GPS relay, soldier sessions) |
| `api/db.js` | PostgreSQL pool + schema auto-init (soldiers, gps_logs tables) |
| `docs/center.html` | ศูนย์บัญชาการ — Leaflet map + Janus VideoRoom subscriber |
| `docs/soldier.html` | หน้าทหาร — camera + GPS + Janus VideoRoom publisher |
| `janus/janus.jcfg` | Janus main config (NAT, TURN credentials, ICE port range) |
| `janus/janus.transport.websockets.jcfg` | WebSocket on :8188 (ห้ามใส่ ws_acl) |
| `nginx/nginx.conf` | Proxy rules: /api, /socket.io → api:3000; /janus, /ws → host Janus |

## Important Notes

- **Janus ใช้ `network_mode: host`** — จำเป็นเพื่อให้ ICE candidates เป็น host IP ตรงๆ ถ้าเปลี่ยนเป็น bridge network ต้องตั้ง `nat_1_1_mapping` ใน janus.jcfg
- **ห้ามใส่ `ws_acl`** ใน janus.transport.websockets.jcfg — format เป็น prefix-based (`"127.,192."`) ไม่ใช่ CIDR ถ้าใส่ผิด Janus จะบล็อกทุก connection
- **TURN credentials** ต้องตรงกัน 3 ที่: `coturn/turnserver.conf`, `janus/janus.jcfg`, และ iceServers ใน soldier.html + center.html (default: janus/januspass)
- **Nginx proxy** ชี้ไป `host.docker.internal` สำหรับ Janus เพราะ Janus ใช้ host network
- **PostgreSQL healthcheck** — api service จะ wait จนกว่า postgres healthy ก่อน start

## Database Schema

- `soldiers` — id, callsign (unique), name, janus_room, janus_feed, is_online
- `gps_logs` — id, soldier_id (FK), lat, lng, accuracy, recorded_at (indexed)

## Ports

| Port | Service |
|------|---------|
| 80 | Nginx (web + proxy) |
| 8088 | Janus HTTP API (host) |
| 8188 | Janus WebSocket (host) |
| 3478 | Coturn TURN |
| 5004/udp | Janus RTP |
| 20000-20100/udp | Janus ICE candidates |
