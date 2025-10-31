# Virtual-Car-Sensor


*Streamlit dashboard ui:

mport streamlit as st
import pandas as pd
import requests
import time
import json
from datetime import datetime

# === CONFIG ===
st.set_page_config(page_title="Virtual Sensor System", layout="wide")
WORQHAT_TRIGGER_URL = "YOUR_WORQHAT_TRIGGER_URL_HERE"  # ← REPLACE THIS
WORQHAT_DB_QUERY_URL = "YOUR_WORQHAT_QUERY_URL_HERE"   # ← Optional (or use CSV)

# === TITLE ===
st.title("Virtual Sensor System - LIVE")
st.markdown("*AI-Powered Car Health Monitoring* | Beeps | Email | Real-Time Logs")

# === SIDEBAR CONTROLS ===
with st.sidebar:
    st.header("Controls")
    if st.button("START DEMO (10 Rows)", type="primary"):
        with st.spinner("Running AI workflow..."):
            try:
                response = requests.post(WORQHAT_TRIGGER_URL)
                if response.status_code == 200:
                    st.success("Demo Complete! Check logs below.")
                else:
                    st.error(f"Error: {response.text}")
            except Exception as e:
                st.error(f"Failed to connect: {e}")

    st.markdown("---")
    st.info("*Beeps* on CRITICAL alerts\n\n*Emails* sent to driver")

# === LIVE LOGS TABLE ===
st.header("Live Sensor Logs")
log_placeholder = st.empty()

# Simulate live update (poll every 3 sec)
for _ in range(20):  # Run 20 times = ~1 min
    try:
        # OPTION 1: Use WorqHat Query Data node (recommended)
        # logs = requests.get(WORQHAT_DB_QUERY_URL).json()

        # OPTION 2: Use local CSV (fallback for demo)
        df = pd.read_csv("car_sensor_data.csv").head(10)
        df["oil_pressure"] = [55 - (t-90)*0.3 if t > 90 else 55 for t in df["engine_temp"]]
        df["noise_index"] = round(df["rpm"]/1000 + df["engine_temp"]/50, 1)
        df["alert_type"] = df["engine_temp"].apply(lambda x: "CRITICAL" if x > 120 else "WARNING" if x > 100 else "NORMAL")
        df["alert_message"] = df.apply(lambda row: 
            "STOP NOW: Engine damage!" if row["alert_type"] == "CRITICAL" 
            else "Caution: High temp" if row["alert_type"] == "WARNING" 
            else "", axis=1)

        # Show only last 10
        log_placeholder.dataframe(df.tail(10), use_container_width=True)

        # === RED ALERT POPUP ===
        latest = df.iloc[-1]
        if latest["alert_type"] == "CRITICAL":
            st.error(f"CRITICAL: {latest['alert_message']} | Temp: {latest['engine_temp']}°C")
            # Optional: Play beep in browser
            st.components.v1.html("""
            <audio autoplay>
              <source src="data:audio/wav;base64,UklGRigAAABXQVZFZm10IBIAAAABAAEARKwAAIhYAQACABAAZGF0YQQAAAD+/+==...” type="audio/wav">
            </audio>
            """, height=0)
        elif latest["alert_type"] == "WARNING":
            st.warning(f"WARNING: {latest['alert_message']}")

    except Exception as e:
        log_placeholder.write("Waiting for data...")

    time.sleep(3)
