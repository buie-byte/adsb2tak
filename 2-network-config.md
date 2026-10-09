# Step 2: Network Configuration

This guide covers connecting the sensor node to a ZeroTier virtual private network (VPN) and configuring persistent local name resolution. These steps are critical for establishing a secure and reliable connection to your TAK Server.

---

## 2.1 Join the ZeroTier Network

ZeroTier creates a secure, private network layer between your `tak-sns01` node and your TAK server, regardless of their physical locations.

1.  **Install ZeroTier (if needed):**
    If ZeroTier is not already installed on your Raspberry Pi, you can install it with the following command:
    ```bash
    curl -s https://install.zerotier.com | sudo bash
    ```

2.  **Join Your ZeroTier Network:**
    Use the `zerotier-cli` command to join the same network that your TAK Server is on. Replace `<YOUR_NETWORK_ID>` with your actual 16-digit ZeroTier Network ID.
    ```bash
    sudo zerotier-cli join <YOUR_NETWORK_ID>
    ```

3.  **Authorize the New Node:**
    Log in to your ZeroTier Central account, find the new device with the `tak-sns01` hostname, and authorize it to join the network. You may also want to assign it a static IP address within ZeroTier.

4.  **Verify Network Status:**
    Once authorized, check the status on your Raspberry Pi.
    ```bash
    sudo zerotier-cli status
    sudo zerotier-cli listnetworks
    ```
    *The `status` command should show `200 INFO ONLINE`. The `listnetworks` command should show the network you joined with an `OK` status and an IP address assigned to your Pi.*

5.  **Record the TAK Server's IP:**
    From the member list in ZeroTier Central, find your TAK server (`takserver-01`) and write down its assigned ZeroTier IP address. You will need this for the next step.

---

## 2.2 Configure Persistent TAK Server Name Resolution

For TLS security to work correctly, the Python application will verify the server's certificate against the hostname `takserver-01`. Therefore, we must configure this sensor node to resolve the name `takserver-01` to the TAK server's private ZeroTier IP address.

> **Why This Is Necessary**
> On many modern OS images (especially those using `cloud-init`), simply editing the `/etc/hosts` file is not permanent and can be overwritten on reboot. The following steps ensure the name resolution is persistent.

1.  **Edit the `cloud-init` Host Template:**
    Open the template file with a text editor like `nano`.
    ```bash
    sudo nano /etc/cloud/templates/hosts.debian.tmpl
    ```

2.  **Add the Host Mapping:**
    Add a line to the template that maps the TAK server's ZeroTier IP to the name `takserver-01`. Replace `<TAK_SERVER_ZEROTIER_IP>` with the actual IP you recorded earlier.

    ```plaintext
    # The following lines are desirable for IPv4 capable hosts
    127.0.1.1 \$fqdn \$hostname
    127.0.0.1 localhost

    # ==> ADD THIS LINE <==
    <TAK_SERVER_ZEROTIER_IP>    takserver-01

    # The following lines are desirable for IPv6 capable hosts
    ...
    ```
    Save the file and exit (`Ctrl+X`, then `Y`, then `Enter`).

3.  **Apply the Template Change:**
    Run `cloud-init` to immediately apply the changes from the template to your `/etc/hosts` file.
    ```bash
    sudo cloud-init single --name update_etc_hosts --frequency always
    ```

4.  **Verify Name Resolution and Port Reachability:**
    Test that the name resolution is working and that the TAK Server port is reachable over the network.
    ```bash
    # Check if the name resolves to the correct IP
    getent hosts takserver-01

    # Check if the IP is in the /etc/hosts file
    grep takserver-01 /etc/hosts

    # Check network connectivity by pinging the server
    ping -c 4 takserver-01

    # Check if the TAK Server TLS port (8089) is open and reachable
    nc -vz takserver-01 8089
    ```
    *A successful `nc` command will report `Connection to takserver-01 8089 port [tcp/*] succeeded!`*

5.  **Final Persistence Test (CRITICAL):**
    Reboot the Raspberry Pi to ensure your changes survive a restart.
    ```bash
    sudo reboot
    ```
    After the system comes back online, run the verification commands from Step 4 again to confirm that name resolution and port reachability are still working correctly.

---

### Next Step

Your network configuration is now complete and verified. You are ready to set up the Python application itself.

➡️ **[Step 3: Application Setup](./3-application-setup.md)**
