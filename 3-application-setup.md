# Step 3: Application Setup

This guide covers the setup of the `adsb2tak` Python application. This includes creating a virtual environment, installing the necessary TLS certificates for secure communication, and downloading the aircraft metadata database used for data enrichment.

---

## 3.1 Create the Application Environment

We will create a dedicated directory and a Python virtual environment to isolate the application's dependencies from the main system.

1.  **Create the Directory Structure:**
    ```bash
    mkdir -p ~/adsb2tak/certs ~/adsb2tak/database

    cd ~/adsb2tak
    ```

2.  **Create the Python Virtual Environment:**
    ```bash
    python3 -m venv .venv
    ```

3.  **Activate the Environment:**
    <br> You must activate the environment in your terminal session before installing packages or running the script.
    ```bash
    source .venv/bin/activate
    ```
    *Your command prompt should now be prefixed with `(.venv)`.*

4.  **Verify the Python Version:**
    <br> Confirm that you are using the Python interpreter from within the virtual environment.
    ```bash
    which python
    ```
    *Expected output: `/home/takadmin/adsb2tak/.venv/bin/python` (or similar, depending on your username).*

The final directory layout for the application will be:

/home/takadmin/adsb2tak/
<br>├── adsb2tak.py
<br>├── test_marker.py
<br>├── .venv/
<br>├── certs/
<br>│ ├── adsb2tak.pem
<br>│ ├── adsb2tak.key
<br>│ ├── adsb2tak-service.key
<br>│ └── adsb2tak-trusted.pem
<br>└── database/
<br>└── aircraft.csv.gz


---

## 3.2 Install the Client Certificate

For secure communication, the application uses a client TLS certificate to authenticate with the TAK Server.

1.  **Issue the Certificate:**
    <br>Using your existing TAK Certificate Authority (CA), issue a dedicated **client certificate** named `adsb2tak`.

2.  **Transfer and Organize Certificate Files:**
    <br>Transfer the three required files to your Raspberry Pi and place them in the `~/adsb2tak/certs/` directory. The required files are:
    *   `adsb2tak.pem`: The client certificate and its chain.
    *   `adsb2tak.key`: The encrypted private key for the certificate.
    *   `adsb2tak-trusted.pem`: The trusted CA chain for verifying the server.

3.  **Secure the Private Key:**
    <br>Set the file permissions on the private key so that only the owner can read it. This is a critical security step.
    ```bash
    chmod 600 ~/adsb2tak/certs/adsb2tak.key
    ```

---

## 3.3 Validate Mutual TLS Connection

Before proceeding, it is crucial to test that the entire TLS chain is working correctly using `openssl`. This command validates the network path, server-side firewall rules, the server's certificate, and the client certificate you just installed.

Execute the following command from your `tak-sns01` terminal:
```bash
openssl s_client \
  -connect takserver-01:8089 \
  -servername takserver-01 \
  -cert ~/adsb2tak/certs/adsb2tak.pem \
  -key ~/adsb2tak/certs/adsb2tak.key \
  -CAfile ~/adsb2tak/certs/adsb2tak-trusted.pem \
  -verify_hostname takserver-01
```

>⚠︎ CRITICAL
><br>A successful test will display a large amount of certificate information and must end with the line: `Verify return code: 0 (ok)`. If you get any other result, resolve the TLS or networking issue before proceeding.

---

## 3.4 Create the Service-Only Private Key
<br>The `adsb2tak.key` file is encrypted and requires a passphrase, which prevents the application from starting automatically. We will now create a decrypted version of this key for unattended use by the system service.

1.  Navigate to the Certs Directory:
    ```bash
    cd ~/adsb2tak/certs
    ```
2.  Create the Decrypted Key:
    <br>You will be prompted for the passphrase of the original `adsb2tak.key` one last time.
    ```bash
    openssl pkey -in adsb2tak.key -out adsb2tak-service.key
    ```
3.  Secure the New Service Key:
    <br>This unencrypted key is sensitive and must be protected.
    ```bash
    chmod 600 adsb2tak-service.key
    ```
    > The Python application will be configured to use this `adsb2tak-service.key` file.
---
## 3.5 ADSB2TAK Python Application
<br> The working application performs six core tasks: continuously read aircraft data from `aircraft.json`, filter for aircraft with valid and recently updated positions, convert aircraft information into Cursor-on-Target (CoT) events, securely transmit CoT messages to TAK Server over a persistent TLS connection, automatically reconnect if the connection is lost, and refresh aircraft positions approximately every two seconds to provide near-real-time tracking in WebTAK. The application also maintains consistent aircraft identifiers, preventing duplicate markers, and assigns expiration timestamps to CoT events to manage aircraft visibility when updates are no longer received.

