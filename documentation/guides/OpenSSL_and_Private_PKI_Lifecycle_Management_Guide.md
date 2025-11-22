# OpenSSL and Private PKI Lifecycle Management Guide


| **Author:** | John Herr          |
| :---------- | ------------------ |
| **Date:**   | Novemeber 13, 2025 |

> **⚠️ Editor's Note/Disclaimer:** This document was collaboratively reviewed and refined by an expert AI (Gemini) to enhance its clarity, structure, and readability for publication. The core technical content, expertise, and original configuration details were provided by the author.


This guide provides a structured reference for establishing a local Private Certificate Authority (CA) and managing the complete certificate lifecycle, focusing on the actions required by both the **Requester (Host)** and the **CA Administrator**. The primary audience is an administrator who manages PKI occasionally.

-----

## Configuration Files Reference

### CA Configuration File (`ca.cnf`)

This file governs the CA's security policies, file locations, and certificate extensions. **Note:** The `unique_subject = no` line has been added for practical renewal workflow management.

```title:ca.cnf
####################################################################
[ ca ]
default_ca = CA_default

####################################################################
[ CA_default ]
# CA directory structure
dir              = .                        # Base directory is the current directory
certs            = $dir/certs               # Directory for archived certificates
crl_dir          = $dir/crl                 # Directory for CRLs
database         = $dir/index.txt           # The CA database ledger file
new_certs_dir    = $dir/newcerts            # Directory for newly signed certs (named by serial)
certificate      = $dir/myca.crt            # The Root CA certificate file name
serial           = $dir/serial              # The current serial number file
private_key      = $dir/private/myca.key    # The Root CA private key file name
RANDFILE         = $dir/private/.rand       # Private random number file
default_crl_days = 30                       # Default validity of crl

# Certificate signing defaults
default_days     = 365                      # 1 year validity for signed certificates
default_md       = sha256                   # SHA-256 digest
policy           = policy_strict            # Strict subject field matching
x509_extensions  = v3_standard_cert         # Extensions for signed certificates
copy_extensions  = copy                     # Copy SANs from CSR to signed certificate
unique_subject   = no                       # **IMPORTANT** Allows signing of certs with the same CN (e.g., for renewals)

####################################################################
[ policy_strict ]
# Subject field policy: must match CA's subject fields
countryName            = match
stateOrProvinceName    = match
organizationName       = match
organizationalUnitName = optional
commonName             = supplied
emailAddress           = optional

####################################################################
[ req ]
# Settings for creating the Root CA certificate
default_bits        = 4096                  # 4096-bit key for Root CA
default_md          = sha256
prompt              = yes                   # Interactive prompts for subject fields
encrypt_key         = yes                   # Encrypt private key with passphrase
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca                 # Extensions for Root CA certificate

####################################################################
[ req_distinguished_name ]
# Subject field prompts and defaults for Root CA creation
countryName                     = Country Name (2 letter code)
countryName_default             = US

stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = Texas

localityName                    = Locality Name (eg, city)
localityName_default            = Austin

organizationName                = Organization Name (eg, company)
organizationName_default        = My Org

commonName                      = Common Name (e.g. server FQDN or YOUR name)
commonName_default              = 357 Root CA
commonName_max                  = 64

emailAddress                    = Email Address
emailAddress_default            = ca-admin@example.org
emailAddress_max                = 64

####################################################################
[ v3_ca ]
# Extensions for Root CA certificate (self-signed)
basicConstraints        = critical, CA:TRUE
keyUsage                = critical, digitalSignature, cRLSign, keyCertSign
subjectKeyIdentifier    = hash

####################################################################
[ v3_standard_cert ]
# Extensions for signed end-entity certificates
basicConstraints        = CA:FALSE
authorityKeyIdentifier  = keyid:always,issuer:always
subjectKeyIdentifier    = hash
keyUsage                = digitalSignature, keyEncipherment
extendedKeyUsage        = serverAuth, clientAuth
```

Notable settings

