# IntelliTwin — Full Implementation Guide

AI-Powered Digital Twin Platform for Smart Building Monitoring, Predictive Maintenance, and Intelligent Energy Optimization

This guide takes your synopsis and turns it into a buildable plan: tools to install, exact folder structure, the order to build things in, and working starter code for every layer (Simulator → Backend → AI Engine → Frontend → 3D Twin).

---

## 1. Tools & Software You Need to Install

| Tool | Purpose | Install |
|---|---|---|
| **Node.js (LTS, v20+)** | Backend + frontend tooling | nodejs.org |
| **Python 3.10+** | AI/ML engine | python.org |
| **MongoDB Community Server** (or MongoDB Atlas free tier) | Database | mongodb.com/try/download/community — Atlas is easier, no local install |
| **VS Code** | Editor | code.visualstudio.com |
| **Git + GitHub account** | Version control, 4-person collaboration | git-scm.com |
| **Postman** | Testing REST APIs | postman.com |
| **MongoDB Compass** | GUI to inspect your DB | comes with MongoDB |
| **Docker Desktop** (optional, for deployment later) | Containerization | docker.com |

VS Code extensions worth adding: ESLint, Prettier, MongoDB for VS Code, Python, Pylance, Thunder Client (Postman alternative inside VS Code).

Python packages (install with `pip install -r requirements.txt` — file given in Section 7):
`scikit-learn`, `pandas`, `numpy`, `flask` (or `fastapi`+`uvicorn`), `joblib`, `python-dotenv`, `pymongo`

Node packages (backend): `express`, `mongoose`, `jsonwebtoken`, `bcryptjs`, `cors`, `dotenv`, `node-cron`, `axios`, `pdfkit`

Node packages (frontend): `react`, `react-router-dom`, `axios`, `three`, `@react-three/fiber`, `@react-three/drei`, `chart.js`, `react-chartjs-2`, `tailwindcss`

---

## 2. Architecture Recap

```
[Virtual IoT Sensor Simulator (Python/Node script)]
              │  POSTs synthetic readings every N seconds
              ▼
[Node.js + Express REST API]  ───────────────►  [MongoDB]
              │  raw + processed sensor data          ▲
              ▼                                        │
[Python AI Engine: Flask/FastAPI microservice]  ───────┘
   - Random Forest → predictive maintenance
   - Regression model → energy forecasting
   - Isolation Forest → anomaly detection
   - Rule engine → recommendations
              │  predictions written back via REST
              ▼
[React + Tailwind + Chart.js dashboard]
[Three.js 3D Digital Twin — room colour = health/risk]
[What-if module, alerts, JWT role-based auth, PDF reports]
```

Key design decision: the **AI Engine is a separate Python microservice**, not code glued into Node. Node calls it over HTTP (or you run predictions as a scheduled batch job that writes results into MongoDB, which the Node API then just serves). This matches your synopsis's "modular REST-based architecture" claim and is much easier for 4 people to split work on.

---

## 3. Full Repository / Folder Structure

Create one GitHub repo with this layout:

