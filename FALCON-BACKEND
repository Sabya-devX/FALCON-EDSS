import os
import uuid
import datetime
import requests
import asyncio
import random
from pathlib import Path
from typing import List, Dict, Any, Optional
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import FileResponse, HTMLResponse
from pydantic import BaseModel

# ==========================================
# 1. CORE APPLICATION & MIDDLEWARE
# ==========================================
app = FastAPI(title="FALCON EDSS // Mission Control Backend", version="2.5.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# ==========================================
# 2. IN-MEMORY STATE MANAGEMENT (Simulated DB)
# ==========================================
incident_ledger: List[Dict[str, Any]] = []

triage_state = {
    "unassigned": 4,
    "active": 12,
    "resolved": 8,
    "verified_beacons": 18
}

fleet_state = {
    "irbAvail": 8, "irbDep": 5,
    "alsAvail": 4, "alsDep": 2,
    "rat": 2400, "trk": 3
}

BASE_DIR = Path(__file__).resolve().parent

# ==========================================
# 3. WEBSOCKET CONNECTION MANAGER
# ==========================================
class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)

    async def broadcast(self, message: dict):
        for connection in list(self.active_connections):
            try:
                await connection.send_json(message)
            except Exception:
                pass

manager = ConnectionManager()


# ==========================================
# 3.5 AUTONOMOUS SCANNER & ENVIRONMENT STATE
# ==========================================
env_state = {
    "water_level_meters": 2.1,
    "danger_threshold": 4.5,
    "is_anomaly_injected": False
}

async def continuous_environmental_scanner():
    global triage_state
    while True:
        await asyncio.sleep(5) # Har 5 second me background scan
        
        # Normal time me paani 2.0 se 2.5 ke beech rahega
        if not env_state["is_anomaly_injected"]:
            env_state["water_level_meters"] = round(random.uniform(2.0, 2.5), 2)
            
        # Agar paani danger mark cross kar gaya
        if env_state["water_level_meters"] >= env_state["danger_threshold"]:
            incident_id = f"AUTO-SYS-{str(uuid.uuid4())[:6].upper()}"
            timestamp = datetime.datetime.now(datetime.timezone.utc).strftime("%H:%M:%S UTC")
            
            auto_record = {
                "incident_id": incident_id,
                "name": "FALCON_AI_NODE_01",
                "contact": "AUTONOMOUS_SENSOR",
                "priority": "CRITICAL",
                "hazard_type": "FLASH FLOOD (AUTO)",
                "souls": 0,
                "latitude": round(20.2961 + random.uniform(-0.02, 0.02), 4),
                "longitude": round(85.8245 + random.uniform(-0.02, 0.02), 4),
                "description": f"CRITICAL ANOMALY: Water spiked to {env_state['water_level_meters']}m. Auto-alert triggered.",
                "timestamp": timestamp
            }
            
            incident_ledger.append(auto_record)
            triage_state["verified_beacons"] += 1
            triage_state["unassigned"] += 1
            
            await manager.broadcast({
                "type": "NEW_SOS",
                "data": auto_record,
                "triage_state": triage_state
            })
            
            # Ek baar alert bhejne ke baad system wapas normal kardo
            env_state["is_anomaly_injected"] = False
            env_state["water_level_meters"] = 2.1

@app.on_event("startup")
async def start_background_scanner():
    asyncio.create_task(continuous_environmental_scanner())

# ==========================================
# 4. PYDANTIC SCHEMAS
# ==========================================
class DispatchPayload(BaseModel):
    sector: str
    asset_type: str
    count: int

# ==========================================
# 5. API ENDPOINTS & ZERO-STORAGE DATABASE VIEW
# ==========================================

@app.get("/")
async def serve_frontend():
    for file in os.listdir(BASE_DIR):
        if file.endswith(".html"):
            return FileResponse(BASE_DIR / file)
            
    return {
        "error": "No HTML file found in folder",
        "files_present": os.listdir(BASE_DIR)
    }

