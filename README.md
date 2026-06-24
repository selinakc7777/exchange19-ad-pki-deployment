# On-Premises Core Infrastructure: Active Directory & Exchange Server 2019 Lab Documentation

This repository documents the complete configuration of a mock enterprise network environment under the internal root domain `selina.world`. It covers Active Directory deployment, Exchange Server 2019 implementation, end-to-end local mail routing, custom SSL/TLS Certificate Authority deployment, and automated Group Policy client trust orchestration.

---

## System Topology & IP Allocations

| Hostname | Role | Operating System | IP Address | Primary DNS |
| --- | --- | --- | --- | --- |
| **Edge Router** | Gateway / WAN Access | - | DHCP (Dynamic) | Auto |
| **DC1** | Domain Controller (AD DS / DNS / AD CS) | Windows Server | `192.168.1.10` | `192.168.1.10` (Loopback) |
| **Mail (Server1)** | Mail Server (Exchange Server 2019 CU15) | Windows Server | `192.168.1.20` | `192.168.1.10` |
| **PC1** | Client Workstation | Windows 10/11 | DHCP | `192.168.1.10` |
| **PC2** | Client Workstation | Windows 10/11 | DHCP | `192.168.1.10` |

---

## Section 1: Active Directory Domain Services (AD DS) Setup

### Concept Breakdown: Domain Control

* **Active Directory Domain Services (AD DS):** A centralized database service that authenticates and authorizes all users and computers in a Windows network.
* **Domain Controller (DC):** The physical or virtual server running AD DS that maintains the user registry, processes login requests, and manages permissions.
* **Forest / Root Domain:** The top-level container structure in Active Directory. In this environment, `selina.world` is the root namespace representing the corporate perimeter.

### Deployment Steps

#### 1. Edge Router Initialization

* Set WAN interface to **Auto [DHCP]** to pick up external addressing.
* Verify live internet connectivity upstream prior to local static scoping.

#### 2. Base Configuration of DC1

* Change host adapter to static IP `192.168.1.10` with DNS pointing natively to `192.168.1.10`.
* Rename server system hostname to `DC1` and perform a system reboot.

#### 3. Role Installation & Promotion

* Open **Server Manager** $\rightarrow$ **Add Roles and Features** $\rightarrow$ select **Active Directory Domain Services (AD DS)** $\rightarrow$ Proceed with installation.
* Click the alert flag and select **Promote this server to a domain controller**.
* Choose **Add a new forest** and specify the Root Domain Name: `selina.world`.
* Execute installation and let the system reboot to finalize directory generation. Run pending Windows Updates post-reboot.

---

## Section 2: Exchange Server 2019 CU15 Pre-requisites & Deployment

### Concept Breakdown: Enterprise Mail Platforms

* **Exchange Server 2019 (CU15):** A local, enterprise-grade email and messaging framework. It extends the Active Directory database schemas to add tracking fields for user mailboxes.
* **RAID 5 Volume:** A storage technique that combines three or more hard drives to build data striping with distributed parity. This optimizes read performance and ensures that if one drive fails, no email data is lost.
* **EAC (Exchange Admin Center):** The secure web-based console used by network administrators to manage mailboxes, migration, parameters, and structural mail flow.

### Deployment Steps

#### 1. Base Configuration of Mail Server (Server1)

* Set static IP assignment to `192.168.1.20`. Set the primary DNS field specifically to `192.168.1.10` (pointing to **DC1**).
* Rename host to `Mail` and reboot.

#### 2. Local Resilient Storage Provisioning

* On **DC1**, open **Computer Management** $\rightarrow$ **Disk Management**.
* Bring new raw target disks online, initialize, and format them into a unified **New RAID 5 Volume** mapped to drive letter `E:` with the volume label `Exchange`.

#### 3. Domain Joining & Exchange Setup Preps

* Download the **Exchange Server 2019 Cumulative Update 15 (CU15)** ISO image.
* Open `sysdm.cpl` (System Properties), click *Change*, set membership to Domain: `selina.world`. Use the domain administrator account credentials to authorize addition, then reboot.
* Log back into the **Mail** server as a Domain Administrator using domain formatting: `selina.world\administrator`.

#### 4. System Binary Extraction & Execution

* Locate the downloaded Exchange ISO, right-click and choose **Mount**.
* Copy all contents out of the virtual drive and create a localized target repository folder at `C:\Software\Exchange`, then paste the files.
* Right-click `Setup.exe` within that directory and select **Run as administrator**.
* Select the **Mailbox Role**, toggle **Automatically install Windows Server roles and features required**, and click next.
* For *Installation Space and Location*, alter path to point to your resilient RAID storage partition: `E:\Program Files\Microsoft\Exchange Server\V15`. Proceed with file checks, validation passes, and initiate installation. Reboot server on completion.

#### 5. Validation of Mail Flow via OWA

* Launch the **Exchange Management Shell (EMS)**. Open a secure browser session pointing to the control panel loopback via: `https://localhost/ecp`.
* Log in with `selina.world\administrator` credentials to check health inside the EAC workspace.
* On **DC1**, navigate to **Active Directory Users and Computers (ADUC)**. Create a new Organizational Unit (OU) named `Employees`.
* Inside the `Employees` OU, create two test accounts: `User A` and `User B`.
* In the **EAC** web console under **Mailboxes**, click **+ $\rightarrow$ Existing User**, click browse, map both `User A` and `User B`, and click save to generate active user mailboxes.
* Join your host workstations (**PC1** and **PC2**) to the `selina.world` domain and reboot.
* Log into **PC1** as `User A` and **PC2** as `User B`.
* Open browsers and navigate to **OWA (Outlook Web App)** via: `https://mail.selina.world/owa` or `https://192.168.1.20/owa`.
* Draft an internal email from `User A` and send it to `User B`. Check `User B`'s screen to verify receipt and confirm the core mail routing matrix is operational.