```
intellitwin/
├── README.md
├── .gitignore
│
├── simulator/                       # Virtual IoT Sensor Simulator
│   ├── simulate.py
│   ├── rooms_config.json            # defines building/floor/room layout
│   ├── requirements.txt
│   └── .env
│
├── backend/                         # Node.js + Express + MongoDB
│   ├── package.json
│   ├── .env
│   ├── server.js
│   ├── config/
│   │   └── db.js
│   ├── models/
│   │   ├── User.js
│   │   ├── Room.js
│   │   ├── SensorReading.js
│   │   ├── Prediction.js
│   │   └── Alert.js
│   ├── routes/
│   │   ├── authRoutes.js
│   │   ├── sensorRoutes.js
│   │   ├── predictionRoutes.js
│   │   ├── roomRoutes.js
│   │   ├── alertRoutes.js
│   │   ├── whatIfRoutes.js
│   │   └── reportRoutes.js
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── sensorController.js
│   │   ├── predictionController.js
│   │   ├── alertController.js
│   │   └── reportController.js
│   ├── middleware/
│   │   ├── authMiddleware.js        # JWT verify
│   │   └── roleMiddleware.js        # Admin / Manager / Technician
│   └── utils/
│       └── generatePDF.js
│
├── ai-engine/                       # Python AI microservice
│   ├── app.py                       # Flask/FastAPI entrypoint
│   ├── requirements.txt
│   ├── .env
│   ├── data/
│   │   └── training_data.csv
│   ├── models/
│   │   ├── train_maintenance_model.py
│   │   ├── train_energy_forecast.py
│   │   ├── train_anomaly_model.py
│   │   └── saved/
│   │       ├── maintenance_rf.joblib
│   │       ├── energy_forecast.joblib
│   │       └── anomaly_iforest.joblib
│   ├── services/
│   │   ├── predictive_maintenance.py
│   │   ├── energy_forecasting.py
│   │   ├── anomaly_detection.py
│   │   └── recommendation_engine.py
│   └── db/
│       └── mongo_client.py
│
├── frontend/                        # React + Tailwind + Chart.js + Three.js
│   ├── package.json
│   ├── tailwind.config.js
│   ├── .env
│   ├── public/
│   └── src/
│       ├── main.jsx
│       ├── App.jsx
│       ├── api/
│       │   └── axiosClient.js
│       ├── context/
│       │   └── AuthContext.jsx
│       ├── pages/
│       │   ├── Login.jsx
│       │   ├── AdminDashboard.jsx
│       │   ├── ManagerDashboard.jsx
│       │   ├── TechnicianDashboard.jsx
│       │   ├── DigitalTwinView.jsx
│       │   ├── WhatIfSimulator.jsx
│       │   └── Reports.jsx
│       ├── components/
│       │   ├── Navbar.jsx
│       │   ├── RoomCard.jsx
│       │   ├── SensorChart.jsx
│       │   ├── AlertList.jsx
│       │   └── ProtectedRoute.jsx
│       └── three/
│           ├── BuildingScene.jsx
│           ├── RoomMesh.jsx
│           └── sceneUtils.js
│
└── docs/
    ├── architecture-diagram.png
    ├── er-diagram.png
    └── api-documentation.md
```

**Suggested split across your 4 team members:**
- Person A → `simulator/` + `backend/` core (auth, models, DB)
- Person B → `ai-engine/` (all 3 ML models + recommendation engine)
- Person C → `frontend/` dashboard pages, charts, auth flow
- Person D → `frontend/three/` 3D Digital Twin + what-if module + PDF reports

---

## 4. Build Order (Phases) — Do It in This Sequence

1. **Phase 0 — Setup (Week 1):** Init Git repo, create folder skeleton above, everyone clones it, set up MongoDB Atlas cluster, agree on the sensor data schema (fields below).
2. **Phase 1 — Simulator:** Build the Python simulator that generates believable per-room data and POSTs it to the backend on a timer.
3. **Phase 2 — Backend core:** Express server, MongoDB models, sensor ingestion route, JWT auth + roles.
4. **Phase 3 — AI Engine v1:** Generate/collect a synthetic training dataset, train Random Forest (maintenance), regression (energy), Isolation Forest (anomaly). Wrap them in a Flask/FastAPI service with `/predict` endpoints.
5. **Phase 4 — Backend ↔ AI integration:** A scheduled job (node-cron) periodically calls the AI engine with recent sensor batches and stores predictions/alerts back in MongoDB.
6. **Phase 5 — Frontend dashboard:** Login, role-based routing, live charts (Chart.js) pulling from backend REST APIs.
7. **Phase 6 — 3D Digital Twin:** Three.js scene, one mesh per room, colour driven by live health/risk score from the predictions API.
8. **Phase 7 — What-if module + alerts + PDF reports:** UI for hypothetical inputs → sent to AI engine → results shown without touching real data; alert list component; PDFKit report generation.
9. **Phase 8 — Polish + testing + optional Docker deployment.**