@app.get("/db", response_class=HTMLResponse)
async def view_database():
    rows = ""
    for inc in reversed(incident_ledger):
        inc_id = inc.get("incident_id", "N/A")
        caller_name = inc.get("name", "Anonymous")
        caller_contact = inc.get("contact", "N/A")
        priority = inc.get("priority", "CRITICAL")
        hazard = inc.get("hazard_type", "EMERGENCY")
        souls = inc.get("souls", 1)
        desc = inc.get("description", "Distress Beacon Broadcasted")
        lat = inc.get("latitude", 0.0)
        lon = inc.get("longitude", 0.0)
        t_stamp = inc.get("timestamp", "Just now")

        # Dynamic Priority Colors
        pri_badge = "#ef4444" if priority == "CRITICAL" else ("#f97316" if priority == "HIGH" else "#eab308")

        rows += f"""
        <tr style='border-bottom: 1px solid #1e293b;'>
            <td style='padding: 14px; color: #38bdf8; font-family: monospace; font-weight: 600;'>{inc_id}</td>
            <td style='padding: 14px; color: #f8fafc;'><b>{caller_name}</b><br><span style="color: #94a3b8; font-size: 11px;">{caller_contact}</span></td>
            <td style='padding: 14px;'><span style='background: {pri_badge}22; color: {pri_badge}; border: 1px solid {pri_badge}55; padding: 3px 8px; border-radius: 4px; font-size: 11px; font-weight: 700;'>{priority}</span></td>
            <td style='padding: 14px; color: #f8fafc; font-weight: 600;'>{hazard}</td>
            <td style='padding: 14px; color: #38bdf8; font-family: monospace; font-weight: bold;'>{souls}</td>
            <td style='padding: 14px; color: #94a3b8; font-family: monospace; font-size: 12px;'>[{lat:.4f}, {lon:.4f}]</td>
            <td style='padding: 14px; color: #cbd5e1; font-size: 13px;'>{desc}</td>
            <td style='padding: 14px; color: #64748b; font-size: 12px; font-family: monospace;'>{t_stamp}</td>
            <td style='padding: 14px;'><span style='background: #14532d; color: #4ade80; padding: 3px 8px; border-radius: 4px; font-size: 11px; font-weight: bold;'>VERIFIED</span></td>
        </tr>
        """
        if not rows:
            rows = "<tr><td colspan='8' style='padding: 30px; text-align: center; color: #64748b; font-family: monospace;'>No SOS distress signals logged yet. Transmit from FALCON Command.</td></tr>"

    html_content = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <title>FALCON EDSS // Mission Control Database</title>
        <meta http-equiv="refresh" content="5">
        <style>
            body {{ background-color: #07090e; color: #e2e8f0; font-family: system-ui, sans-serif; padding: 30px; margin: 0; }}
            .container {{ max-width: 1200px; margin: auto; }}
            .card {{ background: #0e1420; border-radius: 12px; border: 1px solid rgba(255,255,255,0.08); overflow: hidden; box-shadow: 0 10px 30px rgba(0,0,0,0.6); }}
            table {{ width: 100%; border-collapse: collapse; text-align: left; }}
            th {{ background: #0b0f17; padding: 14px; color: #94a3b8; font-size: 11px; text-transform: uppercase; letter-spacing: 0.1em; border-bottom: 1px solid #1e293b; }}
            .header {{ display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }}
            .badge {{ background: rgba(6, 182, 212, 0.15); color: #06b6d4; border: 1px solid rgba(6, 182, 212, 0.4); padding: 5px 12px; border-radius: 20px; font-size: 11px; font-weight: 600; font-family: monospace; }}
            .demo-btn {{ background: #ef4444; color: white; border: 1px solid #dc2626; padding: 6px 14px; border-radius: 20px; font-size: 11px; font-weight: bold; cursor: pointer; margin-left: 15px; font-family: monospace; box-shadow: 0 0 10px rgba(239,68,68,0.2); }}
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <div>
                    <h2 style="margin: 0; color: #f8fafc; letter-spacing: 0.05em;">⚡ FALCON EDSS Live Incident Ledger</h2>
                    <p style="margin: 5px 0 0 0; color: #94a3b8; font-size: 13px;">High-Speed In-Memory State Ledger (Zero Disk Overhead)</p>
                </div>
                <div style="display: flex; align-items: center;">
                    <span class="badge">● LIVE TELEMETRY SYNC</span>
                    <button class="demo-btn" onclick="triggerFakeDisaster()">🚨 DEMO: SIMULATE DISASTER</button>
                </div>
            </div>
            <div class="card">
                <table>
                    <thead>
                        <tr>
                            <th>Incident ID</th>
                            <th>Caller Info</th>
                            <th>Priority</th>
                            <th>Hazard</th>
                            <th>Souls</th>
                            <th>Coordinates</th>
                            <th>Details</th>
                            <th>UTC Timestamp</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody>
                        {rows}
                    </tbody>
                </table>
            </div>
        </div>
        <script>
            async function triggerFakeDisaster() {{
                try {{
                    const response = await fetch('/api/sensor/spike', {{ method: 'POST' }});
                    if(response.ok) {{
                        alert("⚠️ Fake disaster injected! The background scanner will detect it in a few seconds.");
                    }}
                }} catch(err) {{
                    alert("Error! Check if server is running.");
                }}
            }}
        </script>
    </body>
    </html>
    """
    return HTMLResponse(content=html_content)

@app.post("/api/sos")
async def register_sos(payload: dict):
    global triage_state

    incident_id = f"SOS-{str(uuid.uuid4())[:8].upper()}"
    timestamp = datetime.datetime.now(datetime.timezone.utc).strftime("%H:%M:%S UTC")

    record = dict(payload)
    record["incident_id"] = incident_id
    record["timestamp"] = timestamp

    # Update State
    incident_ledger.append(record)
    triage_state["verified_beacons"] += 1
    triage_state["unassigned"] += 1

    # Real-time WebSocket Push
    await manager.broadcast({
        "type": "NEW_SOS",
        "data": record,
        "triage_state": triage_state
    })

    return {
        "status": "success",
        "incident_id": incident_id,
        "total_active": triage_state["verified_beacons"],
        "data": record
    }
@app.post("/api/sensor/spike")
async def inject_fake_disaster():
    # Ye fake button hit hote hi paani ka level badha dega
    env_state["is_anomaly_injected"] = True
    env_state["water_level_meters"] = 5.2 
    return {"status": "success", "message": "Environmental data manipulated. Scanner will catch it in 5 seconds."}

@app.get("/api/incidents")
async def get_incidents():
    return {
        "triage_state": triage_state,
        "recent_incidents": incident_ledger[-10:]
    }

@app.get("/api/telemetry/live")
async def fetch_live_telemetry(lat: float, lon: float, hazard: str = "flood"):
    url = f"https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=precipitation,rain,surface_pressure,wind_speed_10m,wind_gusts_10m,temperature_2m,relative_humidity_2m&hourly=cape&timezone=auto"
    try:
        response = requests.get(url, timeout=5.0)
        response.raise_for_status()
        data = response.json()
        cur = data.get("current", {})
        hourly = data.get("hourly", {})
        
        return {
            "status": "success",
            "source": "OPEN_METEO_API",
            "data": {
                "precip": cur.get("precipitation", 0),
                "wind": cur.get("wind_speed_10m", 0),
                "gust": cur.get("wind_gusts_10m", 0),
                "pressure": cur.get("surface_pressure", 1013),
                "temp": cur.get("temperature_2m", 25),
                "humidity": cur.get("relative_humidity_2m", 80),
                "cape": hourly.get("cape", [0])[0] if hourly.get("cape") else 1200,
                "soil": 65
            }
        }
    except Exception:
        return {
            "status": "fallback",
            "source": "FALCON_HEURISTIC_ENGINE",
            "data": {
                "precip": 64.2, "wind": 45.0, "gust": 65.0, 
                "pressure": 982.0, "temp": 28.5, "humidity": 92.0, 
                "cape": 1850, "soil": 88
            }
        }

@app.post("/api/dispatch")
async def allocate_assets(payload: DispatchPayload):
    global fleet_state
    
    if payload.asset_type == "IRB" and fleet_state["irbAvail"] >= payload.count:
        fleet_state["irbAvail"] -= payload.count
        fleet_state["irbDep"] += payload.count
    elif payload.asset_type == "ALS" and fleet_state["alsAvail"] >= payload.count:
        fleet_state["alsAvail"] -= payload.count
        fleet_state["alsDep"] += payload.count
    else:
        raise HTTPException(status_code=400, detail="Insufficient assets available.")

    await manager.broadcast({
        "type": "FLEET_UPDATE",
        "data": fleet_state,
        "sector": payload.sector
    })

    return {"status": "dispatched", "fleet_state": fleet_state}

@app.get("/api/cdm/schema")
async def get_cdm_schema():
    return {
        "schema_version": "FALCON_CDM_v2.5",
        "description": "Dynamic multi-hazard interoperability standard.",
        "nodes": ["METEO", "HYDRO", "GEO", "SOS_CROWDSOURCED"]
    }

# ==========================================
# 6. WEBSOCKET ROUTE
# ==========================================
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            await websocket.receive_text()
    except WebSocketDisconnect:
        manager.disconnect(websocket)