| Section          | Key              | Value    | Note                                                                                                                     |
| ---------------- | ---------------- | -------- | ------------------------------------------------------------------------------------------------------------------------ |
| CA_default       | copy_extensions  | copy     | Copy SANs from CSR to signed certificate                                                                                 |
|                  | unique_subject   | no       | Allows signing of certs with the same CN (e.g., for renewals)                                                            |
| v3_ca            | basicContraints  | CA:TRUE  | Grants the certificate holder the authority to sign (issue) other certificates. This is for the CA certificate           |
| v3_standard_cert | basicConstraints | CA:FALSE | Does not grant the certificate holder the authority to sign (issue) other certificates. This is for non-CA certificates. |

### Requester Configuration File (`csr.cnf`)

This file is used by the host to generate the CSR, ensuring critical modern fields like **Subject Alternative Names (SANs)** are included.

```title:csr.cnf
[ req ]
default_bits       = 2048                     # Recommended minimum key size
default_md         = sha256                   # Digest algorithm for the signature
prompt             = no                       # Do not prompt for subject fields
encrypt_key        = yes                      # Encrypt the private key with a passphrase
distinguished_name = req_distinguished_name
req_extensions     = req_ext                  # Use this extension section for the CSR

[ req_distinguished_name ]
# These fields will be used directly because prompt = no
countryName         = US
stateOrProvinceName = Texas
localityName        = Austin
organizationName    = My Org
commonName          = dquay.example.org

[ req_ext ]
basicConstraints    = CA:FALSE
keyUsage            = digitalSignature, keyEncipherment
subjectAltName      = @alt_names

[ alt_names ]
DNS.1 = dquay.example.org
IP.1  = 10.0.0.15
```

| Section   | Key              | Value      | Note                                                                                                                                                                   |
| --------- | ---------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| req_ext   | basicConstraints | CA:FALSE   | Does not grant the certificate holder the authority to sign (issue) other certificates. This is for non-CA certificates.                                               |
| alt_names | DNS.{1,2,3,...}  | FQDN       | Alternative (SAN) names to use instead of commonName. Should always put at least the commonName here since commonName is deprecated and some services require SAN now. |
|           | IP.{1,2,3,...}   | IP Address | Same as DNS, but for IP Addresses.                                                                                                                                     |

-----

## TL;DR: Quick Reference Commands

This quick reference table separates commands for the **Requester** and the **CA Admin** for each major process.

