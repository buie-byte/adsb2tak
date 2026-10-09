# Step 5: Operations Guide

This guide provides instructions for the operation of your `tak-sns01` node. It covers how to monitor the service, perform quick health checks, troubleshoot common issues, and safely update the application script.

---

## 5.1 Monitoring Live Operations 

The primary tool for monitoring the `adsb2tak` service is `journalctl`, which provides access to the system logs.

To view a live, continuous stream of log messages from the service, use the following command:

```bash
journalctl -u adsb2tak.service -f
```

*Press `Ctrl+C` to exit the live view.*

### Healthy Log Behavior

When the service is running correctly, you should see the following patterns in the log output:

**Healthy Startup:**  
Upon starting, the service will log the database loading and the successful establishment of a TLS connection to the TAK Server.

```plaintext
Loading aircraft database: /home/takadmin/adsb2tak/database/aircraft.csv.gz
Loaded 623176 aircraft database records.
ADSB2TAK starting...
TAK destination: takserver-01:8089
Connecting to TAK Server takserver-01:8089...
TLS connection established.
```

**Live Track Processing:**  
After connecting, the service will enter its main loop, processing and sending aircraft data approximately every two seconds.

```plaintext
Sent AC538F SWA3430 [B38M] fixed-wing a-u-A-C-F
Sent A46D95 UAL2421 [B739] fixed-wing a-u-A-C-F
Sent A89443 N652BB [B06] helicopter a-u-A-C-H
Sent <ICAO> <CALLSIGN> [B350] fixed-wing a-u-A-M-F
```

---

## 5.2 Quick System Health Check 

This series of commands allows you to quickly verify that each component of the system is functioning correctly.

*   **Check the ADS-B Decoder:**
    ```bash
    systemctl is-active dump1090-fa
    ```

*   **Check for Live ADS-B Data:**
    ```bash
    # This command should output a JSON structure with aircraft data
    cat /run/dump1090-fa/aircraft.json | head -n 10
    ```

*   **Check the Aircraft Database:**
    ```bash
    # The gzip test should produce no output if the file is valid
    gzip -t ~/adsb2tak/database/aircraft.csv.gz
    ls -lh ~/adsb2tak/database/aircraft.csv.gz
    ```

*   **Check the ZeroTier Network:**
    ```bash
    sudo zerotier-cli status
    sudo zerotier-cli listnetworks
    ```

*   **Check TAK Server Name Resolution and Port:**
    ```bash
    getent hosts takserver-01
    nc -vz takserver-01 8089
    ```

*   **Check the Gateway Service:**
    ```bash
    # Verify the service is running
    systemctl is-active adsb2tak

    # View the last 30 log entries
    journalctl -u adsb2tak.service -n 30 --no-pager
    ```

---

## 5.3 Troubleshooting Guide 

If the system is not behaving as expected, use this guide to diagnose the most likely cause.

| Symptom | Likely Cause / Next Action |
|---|---|
| No aircraft appear in `aircraft.json`. | The issue is with the SDR or `dump1090-fa`. Troubleshoot the antenna, SDR connection, and `dump1090-fa` service first. |
| `aircraft.json` has data, but nothing is sent. | The aircraft in the file may be stale or lack position data. Check that `lat`, `lon`, and `seen` fields are present and `seen` is less than 10. |
| Service fails on startup with database errors. | The `aircraft.csv.gz` file may be missing or corrupt. Re-download it and run `gzip -t` to verify. |
| TLS connects, then crashes with `too many values to unpack`. | A call to `get_cot_type()` in your Python script is incorrect. Ensure every call unpacks four values (e.g., `cot_type, a_class, is_mil, a_info = get_cot_type(icao)`). |
| One batch of "Sent" messages appears, then silence. | The main processing loop in `adsb2tak.py` is likely indented inside the `if tak_socket is None:` block. It must be outside this condition. |
| All aircraft appear as generic air symbols in TAK. | The classification logic may be failing. Check the `journalctl` logs to see what CoT type is being sent (e.g., `a-u-A-C-F`). |
| Connection is refused by the server. | Verify the TAK Server is running and its TLS input on port 8089 is active. Also check server-side firewalls. |
| The name `takserver-01` is not known. | Name resolution has failed. Run `getent hosts takserver-01` and check your `cloud-init` template from Step 2. |
| `PEM pass phrase` prompt appears during service startup. | The service is using the wrong private key. Ensure `KEY_FILE` in your script points to `adsb2tak-service.key`, not `adsb2tak.key`. |

---

## 5.4 Safe Update Procedure

Follow these steps to safely update the `adsb2tak.py` script while minimizing downtime and providing a quick rollback path.

1.  **Backup the Current Script:**  
    Create a timestamped backup of the working script before making any changes.
    ```bash
    cp ~/adsb2tak/adsb2tak.py ~/adsb2tak/adsb2tak.py.backup-$(date +%Y%m%d-%H%M%S)
    ```

2.  **Edit the Script:**  
    Make your desired changes to the Python script.
    ```bash
    nano ~/adsb2tak/adsb2tak.py
    ```

3.  **Perform Pre-flight Checks:**  
    Before restarting the service, check the new script for syntax errors.
    ```bash
    # Activate the venv if not already active
    source ~/adsb2tak/.venv/bin/activate
    
    # Run the syntax check (no output means success)
    python -m py_compile ~/adsb2tak/adsb2tak.py
    ```

4.  **Restart the Service:**  
    Apply the changes by restarting the `systemd` service.
    ```bash
    sudo systemctl restart adsb2tak.service
    ```

5.  **Monitor the Logs:**  
    Immediately check the status and live logs to ensure the service starts and runs correctly with your changes.
    ```bash
    systemctl status adsb2tak.service --no-pager
    journalctl -u adsb2tak.service -f
    ```

---

### End of Guide

You have now completed all the steps for setting up, deploying, and maintaining your `tak-sns01` ADS-B sensor node.

⬅️ **[Return to Main README](./README.md)**
