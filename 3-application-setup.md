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

## 3.5 Install the Aircraft Metadata Database
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
### Next Step
<br>The application environment, certificates, and data are now in place. You are ready to deploy and validate the service.

➡️ Step 4: Deployment & Validation
➡️ **[Step 4: Deployment & Validation](./docs/4-deployment-validation.md)**

    