| Process                                                 | Persona       | Description                                             | Commands                                                                                                                                                                                                                                                                                                                                                              |
| :------------------------------------------------------ | :------------ | :------------------------------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [[#Initial CA Setup]]                                   | **CA Admin**  | Create structure, key, and self-signed root cert.       | `mkdir my_local_ca`<br>`cd my_local_ca`<br>`mkdir certs crl newcerts private`<br>`chmod 700 private`<br>`touch index.txt`<br>`echo 1000 > serial`<br>`openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -aes256 -out private/myca.key`<br>`openssl req -new -x509 -days 3650 -key private/myca.key -sha256 -extensions v3_ca -out myca.crt -config ca.cnf` |
| [[#CSR Generation]]                                     | **Requester** | Generate private key and CSR for a service.             | `openssl genpkey -algorithm RSA -out webserver.key -aes256`<br>`chmod 400 webserver.key`<br>`openssl req -new -key webserver.key -out webserver.csr -config csr.cnf`                                                                                                                                                                                                  |
| [[#Certificate Signing]]                                | **CA Admin**  | Sign the submitted CSR.                                 | `openssl ca -in webserver.csr -out newcerts/webserver.crt -config ca.cnf -extensions v3_standard_cert -days 375 -notext -batch`<br>`# Note: OpenSSL also places a serial-named copy in newcerts/`<br>`mv newcerts/*.pem certs/`                                                                                                                                       |
| [[#Requester Steps Key and CSR Generation for Renewal]] | **Requester** | Generate a **NEW** key and CSR for the renewal process. | `openssl genpkey -algorithm RSA -out webserver_new.key -aes256`<br>`openssl req -new -key webserver_new.key -out webserver_new.csr -config csr.cnf`                                                                                                                                                                                                                   |
| [[#CA Admin Steps Signing the Renewal]]                 | **CA Admin**  | Sign the new CSR and archive the old cert.              | `openssl ca -in webserver_new.csr -out newcerts/webserver_new.crt -config ca.cnf -extensions v3_standard_cert -days 375 -notext -batch`<br>`mv newcerts/*.pem certs/`                                                                                                                                                                                                 |
| [[#CA Admin Steps Certificate Revocation]]              | **CA Admin**  | Revoke the certificate and generate the updated CRL.    | `openssl ca -revoke certs/webserver.crt -config ca.cnf`<br>`openssl ca -gencrl -out crl/myca.crl -config ca.cnf`                                                                                                                                                                                                                                                      |
| [[#Requester Steps Certificate Revocation]]             | **Requester** | Replace the revoked cert with a new one (if required).  | `# Securely delete old key`<br>`openssl genpkey -algorithm RSA -out webserver_v2.key -aes256`<br>`openssl req -new -key webserver_v2.key -out webserver_v2.csr -config csr.cnf`                                                                                                                                                                                       |

-----

## 🛠️ Detailed Process Steps

### Initial CA Setup

This process creates the necessary CA directory structure and the self-signed trust anchor.

#### CA Admin Steps: Setup and Root Certificate Creation

1.  **Setup Directory Structure:** Create the file system necessary for the OpenSSL CA database.
```bash
mkdir my_local_ca
cd my_local_ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
```

| **File/Directory** | **Purpose**                                                                                                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| index.txt          | The CA database ledger (empty to start). Tracks issued, expired, and revoked certificates. It includes details of every certificate the CA has ever signed. It starts as an empty file.                       |
| serial             | Contains the next certificate serial number (starts at 1000). It ensures every certificate has a unique serial number.                                                                                        |
| certs              | This directory is typically used to store **old, revoked, or expired certificates**. It acts as an archive of past certificates handled by the CA.                                                            |
| crl                | This stands for **Certificate Revocation List**. It is used to store the publicly available lists of certificates that have been revoked (i.e., invalidated before their expiry date).                        |
| newcerts           | This is the directory where **newly issued and signed certificates** are placed immediately after the CA signs them. They are then moved or distributed from here.                                            |
| private/           | It stores the CA's **secret private key** (`ca.key`). It is essential that this directory and the key file within it have extremely restricted permissions (like `chmod 700`) to prevent unauthorized access. |

2.  **Generate Root CA Private Key:** Create the master secret key, securing it with a strong passphrase.
```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -aes256 -out private/myca.key
chmod 400 private/myca.key
```

3.  **Create Self-Signed Root Certificate:** Generate the public trust anchor (`myca.crt`) using the key and the `v3_ca` extensions.
```bash
openssl req -new -x509 -days 3650 -key private/myca.key -sha256 -extensions v3_ca -out myca.crt -config ca.cnf
```

| Command Option | Purpose                                                                |
| -------------- | ---------------------------------------------------------------------- |
| -x509          | Tells OpenSSL to create a self-signed certificate instead of a CSR     |
| -days          | Validity of certificate                                                |
| -extensions    | Extension section in CA configuration file to use. `v3_ca` is for a CA |

### CSR Generation

This is the process of generating the keypair on the host.

#### Requester Steps: Key and CSR Generation

1.  **Create Service Private Key:** Generate a new key and apply restrictive permissions.
```bash
openssl genpkey -algorithm RSA -out webserver.key -aes256
chmod 400 webserver.key
```

> Note: While it is best practice to put a passphrase here when prompted, some services will not use an encrypted certificate and will fail.

2.  **Generate Certificate Signing Request (CSR):** Use the private key and the configuration file to create the request.
```bash
openssl req -new -key webserver.key -out webserver.csr -config csr.cnf
```

3.  **Submit:** Securely send `webserver.csr` to the CA Admin.

> Note: **The original `webserver.key` (private key) remains on the host and is never shared.**

### Certificate Signing

This is where the CA validates and signs the request, issuing the new certificate.

#### CA Admin Steps: Signing and Issuance

1.  **Sign the CSR:** Sign the request using the CA's private key (passphrase required).
```bash
openssl ca -in webserver.csr -out newcerts/webserver.crt -config ca.cnf -extensions v3_standard_cert -days 375 -notext -batch
```

| Command Option               | Purpose                                                                                                              |
| :--------------------------- | :------------------------------------------------------------------------------------------------------------------- |
| -in webserver.csr            | Specifies the input CSR file.                                                                                        |
| -out certs/webserver.crt     | Specifies the output file path for the new signed certificate.                                                       |
| -extensions v3_standard_cert | Applies the server extensions defined in the [ v3_standard_cert ] section of ca.cnf (sets CA:FALSE, keyUsage, etc.). |
| -days 375                    | Sets the validity period to 375 days (common for host certificates).                                                 |
| -notext -batch               | Suppresses verbose output and runs non-interactively.                                                                |

2.  **Archive the Serialized Copy:** Move the automatically generated serial-named copy from the `newcerts` staging directory to the permanent `certs` archive.
```bash
mv newcerts/*.pem certs/
```

3.  **Issue Files:** Send the signed certificate (`webserver.crt`) and the CA's public certificate (`myca.crt`) back to the Requester.

### Certificate Renewal

The standard workflow involves generating a new keypair and CSR to replace the soon-to-expire certificate.

#### Requester Steps: Key and CSR Generation for Renewal

1.  **Generate a NEW Private Key:** Create a fresh key to maintain forward secrecy.
```bash
openssl genpkey -algorithm RSA -out webserver_new.key -aes256
```

2.  **Generate a NEW CSR:** Use the new key and submit the request.
```bash
openssl req -new -key webserver_new.key -out webserver_new.csr -config csr.cnf
```

3.  **Install:** Once signed, install the new key and certificate, and archive or delete the old, expired key/certificate.

#### CA Admin Steps: Signing the Renewal

1.  **Sign the New CSR:** Sign the new CSR. The CA database gains a new entry with a different serial number. The `unique_subject = no` setting in `ca.cnf` allows this renewal with the same hostname.
```bash
openssl ca -in webserver_new.csr -out newcerts/webserver_new.crt -config ca.cnf -extensions v3_standard_cert -days 375 -notext -batch
```

2.  **Archive the Serialized Copy:** Move the serial-named copy to the permanent archive.
```bash
mv newcerts/*.pem certs/
```

### Certificate Revocation

This process invalidates a certificate before its expiration date and updates the Certificate Revocation List (CRL).

#### CA Admin Steps: Certificate Revocation

1.  **Revoke the Certificate:** Update the internal database (`index.txt`), marking the certificate as revoked.
```bash
openssl ca -revoke certs/webserver.crt -config ca.cnf
```

2.  **Generate the CRL:** Create the public list of revoked certificates for client consumption.
```bash
openssl ca -gencrl -out crl/myca.crl -config ca.cnf
```

3.  **Verify Revocation:** Check the contents to confirm the certificate's serial number is listed.
```bash
openssl crl -in crl/myca.crl -text -noout
```

#### Requester Steps: Certificate Revocation

1.  **Stop Service and Delete Key:** Immediately stop the service using the compromised key and securely delete the compromised key file.

2.  **Notify CA Admin:** Request revocation.

3.  **Generate Replacement Keypair and CSR:** Create a fresh keypair and CSR for a replacement certificate.
```bash
openssl genpkey -algorithm RSA -out webserver_v2.key -aes256
openssl req -new -key webserver_v2.key -out webserver_v2.csr -config csr.cnf
```

4.  **Install Replacement:** Install the new key and certificate once issued by the CA.

-----
## Extras
### Troubleshooting and Gotchas

This section highlights common errors and important nuances for the occasional administrator.

| Issue                                                                                          | Cause                                                                                                                                          | Fix                                                                                                                              |
| :--------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------- |
| **SANs are missing**                                                                           | The **`-config`** flag was omitted when running `openssl req`.                                                                                 | Re-run the command with: `openssl req -new... -config csr.cnf`.                                                                  |
|                                                                                                | The `copy_extensions = copy` setting is missing.                                                                                               | Add the `copy_extensions = copy` line under the CA_default section in the CA's configuration file.                               |
| **Invalid Field Name Error**                                                                   | Using the `_default` suffix (e.g., `countryName_default`) when `prompt = no` is set.                                                           | Remove the `_default` suffix from the field names in the `[req_distinguished_name]` section.                                     |
| **TXT\_DB error number 2**                                                                     | Attempting to sign a CSR with a Common Name (or Subject) that already exists in the `index.txt` database.                                      | Ensure `unique_subject = no` is set in the `[CA_default]` section of `ca.cnf` to allow duplicates for renewals/replacements.     |
| **Signed Cert Lacks SANs**                                                                     | The `[CA_default]` section in `ca.cnf` is missing **`copy_extensions = copy`**.                                                                | Verify that `copy_extensions = copy` is present in your `[CA_default]` section.                                                  |
| **Permission Denied**                                                                          | The CA private key (`private/myca.key`) or the host's private key (`webserver.key`) does not have restrictive permissions (e.g., `chmod 400`). | Correct the permissions using `chmod 400 <file>`.                                                                                |
| **The stateOrProvinceName field is different between  CA certificate (X) and the request (Y)** | The CA's configuration file has `stateOrProvinceName` set to `match` in the `policy_strict` section.                                           | Either change the CSR's configuration to have the same stateOrProvinceName or change the setting in the CA's configuration file. |

-----

### Handling Duplicate Common Names

#### Common Name Uniqueness

When signing a Certificate Signing Request (CSR) with `openssl ca`, the default behavior is to enforce a **unique subject**. The `[policy_strict]` section in your `ca.cnf` ensures the country, state, and organization fields match the CA's, but the Common Name (CN) is set to `supplied`.

The primary issue is not with the CN, but with the entire **Subject Distinguished Name**. If you try to sign a new certificate for a CN that is already present in the CA's internal database (`index.txt`), the `openssl ca` command will fail with a **"TXT\_DB error number 2"**.

#### Solution: Allow Duplicate Subjects

To generate a new certificate with the same subject name as a previous certificate (which is necessary for renewals/replacements), you must explicitly tell the CA to allow non-unique subjects.

In your `ca.cnf`, under the `[ CA_default ]` section, you must add the following directive:

```ini
[ CA_default ]
# ... other settings
unique_subject = no  # ✅ Allows the CA to sign certificates with the same CN/Subject.
# ...
```

This is the standard approach for renewing certificates with the same hostname.

#### Automatic Serial Number Naming

OpenSSL places a copy of the newly signed certificate into the **`newcerts`** directory, automatically named after its serial number (e.g., `1000.pem`).

This design is excellent for the **CA Administrator** because it provides an immediate, immutable, and unique record of the signed certificate tied to its serial number, which is essential for tracking and revocation.

**Recommended Workflow:**

1.  The CA signs the CSR, which outputs the file to the path specified by `-out` (e.g., `newcerts/webserver.crt`).
2.  The CA also places a copy in the archive location (e.g., `newcerts/1000.pem`).
3.  The CA Admin **manually moves** the serial-named certificate to the permanent archive directory (`certs/`) for long-term tracking.

-----

### Best Practice: Different Certs for Different Services

**You should generally use separate certificates for distinct services** (e.g., a web server and a mail server), even if they are on the same host. This follows the **Principle of Least Privilege (PoLP)**: if one service's key is compromised, the keys for other services remain secure, limiting the scope of attack.

While you could theoretically create one certificate with both hostnames (`www.example.org` and `mail.example.org` in the SAN), using **separate certificates with separate private keys** is the recommended best practice:

1. **Isolation:** If the mail server's private key is compromised (e.g., due to a software vulnerability), the **web server's private key remains secure**, limiting the scope of the attack.
    
2. **Service-Specific Extensions:** Although both are generally "serverAuth," a mail server certificate may sometimes require slightly different or fewer **Key Usage** extensions than a web server certificate, making a dedicated cert cleaner.
    
3. **Deployment:** Each certificate is installed and managed independently in the configuration file of its respective service (Apache/Nginx for web, Postfix/Exim for mail), simplifying maintenance.


#### Differentiation by Hostname (SAN)

Certificates are differentiated primarily by the hostnames listed in the **Subject Alternative Name (SAN)** field.
- A certificate for a web server (`www.example.org`) must have that specific FQDN listed in the SAN.    
- A certificate for a mail server (`mail.example.org`) must have its FQDN listed in its own separate SAN list.    

If a client connects to a hostname that is not in the certificate's SAN list, it will result in a **hostname mismatch error**.

#### The Non-WWW vs. WWW Problem

A web server certificate often requires two entries in the SAN: **`example.org` (the base domain)** and **`www.example.org` (the subdomain)**. This is because browsers treat these as two distinct hostnames. Including both ensures the connection is trusted regardless of how the user types the address.

### Certificate Installation and Trust

**High-Level Overview:** The signed certificate is installed on the web server, and the Root CA certificate must be installed on client machines (workstations/browsers) to establish trust in the new certificate chain.

#### TL;DR (Installation and Trust)

1. **Web Server Install:**    
    - Configure web server (Nginx/Apache) to use `webserver.key`, `webserver.crt`, and `ca.crt`.
        
2. **System Trust (Fedora/Linux):**    
    - `sudo cp ca.crt /etc/pki/ca-trust/source/anchors/`        
    - `sudo update-ca-trust extract` (for system apps, Chrome/Edge).
        
3. **Firefox Trust:**    
    - Manually import `ca.crt` into Firefox's **Authorities** settings.
        


#### Web Server Installation

The signed certificate (`webserver.crt`) and the CA certificate (`ca.crt`) are configured in the web server software (e.g., Nginx, Apache) alongside the original private key (`webserver.key`).

``` nginx
server {
    listen 443 ssl;
    server_name webserver.devopscorp.local;
    
    # The server's private key (MUST be secure)
    ssl_key /etc/ssl/private/webserver.key; 
    
    # The server's signed certificate
    ssl_certificate /etc/ssl/certs/webserver.crt;
    
    # The CA's certificate (or a bundle) - completes the chain of trust
    ssl_trusted_certificate /etc/ssl/certs/ca.crt; 
    
    # ... other settings
}
```

#### Trusting the CA on Your Fedora Workstation (System Trust Store)

To avoid security warnings in applications that rely on the system trust store (like cURL, wget, Chrome, Edge), you must install your `ca.crt` into the operating system's trust store.

The process for Fedora (and similar Linux distributions) uses `update-ca-trust`:
1. **Copy the CA Certificate** to the designated system anchor directory:

   ``` bash
    # Assuming ca.crt is in your current directory
    sudo cp ca.crt /etc/pki/ca-trust/source/anchors/
    ```
    
2. **Update the Trust Store** to incorporate the new certificate:
    
    ``` bash
    sudo update-ca-trust extract
    ```
    

#### Browser Trust (Firefox vs. Chrome/Edge)

- **Google Chrome / Chromium / Edge:** These browsers typically rely on the **Operating System's system trust store**. Once you complete the `update-ca-trust` step above, these browsers will automatically trust the CA.
    
- **Mozilla Firefox:** This browser uses its **own independent trust store** for cross-platform compatibility. You must manually import the `ca.crt` file via the browser settings:
    
    1. Open **Settings** > **Privacy & Security** > **View Certificates**.        
    2. Navigate to the **Authorities** tab.        
    3. Click **Import...** and select your `ca.crt` file.        
    4. Check the box for **"Trust this CA to identify websites."**




