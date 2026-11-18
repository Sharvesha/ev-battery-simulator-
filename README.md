# EV Battery Temperature Monitoring Simulator

A simple, physics-based thermal simulation of a 48-cell EV battery pack (4 modules x 12 cells, 21700 Li-ion).
Backend: Python, FastAPI, WebSockets. Frontend: React, Vite, Tailwind CSS, Recharts, Lucide.

## Project structure

```
ev-battery-simulator/
├── backend/
│   ├── requirements.txt
│   ├── thermal_model.py      # physics, cell matrix, BMS state machine
│   └── main.py               # FastAPI app, REST controls, 1 Hz WebSocket broadcaster
└── frontend/
    ├── package.json
    ├── vite.config.js        # proxies /api and /ws to the backend on :8000
    ├── tailwind.config.js
    ├── postcss.config.js
    ├── index.html
    └── src/
        ├── main.jsx
        ├── index.css
        ├── App.jsx
        ├── hooks/useTelemetry.js
        ├── lib/ (api.js, colors.js, csv.js)
        └── components/ (CellHeatmap, TelemetryCharts, ControlPanel, EventLog)
```

## Setup and run

You need Python 3.10+ and Node 18+. Use two terminals.

**Terminal 1: backend**

```bash
cd ev-battery-simulator/backend
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

**Terminal 2: frontend**

```bash
cd ev-battery-simulator/frontend
npm install
npm run dev
```

Open http://localhost:5173.

## Verify the connection

```bash
curl http://localhost:8000/api/health
# {"status":"ok","sim_time_s":42,"ws_clients":1,"bms_state":"NORMAL"}
```

`ws_clients` is 1 when the browser is connected. The header badge on the page should read "Live".
You can also watch the raw stream (`pip install websockets` first):

```bash
python -m websockets ws://localhost:8000/ws
```

Interactive API docs: http://localhost:8000/docs

## Try these scenarios

1. **Warning and derating:** Thermal mode Eco, Drive cycle Track, Ambient 35 °C, speed 5x.
2. **Pump failure:** Drive cycle Track, press "Simulate coolant pump failure". Temperatures climb through Warning to Derating.
3. **Thermal runaway:** press "Inject cell #14 thermal runaway". About 35 s later the BMS goes Critical, opens the contactors and logs the event. Then press "Clear all faults" and "Close contactors" once the pack cools below 45 °C.
4. **Sensor dropout:** drop out the runaway cell's thermistor first and the BMS never sees the hotspot.
5. **Fast charge:** DC 350 kW in Eco mode at 30 °C. Charge current tapers above 80 % SoC.

## REST endpoints

| Method | Path | Body |
|---|---|---|
| POST | /api/profile | `{"profile": "idle / city / highway / track / fast_charge_150 / fast_charge_350"}` |
| POST | /api/ambient | `{"ambient_c": -10..50}` |
| POST | /api/cooling | `{"state": "auto / manual / failure", "manual_pump_pct": 0..100, "thermal_mode": "eco / balanced / performance"}` (all optional) |
| POST | /api/fault/runaway | `{"cell_id": 1..48}` |
| POST | /api/fault/sensor | `{"cell_id": 1..48}` |
| POST | /api/fault/clear | none |
| POST | /api/contactor/close | none |
| POST | /api/speed | `{"speed": 1, 5 or 10}` |
| POST | /api/reset | none |
| GET | /api/health, /api/state | none |

## How the model works (kept simple)

- Heat per cell: `Q_gen = I² · R(T, SoC) + entropic term`, cooling `Q_cool = h·A·(T_cell − T_coolant)` scaled by pump duty,
  plus conduction between neighbouring cells and a small loss to ambient.
- The coolant loop has its own thermal mass, a radiator and a chiller. Chiller power depends on the Eco, Balanced or Max cool mode.
- BMS states with 2 °C hysteresis: Normal below 40 °C, Warning 40 to 50 °C (pump boosted),
  Derating 50 to 60 °C (Imax scaled down), Critical above 60 °C (contactors open, latched until cleared).
- Currents are scaled down (each simulated cell stands in for a small slice of a real pack) so the heating is visible within minutes.
  The 150 kW and 350 kW profiles map to 15 A and 30 A per cell. Use the 5x and 10x speed buttons for quicker demos.
- The BMS only uses working sensors, so a thermistor dropout hides that cell from protection logic.
- Alarm sound is off by default because browsers need a click first. Use the "Alarm sound" button in the header.