---

## Section 3: SSL/TLS Certificate Generation & Local PKI Deployment

### Concept Breakdown: Transport Security & Encryption

* **SSL/TLS (Secure Sockets Layer / Transport Layer Security):** Cryptographic protocols working at the presentation layer of the OSI model. They secure channels by encrypting communications moving between network endpoints (like an Outlook app or web browser talking to the Exchange server).
* **CSR (Certificate Signing Request):** An encrypted digital petition text file created by a web host server. It embeds server identity info and a public key, which must be verified and digitally signed by an authorized Certificate Authority.
* **AD CS (Active Directory Certificate Services):** A dedicated server role that builds an internal Public Key Infrastructure (PKI). It allows your network domain controller to act as a fully recognized root Certificate Authority (CA) to sign internal security certificates.

### Deployment Steps

#### 1. Generating the CSR on Exchange

* From the **EAC**, go to **Servers** $\rightarrow$ **Certificates** $\rightarrow$ click **+** to build a new Exchange Certificate wizard.
* Add your local access domain entries to register coverage: `mail.selina.world` and `autodiscover.selina.world`.
* Complete the wizard to output a raw text signing request bundle. Open the resulting `.req` or `.txt` file with Notepad and copy all string headers and body codes.

#### 2. Installing the Internal Root Certificate Authority

* Navigate back to **DC1** (or Server2 if designated for PKI).
* Go to **Add Roles and Features** $\rightarrow$ select **Active Directory Certificate Services (AD CS)**.
* Check the following sub-features for web enrollment capability:
* *Certification Authority*
* *Certification Authority Web Enrollment*
* *Certificate Enrollment Policy Web Service*
* *Certificate Enrollment Web Service*


* Click install, finish the wizard, and run the configuration prompt to bind services into your active forest hierarchy. Restart the machine.

#### 3. Requesting and Signing the Certificate via Web Enrollment

* Open a browser window to access the local enrollment landing page at `https://localhost/certsrv` (or from Mail server at `http://dc1/certsrv`).
* Select **Request a certificate** $\rightarrow$ **Submit a certificate request**. Paste your copied Exchange CSR text block into the Base-64 encoded request field.
* Select **Web Server** as the Certificate Template, click submit, and select **Download certificate** (saving it as `certnew.cer`).

#### 4. Completing the Certificate Lifecycle on Exchange

* Go to the **Exchange Management Console (MMC)** or the **EAC** workspace under Certificates. Select the pending request action and upload `certnew.cer` to finalize validation.
* Edit the properties of this active certificate file inside the EAC and check the box under **Services** to assign it to **IIS** (Internet Information Services) to activate HTTPS protection for OWA.

---

## Section 4: Enterprise Trust Orchestration via Group Policy (GPO)

### Concept Breakdown: Enterprise Trust Distribution

* **Group Policy Object (GPO):** A set of rules and automated settings deployed by network admins to configure user and computer systems globally across an active domain ecosystem.
* **Trusted Root Certification Authorities Store:** A highly protected system security partition where an OS keeps its lists of trusted digital roots. If a Certificate Authority's root file is not listed here, browsers will throw aggressive security block errors.
* **`gpupdate /force`:** A command-line utility used to manually bypass the default background time delays and force a machine to instantly pull down new GPO rule changes from the nearest Domain Controller.

### Deployment Steps

#### 1. Organizing the Target Assets

* On **DC1**, locate the exported CA root public key certificate. Move it to a dedicated repository folder structure at `C:\Certs\RootCert.cer`.

#### 2. Configuring Global Trust Policy

* Go to **Tools** $\rightarrow$ **Group Policy Management**. Right-click **Default Domain Policies** and choose **Edit**.
* Drills down into the policy tree mapping structure exactly as follows:
`Computer Configuration` $\rightarrow$ `Policies` $\rightarrow$ `Windows Settings` $\rightarrow$ `Security Settings` $\rightarrow$ `Public Key Policies` $\rightarrow$ `Trusted Root Certification Authorities`.
* Right-click inside the blank right pane space and select **Import**.
* Complete the path string targets wizard by pointing to `C:\Certs\RootCert.cer`. Save the settings and close the window.

#### 3. Enforcing and Validating Trust Deployment

* Open an administrative command prompt on user endpoints (such as **PC2**) and execute:
```cmd
gpupdate /force

```


* Open a browser on **PC2** and navigate out to `https://mail.selina.world/owa`. The site will now display a green padlock icon with zero security warning blocks, verifying that the domain-wide SSL implementation was deployed successfully!

---

## Section 5: Client Workspace Software Provisioning (Outlook Setup)

### Deployment Steps

#### 1. Configuring Server Deployment Shares

* On **Mail (Server1)**, navigate via File Explorer to your volume software storage directory containing your bulk setup binaries: `C:\Software\`.
* Right-click your Microsoft Office installation image directory file, go to properties, select **Sharing**, and enable access permissions so network workstations can reach it.

#### 2. Network Client Provisioning

* Log into **PC2**. Access the remote network file folder directory path directly from the Run box (`Win + R`):
```text
\\mail\f

```


* Locate and execute `setup64.exe` to deploy the unified email office workspace.
* Launch Outlook once setup finishes. The application will use **Autodiscover** records over HTTPS to automatically pull the current logged-in user context and map the matching exchange server configurations without requiring manual input. Perform a final system reboot to complete the lab!