---

## 5. Sensor Data Schema (agree on this first — everything depends on it)

```json
{
  "roomId": "B1-F2-R204",
  "buildingId": "B1",
  "floor": 2,
  "timestamp": "2026-08-29T10:15:00Z",
  "temperature": 24.6,
  "humidity": 51.2,
  "occupancy": 6,
  "power": 3.42,
  "smoke": 0.02,
  "motion": true,
  "equipment": {
    "id": "AC-204",
    "runtimeHours": 1820,
    "ageMonths": 14
  }
}
```

---

## 6. Phase 1 — Virtual IoT Sensor Simulator (Python)

`simulator/requirements.txt`
```
requests
numpy
python-dotenv
schedule
```

`simulator/rooms_config.json`
```json
[
  { "roomId": "B1-F1-R101", "buildingId": "B1", "floor": 1, "baseOccupancy": 8, "equipmentId": "AC-101", "ageMonths": 6 },
  { "roomId": "B1-F1-R102", "buildingId": "B1", "floor": 1, "baseOccupancy": 4, "equipmentId": "AC-102", "ageMonths": 30 },
  { "roomId": "B1-F2-R201", "buildingId": "B1", "floor": 2, "baseOccupancy": 12, "equipmentId": "AC-201", "ageMonths": 48 }
]
```

`simulator/simulate.py`
```python
import json, random, time, math, requests, os
from datetime import datetime, timezone
from dotenv import load_dotenv

load_dotenv()
BACKEND_URL = os.getenv("BACKEND_URL", "http://localhost:5000/api/sensors/ingest")
INTERVAL_SECONDS = 10

with open("rooms_config.json") as f:
    ROOMS = json.load(f)

# runtime hours accumulate as the "simulation clock" advances
runtime_tracker = {r["roomId"]: random.randint(500, 3000) for r in ROOMS}

def hour_of_day():
    return datetime.now().hour

def weather_factor():
    # crude day/season proxy: hotter mid-afternoon
    h = hour_of_day()
    return 1.0 + 0.4 * math.sin((h - 6) / 24 * 2 * math.pi)

def occupancy_for(room):
    h = hour_of_day()
    # business hours 9-18 -> occupancy near base, otherwise near 0
    if 9 <= h <= 18:
        return max(0, int(random.gauss(room["baseOccupancy"], 2)))
    return random.randint(0, 1)

def generate_reading(room):
    occ = occupancy_for(room)
    wf = weather_factor()
    base_temp = 22 + 2.5 * wf - 0.15 * occ
    temperature = round(base_temp + random.gauss(0, 0.4), 2)
    humidity = round(45 + 5 * wf + random.gauss(0, 2), 2)

    # power draw scales with occupancy, equipment age (older = less efficient), and temp delta
    age_penalty = 1 + (room["ageMonths"] / 100)
    power = round((0.15 * occ + 0.08 * abs(temperature - 22)) * age_penalty + random.uniform(0.1, 0.3), 3)

    runtime_tracker[room["roomId"]] += INTERVAL_SECONDS / 3600
    # inject occasional anomaly / fault drift for older equipment
    fault_drift = 0
    if room["ageMonths"] > 36 and random.random() < 0.02:
        fault_drift = random.uniform(2, 6)  # sudden temp/power spike

    return {
        "roomId": room["roomId"],
        "buildingId": room["buildingId"],
        "floor": room["floor"],
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "temperature": round(temperature + fault_drift, 2),
        "humidity": humidity,
        "occupancy": occ,
        "power": round(power + fault_drift * 0.3, 3),
        "smoke": round(max(0, random.gauss(0.01, 0.005) + (0.05 if fault_drift else 0)), 4),
        "motion": occ > 0,
        "equipment": {
            "id": room["equipmentId"],
            "runtimeHours": round(runtime_tracker[room["roomId"]], 1),
            "ageMonths": room["ageMonths"]
        }
    }

def run():
    while True:
        for room in ROOMS:
            reading = generate_reading(room)
            try:
                r = requests.post(BACKEND_URL, json=reading, timeout=5)
                print(f"[{reading['roomId']}] sent -> {r.status_code}")
            except requests.exceptions.RequestException as e:
                print(f"[ERROR] could not reach backend: {e}")
        time.sleep(INTERVAL_SECONDS)

if __name__ == "__main__":
    run()
```