Create the `adsb2tak.py`file:
```bash
sudo nano ~/adsb2tak/adsb2tak.py
```
Copy and paste this output into `adsb2tak.py`.

```bash
#!/usr/bin/env python3

import json
import socket
import ssl
import time
from pathlib import Path
from datetime import datetime, timedelta, timezone
import xml.etree.ElementTree as ET


# ============================================================
# ADSB2TAK CONFIGURATION
# ============================================================

AIRCRAFT_FILE = Path("/run/dump1090-fa/aircraft.json")

TAK_HOST = "takserver-01"
TAK_PORT = 8089

CERT_FILE = Path.home() / "adsb2tak/certs/adsb2tak.pem"
KEY_FILE = Path.home() / "adsb2tak/certs/adsb2tak-service.key"
CA_FILE = Path.home() / "adsb2tak/certs/adsb2tak-trusted.pem"

# Read and transmit aircraft every 2 seconds
SEND_INTERVAL = 2

# Only transmit aircraft with recently received positions
MAX_POSITION_AGE = 10

# Current CoT stale interval
STALE_MINUTES = 20

# Retry connection after failure
RECONNECT_INTERVAL = 10


# ============================================================
# TIME FORMATTING
# ============================================================

def cot_time(dt):
    return dt.strftime("%Y-%m-%dT%H:%M:%S.%fZ")


# ============================================================
# CONVERT AIRCRAFT DATA TO CoT
# ============================================================

def aircraft_to_cot(plane):

    lat = plane.get("lat")
    lon = plane.get("lon")

    # Aircraft must have valid coordinates
    if lat is None or lon is None:
        return None

    # Ignore aircraft without fresh position data
    seen = plane.get("seen", 999)

    if not isinstance(seen, (int, float)):
        return None

    if seen > MAX_POSITION_AGE:
        return None

    # Aircraft ICAO address
    icao = str(plane.get("hex") or "").upper()

    if not icao:
        return None

    # Aircraft callsign
    callsign = str(plane.get("flight") or "").strip()

    if not callsign:
        callsign = icao

    # Altitude: feet to meters
    altitude_ft = plane.get("alt_baro", 0)

    if isinstance(altitude_ft, (int, float)):
        altitude_m = altitude_ft * 0.3048
    else:
        altitude_m = 0

    # Ground speed: knots to meters per second
    speed_knots = plane.get("gs", 0)

    if isinstance(speed_knots, (int, float)):
        speed_mps = speed_knots * 0.514444
    else:
        speed_mps = 0

    # Aircraft heading
    track = plane.get("track", 0)

    if not isinstance(track, (int, float)):
        track = 0

    # Current timestamp and expiration
    now = datetime.now(timezone.utc)

    stale = now + timedelta(minutes=STALE_MINUTES)

    # Create CoT event
    event = ET.Element(
        "event",
        {
            "version": "2.0",
            "uid": f"ADSB-{icao}",
            "type": "a-u-A",
            "how": "m-g",
            "time": cot_time(now),
            "start": cot_time(now),
            "stale": cot_time(stale),
        },
    )

    # Aircraft location
    ET.SubElement(
        event,
        "point",
        {
            "lat": str(lat),
            "lon": str(lon),
            "hae": str(round(altitude_m, 1)),
            "ce": "50",
            "le": "50",
        },
    )

    # CoT details
    detail = ET.SubElement(event, "detail")

    ET.SubElement(
        detail,
        "contact",
        {
            "callsign": callsign,
        },
    )

    ET.SubElement(
        detail,
        "track",
        {
            "speed": str(round(speed_mps, 1)),
            "course": str(track),
        },
    )

    ET.SubElement(detail, "remarks").text = (
        f"ADS-B aircraft ICAO {icao}; "
        f"altitude {altitude_ft} ft; "
        f"speed {speed_knots} kt"
    )

    return ET.tostring(event, encoding="utf-8")


# ============================================================
# CONNECT TO TAK SERVER USING TLS
# ============================================================

def connect_to_tak():

    context = ssl.create_default_context(
        ssl.Purpose.SERVER_AUTH,
        cafile=str(CA_FILE),
    )

    context.load_cert_chain(
        certfile=str(CERT_FILE),
        keyfile=str(KEY_FILE),
    )

    raw_socket = socket.create_connection(
        (TAK_HOST, TAK_PORT),
        timeout=10,
    )

    try:
        tls_socket = context.wrap_socket(
            raw_socket,
            server_hostname=TAK_HOST,
        )

        return tls_socket

    except Exception:
        raw_socket.close()
        raise


# ============================================================
# MAIN ADSB2TAK SERVICE
# ============================================================

def main():

    tak_socket = None
    last_idle_message = 0

    print("========================================")
    print("ADSB2TAK SERVICE STARTING")
    print("========================================")

    print(f"TAK destination: {TAK_HOST}:{TAK_PORT}")
    print(f"ADS-B source: {AIRCRAFT_FILE}")
    print(f"Update interval: {SEND_INTERVAL} seconds")
    print(f"Maximum position age: {MAX_POSITION_AGE} seconds")
    print(f"CoT stale interval: {STALE_MINUTES} minutes")

    while True:

        try:

            # ------------------------------------------------
            # CONNECT OR RECONNECT TO TAK SERVER
            # ------------------------------------------------

            if tak_socket is None:

                try:

                    print(
                        f"Connecting to TAK Server "
                        f"{TAK_HOST}:{TAK_PORT}..."
                    )

                    tak_socket = connect_to_tak()

                    print("TLS connection established.")

                except Exception as e:

                    print(f"TAK unavailable: {e}")

                    print(
                        f"Retrying in "
                        f"{RECONNECT_INTERVAL} seconds..."
                    )

                    time.sleep(RECONNECT_INTERVAL)

                    continue

            # ------------------------------------------------
            # READ CURRENT ADS-B DATA
            #
            # IMPORTANT:
            # This block must remain OUTSIDE
            # the "if tak_socket is None" condition.
            # ------------------------------------------------

            try:

                with AIRCRAFT_FILE.open("r") as f:
                    data = json.load(f)

            except (
                FileNotFoundError,
                json.JSONDecodeError,
                OSError,
            ) as e:

                print(f"ADS-B data unavailable: {e}")

                time.sleep(SEND_INTERVAL)

                continue

            sent = 0

            # ------------------------------------------------
            # PROCESS AND TRANSMIT AIRCRAFT
            # ------------------------------------------------

            for plane in data.get("aircraft", []):

                cot = aircraft_to_cot(plane)

                if cot is None:
                    continue

                try:

                    tak_socket.sendall(cot + b"\n")

                    icao = str(
                        plane.get("hex") or "UNKNOWN"
                    ).upper()

                    callsign = str(
                        plane.get("flight") or ""
                    ).strip()

                    if callsign:

                        print(
                            f"Sent {icao} {callsign}"
                        )

                    else:

                        print(
                            f"Sent {icao}"
                        )

                    sent += 1

                except (OSError, ssl.SSLError) as e:

                    print(f"TAK connection lost: {e}")

                    try:
                        tak_socket.close()
                    except Exception:
                        pass

                    tak_socket = None

                    break

            # ------------------------------------------------
            # STATUS LOGGING
            # ------------------------------------------------

            if sent == 0:

                now = time.time()

                if now - last_idle_message >= 30:

                    print(
                        "Waiting for fresh positioned aircraft..."
                    )

                    last_idle_message = now

            # ------------------------------------------------
            # CONTINUOUS 2-SECOND UPDATE LOOP
            # ------------------------------------------------

            time.sleep(SEND_INTERVAL)

        except KeyboardInterrupt:

            print("ADSB2TAK stopping.")

            break

        except Exception as e:

            print(f"Unexpected error: {e}")

            print(
                f"Continuing in "
                f"{RECONNECT_INTERVAL} seconds..."
            )

            if tak_socket is not None:

                try:
                    tak_socket.close()
                except Exception:
                    pass

                tak_socket = None

            time.sleep(RECONNECT_INTERVAL)

    # --------------------------------------------------------
    # CLEAN SHUTDOWN
    # --------------------------------------------------------

    if tak_socket is not None:

        try:
            tak_socket.close()
        except Exception:
            pass

    print("ADSB2TAK stopped.")


# ============================================================
# PROGRAM ENTRY POINT
# ============================================================

if __name__ == "__main__":
    main()
```

Save the file and exit (`Ctrl+X`, then `Y`, then `Enter`).

---

## 3.6 Install the Aircraft Metadata Database
<br>This database is used to enrich raw ADS-B data with details like aircraft type, operator, and military/civilian status.

1. Navigate to the Database Directory:

```bash
cd ~/adsb2tak/database
```
2. Download the Database File:
```bash
wget -O aircraft.csv.gz https://github.com/wiedehopf/tar1090-db/raw/refs/heads/csv/aircraft.csv.gz
```

3. Verify the Download:
<br>Run these commands to ensure the file was downloaded correctly and is not corrupt.
```bash
# Check that the file has a non-zero size
ls -lh

# Test the archive integrity (no output means success)
gzip -t aircraft.csv.gz
```
---
### Next Step
The application environment, certificates, and data are now in place. You are ready to deploy and validate the service.

➡️ **[Step 4: Deployment & Validation](./4-deployment-validation.md)**

    
