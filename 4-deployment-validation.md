# Step 4: Deployment & Validation

This guide covers the final steps for deploying the `adsb2tak` application. We will perform pre-flight checks on the Python code, run an end-to-end test to the TAK Server, install the application as a persistent `systemd` service, and validate that it starts automatically on reboot.

---

## 4.1 Pre-Deployment Code Validation

Before deploying the script as a service, it's critical to run these checks to catch any syntax or logic errors in the Python code.

> **Note:** Ensure your Python virtual environment is active for these commands (`source ~/adsb2tak/.venv/bin/activate`).

1.  **Check for Syntax Errors:**
    Use the `py_compile` module to check the `adsb2tak.py` script for syntax errors. This command should be run from your application's home directory.
    ```bash
    cd ~/adsb2tak
    /home/takadmin/adsb2tak/.venv/bin/python -m py_compile adsb2tak.py
    ```
    *No output from this command means the syntax check passed successfully.*

2.  **Test the Classification Logic:**
    Run this self-contained test to ensure the aircraft classification logic correctly identifies fixed-wing vs. helicopter and civilian vs. military aircraft based on their ICAO codes.

    ```bash
    /home/takadmin/adsb2tak/.venv/bin/python - <<'PY'
    import adsb2tak

    # You must have adsb2tak.py in your current directory for this to work
    # and the database must be installed.

    tests = [
        ("A033A9", "Army EC145"),
        ("A1B9FC", "Navy King Air"),
        ("A12D67", "SkyWest E175"),
        ("00830B", "Civilian R44"),
    ]

    for icao, label in tests:
        cot_type, aircraft_class, military, info = adsb2tak.get_cot_type(icao)
        print("\n" + "="*20)
        print(label)
        print("="*20)
        print("ICAO:    ", icao)
        print("TYPE:    ", info.get("type") if info else "UNKNOWN")
        print("CLASS:   ", aircraft_class)
        print("MILITARY:", military)
        print("COT:     ", cot_type)
    PY
    ```

    **Expected Output:**
    The output should match the following classification results:

    | Test Case | Expected Class | Military | Expected CoT |
    |:---|:---|:---|:---|
    | Army EC145 | helicopter | `True` | `a-u-A-M-H` |
    | Navy King Air | fixed-wing | `True` | `a-u-A-M-F` |
    | SkyWest E175 | fixed-wing | `False` | `a-u-A-C-F` |
    | Civilian R44 | helicopter | `False` | `a-u-A-C-H` |

---

## 4.2 End-to-End Connection Test

The `test_marker.py` script sends a single, static test marker to the TAK Server. This is the best way to confirm that the entire communication path is working before running the live script.

1.  **Run the Test Script:**
    ```bash
    /home/takadmin/adsb2tak/.venv/bin/python ~/adsb2tak/test_marker.py
    ```

2.  **Verify in ATAK/WebTAK:**
    If the test is successful, a new marker with the UID `ADSB-TEST-01` and callsign `ADSB-TEST` will appear in your TAK client.

    *   **If the marker appears:** Your TLS connection, client certificate, CoT formatting, and TAK server ingestion are all working correctly.
    *   **If the marker does not appear:** Do not proceed. Re-check your TLS validation (Step 3.3) and network configuration (Step 2).

---

## 4.3 Deploy as a systemd Service

To ensure the `adsb2tak` gateway runs automatically and reliably in the background, we will configure it as a `systemd` service.

1.  **Create the Service File:**
    Use a text editor with `sudo` to create a new service file.
    ```bash
    sudo nano /etc/systemd/system/adsb2tak.service
    ```

2.  **Add the Service Configuration:**
    Copy and paste the entire block below into the editor. This defines how the service starts, who it runs as, and what to do if it fails.

    ```ini
    [Unit]
    Description=ADS-B to TAK Cursor-on-Target Gateway
    After=network-online.target zerotier-one.service dump1090-fa.service
    Wants=network-online.target zerotier-one.service dump1090-fa.service

    [Service]
    Type=simple
    User=takadmin
    Group=takadmin
    WorkingDirectory=/home/takadmin/adsb2tak
    ExecStart=/home/takadmin/adsb2tak/.venv/bin/python -u /home/takadmin/adsb2tak/adsb2tak.py
    Restart=always
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    ```
    Save the file and exit (`Ctrl+X`, then `Y`, then `Enter`).

3.  **Enable and Start the Service:**
    ```bash
    # Reload systemd to recognize the new service file
    sudo systemctl daemon-reload

    # Enable the service to start automatically on boot
    sudo systemctl enable adsb2tak.service

    # Start the service immediately
    sudo systemctl start adsb2tak.service
    ```

4.  **Check the Service Status:**
    ```bash
    systemctl status adsb2tak.service --no-pager
    ```
    *Look for `active (running)` in the status output. You can view live logs with `journalctl -u adsb2tak.service -f`.*

---

## 4.4 Final Autostart Validation

The final test is to reboot the system and confirm that the service starts automatically and can still connect to the TAK Server.

1.  **Reboot the System:**
    ```bash
    sudo reboot
    ```

2.  **Perform Post-Reboot Checks:**
    After the Raspberry Pi reconnects, log back in and run the following checks. **Do not manually start anything.**

    ```bash
    # Verify the adsb2tak service is active and running
    systemctl status adsb2tak.service --no-pager

    # Check the logs to see recent 'Sent' messages
    journalctl -u adsb2tak.service -b --no-pager | tail -n 20

    # Verify that hostname resolution is still working
    getent hosts takserver-01

    # Verify that the TAK Server port is still reachable
    nc -vz takserver-01 8089
    ```
    *If all checks pass, your `tak-sns01` node is fully deployed and operational.*

---

### Next Step

Your sensor node is now deployed. The final step is to understand how to monitor, troubleshoot, and maintain it during normal operations.

➡️ **[Step 5: Operations Guide](./5-operations-guide.md)**