Run it with: `python simulate.py` (after backend is up).

---

## 7. Phase 2 — Backend (Node.js + Express + MongoDB)

`backend/package.json` (key deps)
```json
{
  "name": "intellitwin-backend",
  "type": "commonjs",
  "scripts": { "dev": "nodemon server.js", "start": "node server.js" },
  "dependencies": {
    "express": "^4.19.2",
    "mongoose": "^8.5.0",
    "jsonwebtoken": "^9.0.2",
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "node-cron": "^3.0.3",
    "axios": "^1.7.2",
    "pdfkit": "^0.15.0"
  },
  "devDependencies": { "nodemon": "^3.1.0" }
}
```

`backend/.env`
```
PORT=5000
MONGO_URI=mongodb+srv://<user>:<password>@cluster0.mongodb.net/intellitwin
JWT_SECRET=replace_with_a_long_random_string
AI_ENGINE_URL=http://localhost:8000
```

`backend/config/db.js`
```javascript
const mongoose = require("mongoose");

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log("MongoDB connected");
  } catch (err) {
    console.error("MongoDB connection error:", err.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

`backend/models/SensorReading.js`
```javascript
const mongoose = require("mongoose");

const SensorReadingSchema = new mongoose.Schema({
  roomId: { type: String, required: true, index: true },
  buildingId: String,
  floor: Number,
  timestamp: { type: Date, default: Date.now, index: true },
  temperature: Number,
  humidity: Number,
  occupancy: Number,
  power: Number,
  smoke: Number,
  motion: Boolean,
  equipment: {
    id: String,
    runtimeHours: Number,
    ageMonths: Number
  }
});

module.exports = mongoose.model("SensorReading", SensorReadingSchema);
```

`backend/models/User.js`
```javascript
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");

const UserSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true, required: true },
  password: { type: String, required: true },
  role: { type: String, enum: ["Admin", "Manager", "Technician"], default: "Technician" }
});

UserSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

UserSchema.methods.comparePassword = function (candidate) {
  return bcrypt.compare(candidate, this.password);
};

module.exports = mongoose.model("User", UserSchema);
```

`backend/models/Prediction.js`
```javascript
const mongoose = require("mongoose");

const PredictionSchema = new mongoose.Schema({
  roomId: { type: String, index: true },
  timestamp: { type: Date, default: Date.now },
  failureProbability: Number,      // Random Forest output, 0-1
  forecastedEnergyKwh: Number,     // regression output
  isAnomaly: Boolean,              // Isolation Forest output
  anomalyScore: Number,
  riskLevel: { type: String, enum: ["Low", "Medium", "High"], default: "Low" },
  recommendation: String
});

module.exports = mongoose.model("Prediction", PredictionSchema);
```

`backend/middleware/authMiddleware.js`
```javascript
const jwt = require("jsonwebtoken");

