# Step 1: System Setup

This guide covers the initial setup of the Raspberry Pi operating system, the installation of the RTL-SDR radio, and the configuration of the `dump1090-fa` ADS-B decoder.

---

## 1.1 Hardware & Software Prerequisites

Ensure you have the following components before you begin:

*   **Raspberry Pi**: A Raspberry Pi 3B (or newer) running a current version of Raspberry Pi OS (or a similar Debian-based Linux distribution).
*   **SDR**: An RTL-SDR Blog V3 USB software-defined radio.
*   **Antenna**: A 1090 MHz ADS-B antenna with the appropriate cables to connect to the SDR.

The remaining software components (ZeroTier, Python, etc.) will be installed in subsequent steps.

---

## 1.2 Prepare the Raspberry Pi

First, we will set the system's hostname to `tak-sns01` and apply all current OS updates.

1.  **Set the Hostname:**
    ```bash
    sudo hostnamectl set-hostname tak-sns01
    ```

2.  **Update the System:**
    ```bash
    sudo apt update
    sudo apt full-upgrade -y
    ```

3.  **Verify the Hostname:**
    Reboot the Pi or open a new terminal. The command prompt should now show `tak-sns01`. You can also run the following command to confirm:
    ```bash
    hostname
    ```
    *Expected output: `tak-sns01`*

---

## 1.3 Install and Verify the RTL-SDR

Next, install the drivers for the RTL-SDR and test that the hardware is recognized by the system.

1.  **Install the `rtl-sdr` Package:**
    ```bash
    sudo apt install rtl-sdr -y
    ```

2.  **Verify the USB Device:**
    Plug in the RTL-SDR dongle and run `lsusb` to confirm the operating system sees it.
    ```bash
    lsusb
    ```
    *Look for a device listed as a `Realtek RTL2838 DVB-T` device (e.g., USB ID `0bda:2838`).*

3.  **Test the SDR Hardware:**
    Run the `rtl_test` utility to perform a basic hardware test.
    ```bash
    rtl_test -t
    ```
    *You should see it find the device and report supported tuner types. Press `Ctrl+C` to exit after a few seconds.*

    > **Note**
    > If this command fails because the device is busy, it may already be in use by `dump1090-fa` (if installed). You can temporarily stop the service with `sudo systemctl stop dump1090-fa` before re-running the test.

---

## 1.4 Install and Verify dump1090-fa

Finally, install the `dump1090-fa` software from FlightAware, which is responsible for decoding ADS-B signals from the SDR.

1.  **Download the FlightAware Repository Package:**
    This package configures `apt` to use the FlightAware repository.
    ```bash
    wget https://www.flightaware.com/adsb/piaware/files/packages/pool/piaware/f/flightaware-apt-repository/flightaware-apt-repository_1.3_all.deb
    ```
    > **Warning**
    > The repository URL may change. Before running this on a new OS image, verify that this is still the current package on the [FlightAware PiAware page](https://www.flightaware.com/adsb/piaware/install.rvt).

2.  **Install the Repository and `dump1090-fa`:**
    ```bash
    sudo dpkg -i flightaware-apt-repository_1.3_all.deb
    sudo apt update
    sudo apt install dump1090-fa -y
    ```

3.  **Enable and Verify the Service:**
    Check that the service is running correctly and is enabled to start on boot.
    ```bash
    sudo systemctl enable dump1090-fa
    systemctl status dump1090-fa --no-pager
    ```
    *Look for `active (running)` in the status output.*

4.  **Check for Live Aircraft Data:**
    If aircraft are nearby, `dump1090-fa` will create a JSON file with live track data.
    ```bash
    ls -l /run/dump1090-fa/
    ```
    *You should see several files, including `aircraft.json`. View its contents to see live data:*
    ```bash
    cat /run/dump1090-fa/aircraft.json
    ```
    *This file is the data source for our Python application.*

---

### Next Step

The foundational system is now prepared. You are ready to proceed with networking configuration.

➡️ **[Step 2: Network Configuration](./2-Network-Config.md)**