module.exports = (req, res, next) => {
  const header = req.headers.authorization;
  if (!header || !header.startsWith("Bearer ")) {
    return res.status(401).json({ message: "No token provided" });
  }
  try {
    const decoded = jwt.verify(header.split(" ")[1], process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch {
    res.status(401).json({ message: "Invalid or expired token" });
  }
};
```

`backend/middleware/roleMiddleware.js`
```javascript
module.exports = (...allowedRoles) => (req, res, next) => {
  if (!allowedRoles.includes(req.user.role)) {
    return res.status(403).json({ message: "Forbidden: insufficient role" });
  }
  next();
};
```

`backend/controllers/sensorController.js`
```javascript
const SensorReading = require("../models/SensorReading");

exports.ingest = async (req, res) => {
  try {
    const reading = await SensorReading.create(req.body);
    res.status(201).json(reading);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
};

exports.getLatestByRoom = async (req, res) => {
  const { roomId } = req.params;
  const readings = await SensorReading.find({ roomId }).sort({ timestamp: -1 }).limit(50);
  res.json(readings);
};

exports.getAllLatest = async (req, res) => {
  // latest reading per room, useful for the 3D dashboard overview
  const results = await SensorReading.aggregate([
    { $sort: { timestamp: -1 } },
    { $group: { _id: "$roomId", latest: { $first: "$$ROOT" } } }
  ]);
  res.json(results.map(r => r.latest));
};
```

`backend/routes/sensorRoutes.js`
```javascript
const express = require("express");
const router = express.Router();
const ctrl = require("../controllers/sensorController");

router.post("/ingest", ctrl.ingest);                 // called by the simulator
router.get("/room/:roomId", ctrl.getLatestByRoom);
router.get("/latest", ctrl.getAllLatest);

module.exports = router;
```

`backend/server.js`
```javascript
require("dotenv").config();
const express = require("express");
const cors = require("cors");
const cron = require("node-cron");
const connectDB = require("./config/db");
const axios = require("axios");

const SensorReading = require("./models/SensorReading");
const Prediction = require("./models/Prediction");

const authRoutes = require("./routes/authRoutes");
const sensorRoutes = require("./routes/sensorRoutes");
const predictionRoutes = require("./routes/predictionRoutes");

const app = express();
app.use(cors());
app.use(express.json());

connectDB();

app.use("/api/auth", authRoutes);
app.use("/api/sensors", sensorRoutes);
app.use("/api/predictions", predictionRoutes);

// Every 60s: pull latest readings, ask the AI engine for predictions, store them
cron.schedule("*/60 * * * * *", async () => {
  try {
    const latest = await SensorReading.aggregate([
      { $sort: { timestamp: -1 } },
      { $group: { _id: "$roomId", latest: { $first: "$$ROOT" } } }
    ]);
    for (const { latest: reading } of latest) {
      const { data } = await axios.post(`${process.env.AI_ENGINE_URL}/predict`, reading);
      await Prediction.create({ roomId: reading.roomId, ...data });
    }
  } catch (err) {
    console.error("Prediction cron failed:", err.message);
  }
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Backend running on port ${PORT}`));
```

(`authRoutes.js`/`authController.js` and `predictionRoutes.js` follow the same pattern as `sensorController.js` — register/login issuing JWTs with `{ id, role }`, and a simple `GET /api/predictions/:roomId` reading from the `Prediction` collection.)

---

## 8. Phase 3 — AI Engine (Python, Flask microservice)

`ai-engine/requirements.txt`
```
flask
flask-cors
scikit-learn
pandas
numpy
joblib
python-dotenv
```

`ai-engine/models/train_maintenance_model.py`
```python
"""
Trains a Random Forest classifier to estimate failure probability.
Since you have no real historical failure data yet, generate a labeled
synthetic dataset with a rule that ties failure risk to temperature,
power, runtime hours, and equipment age -- this is standard practice
for a prototype and is explainable in your review/viva.
"""
import numpy as np
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
import joblib

np.random.seed(42)
N = 5000

df = pd.DataFrame({
    "temperature": np.random.normal(26, 4, N),
    "power": np.random.normal(2.5, 1.2, N).clip(0),
    "runtimeHours": np.random.uniform(100, 5000, N),
    "ageMonths": np.random.uniform(1, 60, N),
})

# synthetic ground truth rule: risk rises with age, runtime, and temp extremes
risk_score = (
    0.015 * df["ageMonths"] +
    0.0006 * df["runtimeHours"] +
    0.08 * (df["temperature"] - 26).abs() +
    0.2 * (df["power"] > 4.5).astype(int) +
    np.random.normal(0, 0.5, N)
)
df["failure"] = (risk_score > risk_score.quantile(0.8)).astype(int)

X = df[["temperature", "power", "runtimeHours", "ageMonths"]]
y = df["failure"]

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
model = RandomForestClassifier(n_estimators=200, max_depth=8, random_state=42)
model.fit(X_train, y_train)

print(classification_report(y_test, model.predict(X_test)))
joblib.dump(model, "saved/maintenance_rf.joblib")
print("Saved model to saved/maintenance_rf.joblib")
```

`ai-engine/models/train_anomaly_model.py`
```python
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest
import joblib

np.random.seed(1)
N = 5000
df = pd.DataFrame({
    "temperature": np.random.normal(24, 1.5, N),
    "humidity": np.random.normal(50, 5, N),
    "power": np.random.normal(2.2, 0.7, N).clip(0),
    "smoke": np.random.normal(0.01, 0.005, N).clip(0),
})

model = IsolationForest(n_estimators=200, contamination=0.03, random_state=1)
model.fit(df)
joblib.dump(model, "saved/anomaly_iforest.joblib")
print("Saved model to saved/anomaly_iforest.joblib")
```

`ai-engine/models/train_energy_forecast.py`
```python
import numpy as np
import pandas as pd
from sklearn.linear_model import Ridge
import joblib

np.random.seed(7)
N = 5000
df = pd.DataFrame({
    "occupancy": np.random.randint(0, 20, N),
    "temperature": np.random.normal(26, 4, N),
    "hourOfDay": np.random.randint(0, 24, N),
})
df["power"] = (
    0.15 * df["occupancy"] +
    0.08 * (df["temperature"] - 22).clip(lower=0) +
    0.02 * np.sin(df["hourOfDay"] / 24 * 2 * np.pi) +
    np.random.normal(0, 0.2, N)
).clip(lower=0)

X = df[["occupancy", "temperature", "hourOfDay"]]
y = df["power"]
model = Ridge(alpha=1.0)
model.fit(X, y)
joblib.dump(model, "saved/energy_forecast.joblib")
print("Saved model to saved/energy_forecast.joblib")
```

Run all three once to produce the `.joblib` files:
```
cd ai-engine/models
python train_maintenance_model.py
python train_anomaly_model.py
python train_energy_forecast.py
```

`ai-engine/services/recommendation_engine.py`
```python
def generate_recommendation(failure_prob, is_anomaly, forecast_kwh):
    if failure_prob > 0.7:
        return "High failure risk: schedule technician inspection within 48 hours."
    if is_anomaly:
        return "Abnormal sensor pattern detected: verify equipment and sensor wiring."
    if forecast_kwh > 4.0:
        return "Elevated energy forecast: consider adjusting HVAC setpoint or occupancy scheduling."
    return "No action needed. Systems operating within normal parameters."

def risk_level(failure_prob):
    if failure_prob > 0.7:
        return "High"
    if failure_prob > 0.4:
        return "Medium"
    return "Low"
```

`ai-engine/app.py`
```python
from flask import Flask, request, jsonify
from flask_cors import CORS
import joblib
import pandas as pd
from datetime import datetime
from services.recommendation_engine import generate_recommendation, risk_level

app = Flask(__name__)
CORS(app)

maintenance_model = joblib.load("models/saved/maintenance_rf.joblib")
anomaly_model = joblib.load("models/saved/anomaly_iforest.joblib")
energy_model = joblib.load("models/saved/energy_forecast.joblib")

@app.route("/predict", methods=["POST"])
def predict():
    data = request.get_json()

    maint_features = pd.DataFrame([{
        "temperature": data["temperature"],
        "power": data["power"],
        "runtimeHours": data["equipment"]["runtimeHours"],
        "ageMonths": data["equipment"]["ageMonths"],
    }])
    failure_prob = float(maintenance_model.predict_proba(maint_features)[0][1])

    anomaly_features = pd.DataFrame([{
        "temperature": data["temperature"],
        "humidity": data["humidity"],
        "power": data["power"],
        "smoke": data["smoke"],
    }])
    anomaly_pred = anomaly_model.predict(anomaly_features)[0]   # -1 = anomaly, 1 = normal
    anomaly_score = float(anomaly_model.decision_function(anomaly_features)[0])
    is_anomaly = bool(anomaly_pred == -1)

    energy_features = pd.DataFrame([{
        "occupancy": data["occupancy"],
        "temperature": data["temperature"],
        "hourOfDay": datetime.now().hour,
    }])
    forecast_kwh = float(energy_model.predict(energy_features)[0])

    return jsonify({
        "failureProbability": round(failure_prob, 3),
        "forecastedEnergyKwh": round(forecast_kwh, 3),
        "isAnomaly": is_anomaly,
        "anomalyScore": round(anomaly_score, 3),
        "riskLevel": risk_level(failure_prob),
        "recommendation": generate_recommendation(failure_prob, is_anomaly, forecast_kwh)
    })

@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000, debug=True)
```

Run with: `python app.py` (from inside `ai-engine/`).

---

## 9. Phase 5 — Frontend (React + Tailwind + Chart.js)

`frontend/src/api/axiosClient.js`
```javascript
import axios from "axios";

const axiosClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || "http://localhost:5000/api",
});

axiosClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default axiosClient;
```

`frontend/src/components/SensorChart.jsx`
```jsx
import { Line } from "react-chartjs-2";
import {
  Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Tooltip, Legend,
} from "chart.js";

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Tooltip, Legend);

export default function SensorChart({ readings, field, label, color = "#3b82f6" }) {
  const data = {
    labels: readings.map(r => new Date(r.timestamp).toLocaleTimeString()),
    datasets: [{
      label,
      data: readings.map(r => r[field]),
      borderColor: color,
      tension: 0.3,
    }],
  };
  return <Line data={data} options={{ responsive: true, animation: false }} />;
}
```

`frontend/src/components/RoomCard.jsx`
```jsx
export default function RoomCard({ room, prediction }) {
  const riskColor = {
    High: "bg-red-100 text-red-700 border-red-300",
    Medium: "bg-yellow-100 text-yellow-700 border-yellow-300",
    Low: "bg-green-100 text-green-700 border-green-300",
  }[prediction?.riskLevel || "Low"];

  return (
    <div className={`rounded-xl border p-4 shadow-sm ${riskColor}`}>
      <h3 className="font-semibold">{room.roomId}</h3>
      <p className="text-sm">Temp: {room.temperature}°C | Power: {room.power} kW</p>
      <p className="text-sm">Risk: {prediction?.riskLevel ?? "—"}</p>
      {prediction?.recommendation && <p className="text-xs mt-1 italic">{prediction.recommendation}</p>}
    </div>
  );
}
```

---

## 10. Phase 6 — 3D Digital Twin (Three.js via @react-three/fiber)

`frontend/src/three/BuildingScene.jsx`
```jsx
import { Canvas } from "@react-three/fiber";
import { OrbitControls } from "@react-three/drei";
import RoomMesh from "./RoomMesh";

export default function BuildingScene({ rooms, predictions }) {
  const predictionFor = (roomId) => predictions.find(p => p.roomId === roomId);

  return (
    <Canvas camera={{ position: [10, 10, 10], fov: 50 }} style={{ height: "600px" }}>
      <ambientLight intensity={0.6} />
      <directionalLight position={[5, 10, 5]} intensity={0.8} />
      <OrbitControls />
      {rooms.map((room, i) => (
        <RoomMesh
          key={room.roomId}
          position={[(i % 4) * 3, Math.floor(i / 4) * 3, 0]}
          room={room}
          prediction={predictionFor(room.roomId)}
        />
      ))}
    </Canvas>
  );
}
```

`frontend/src/three/RoomMesh.jsx`
```jsx
import { useState } from "react";
import { Text } from "@react-three/drei";

const riskColorMap = { High: "#ef4444", Medium: "#f59e0b", Low: "#22c55e" };

export default function RoomMesh({ position, room, prediction }) {
  const [hovered, setHovered] = useState(false);
  const color = riskColorMap[prediction?.riskLevel] || "#94a3b8";

  return (
    <group position={position}>
      <mesh
        onPointerOver={() => setHovered(true)}
        onPointerOut={() => setHovered(false)}
        scale={hovered ? 1.1 : 1}
      >
        <boxGeometry args={[2, 2, 2]} />
        <meshStandardMaterial color={color} opacity={0.85} transparent />
      </mesh>
      <Text position={[0, 1.4, 0]} fontSize={0.3} color="black">
        {room.roomId}
      </Text>
    </group>
  );
}
```

Each room is a box; colour reflects live risk level pulled from `/api/predictions`. Clicking/hovering can pop out a detail panel with the live chart from Section 9.

---

## 11. Phase 7 — What-If Module (concept)

- Frontend form: sliders for occupancy, target temperature, hour of day.
- On submit, POST that hypothetical reading straight to the AI engine's `/predict` endpoint (bypassing MongoDB — it's hypothetical, not real).
- Display the returned failure probability / energy forecast / recommendation without writing anything to the database.
- This directly matches the "what-if scenario simulation module" advantage you list in the synopsis.

**PDF reporting** (`backend/utils/generatePDF.js`) — use `pdfkit` to stream a report of predictions/alerts over a selected date range; expose it via `GET /api/reports/:buildingId?from=...&to=...` returning `application/pdf`.

---

## 12. Testing & Integration Checklist

- [ ] Simulator successfully POSTs readings, visible in MongoDB Compass
- [ ] `/api/sensors/latest` returns one doc per room
- [ ] AI engine `/health` returns 200, `/predict` returns valid JSON for a sample reading
- [ ] node-cron job populates the `Prediction` collection every 60s
- [ ] Login issues JWT; protected routes reject requests without a valid token
- [ ] Role middleware blocks Technician from Admin-only routes
- [ ] Dashboard charts update on refresh/poll (simple `setInterval` fetch every 10–15s is fine for a prototype)
- [ ] 3D room colours change when you manually insert an anomalous reading
- [ ] What-if form returns different results for different inputs without touching stored data
- [ ] PDF report downloads and opens correctly

---

## 13. Optional: Deployment

- Backend → Render or Railway (free tiers support Node + env vars)
- AI engine → Render (Python web service) or a Docker container
- MongoDB → Atlas free cluster (already assumed above)
- Frontend → Vercel or Netlify
- If you containerize, a `docker-compose.yml` at the repo root with three services (`backend`, `ai-engine`, `frontend`) plus `MONGO_URI` pointing at Atlas is enough — you don't need to containerize MongoDB itself.

---

## 14. What to Show in Your Review/Viva

Given your literature survey's stated gaps (no integrated software platform, no what-if module, hardware dependence), make sure your demo explicitly hits: (1) live simulator running with no physical hardware, (2) predictive maintenance flagging a room before failure, (3) energy forecast changing with occupancy, (4) anomaly detected on an injected spike, (5) 3D twin colour-coding rooms live, (6) what-if scenario producing different output than the current real state, (7) role-based views differing between Admin/Manager/Technician, (8) a generated PDF report. That maps every "Advantage" bullet in your synopsis to a visible demo moment.
