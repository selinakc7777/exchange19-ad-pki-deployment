# Enterprise Identity & Mail Infrastructure Lab
In this project, I built a mock corporate network environment under the internal domain **`selina.world`**.

This guide walks through setting up **Active Directory (the network's brain)**, deploying **Exchange Server 2019 (the mail system)**, and configuring **SSL/TLS Certificates (the security layers)** to allow secure email routing between client machines.

---

## Network Blueprint (Topology)

Before diving in, here is how the lab network is mapped out:

| Device Name | Role (What it does) | Operating System | IP Address | Primary DNS |
| --- | --- | --- | --- | --- |
| **Edge Router** | Connects the lab to the Internet | - | DHCP (Automatic) | Auto |
| **DC1** | Domain Controller (Active Directory & DNS) | Windows Server | `192.168.1.10` | `192.168.1.10` |
| **Mail** | Mail Server (Exchange Server 2019) | Windows Server | `192.168.1.20` | `192.168.1.10` |
| **PC1 / PC2** | Employee Workstations | Windows 10/11 | DHCP (Automatic) | `192.168.1.10` |

---

## Deep-Dive
### 1. Why do we need Active Directory first?

Think of Active Directory (AD) as the single source of truth for the entire company.

* **The Problem:** Without AD, if a company has 500 employees, you would have to manually create 500 usernames and passwords on *every single server and computer* individually.
* **The Solution:** Active Directory centralizes this. Microsoft Exchange Server **cannot function without Active Directory** because Exchange doesn't keep its own list of users. When you create a mailbox, Exchange attaches that mailbox directly to an existing Active Directory user account.

### 2. Why do we need Storage Design (RAID 5)?

In a business environment, if a hard drive containing company emails crashes, the company loses money and critical data.

* **The Solution:** We implement **RAID 5** (Redundant Array of Independent Disks). RAID 5 takes three or more hard drives and splits data across them along with "parity" bits (mathematical backup data). If any single hard drive physically dies, the server keeps running without missing a beat, calculating the missing data on the fly until the broken drive is replaced.

### 3. Why do we need SSL/TLS Certificates?

When you log into Outlook Web App (OWA), your browser sends your username and password across the network to the Mail server.

* **The Problem (Clear Text):** By default, this data is sent in plain text. Anyone running a simple packet-sniffing tool on the same network can steal your password.
* **The Solution:** An SSL/TLS certificate creates an encrypted tunnel between the user's browser and the Exchange server. Even if someone intercepts the network traffic, it looks like unreadable gibberical code.

### 4. Why do we need our own Certificate Authority (AD CS)?

* **The Problem:** Normally, companies buy SSL certificates from public vendors like DigiCert or GoDaddy. However, public vendors *will not* issue certificates for internal, private domains (like our lab's `selina.world`).
* **The Solution:** We build our own **Active Directory Certificate Services (AD CS)** server. This turns our domain controller into a local "GoDaddy." It allows our company to sign its own certificates for internal servers completely for free.

### 5. Why do we use Group Policy (GPO) for trust?

* **The Problem:** If you generate a certificate from a custom, home-built Certificate Authority, Windows computers will reject it by default, throwing an aggressive **"NOT SECURE / UNTRUSTED CA"** warning. This is because Windows doesn't know who your local CA is.
* **The Solution:** Instead of walking over to every employee's computer to manually install the certificate root, we use a **Group Policy Object (GPO)**. The GPO says: *"Attention all computers in the domain, our company built this CA. You must instantly trust it."* This automates the trust deployment silently across thousands of devices.

---

## Step-by-Step Lab Setup

### Phase 1: Setting up the Core Network & Active Directory

First, we need to create our domain controller (**DC1**) so our other computers have a network to join.

1. **Router Configuration:** Ensure the edge router's WAN interface is set to **DHCP** so the lab can talk to the internet.
2. **Set Static IP on DC1:** Change the network adapter properties of your main server to IP: `192.168.1.10` and DNS: `192.168.1.10`.
3. **Rename Server:** Rename the computer to **`DC1`** and restart it.
4. **Install Active Directory:** Open **Server Manager** $\rightarrow$ Click **Add Roles and Features** $\rightarrow$ Select **Active Directory Domain Services (AD DS)** and hit Install.
5. **Promote the Server:** Once installed, click the yellow flag notification at the top of Server Manager and select **Promote this server to a domain controller**.
6. **Create the Forest:** Select **Add a new forest** and type your root domain name: `selina.world`. Complete the wizard and let the server restart.

---

### Phase 2: Installing Exchange Server 2019

Now we will set up our dedicated mail server (**Mail**) and connect it to our domain.

1. **Configure Mail Server Network:** On your second server, set a static IP of `192.168.1.20`. Set the Primary DNS to `192.168.1.10` (pointing to DC1). Rename this server to **`Mail`** and restart.
2. **Prepare Resilient Storage:** Exchange needs dedicated storage. On **DC1**, open **Disk Management**, initialize your extra raw hard drives, and format them as a **New RAID 5 Volume** mapped to drive letter `E:` with the label `Exchange`.
3. **Join Mail Server to Domain:** On the **Mail** server, open system properties (`sysdm.cpl`), click *Change*, choose *Domain*, type `selina.world`, and log in with your domain administrator credentials. Restart the machine.
4. **Extract Exchange Files:** Log back into the Mail server as `selina.world\administrator`. Mount your downloaded Exchange 2019 ISO, copy all files, and paste them into a local folder at `C:\Software\Exchange`.
5. **Run Exchange Installation:** Right-click `Setup.exe` $\rightarrow$ Select **Run as administrator**. Follow the wizard:
* Choose the **Mailbox Role**.
* Check **Automatically install Windows Server roles**.
* Change the installation path to your custom RAID volume: `E:\Program Files\Microsoft\Exchange Server\V15`.


6. **Create Test Accounts:** Once installed, go back to **DC1**, open **Active Directory Users and Computers**, create a new folder (Organizational Unit) named `Employees`, and create two users: `User A` and `User B`.
7. **Provision Mailboxes:** Log into the **Exchange Admin Center (EAC)** at `https://localhost/ecp`. Go to *Mailboxes* $\rightarrow$ Click **+** $\rightarrow$ *Existing User* $\rightarrow$ Select `User A` and `User B`.
8. **Test Local Mail flow:** Connect client machines **PC1** and **PC2** to the domain. Log into Outlook Web App (OWA) at `https://mail.selina.world/owa` and verify that User A and User B can successfully send emails back and forth.

---

### Phase 3: Creating and Installing SSL Certificates

By default, browsers will show a red "Not Secure" warning when accessing OWA. We will build an internal Certificate Authority to fix this.

1. **Generate the CSR:** In the Exchange Admin Center (EAC), go to **Servers** $\rightarrow$ **Certificates** $\rightarrow$ Click **+**. Add `mail.selina.world` and `autodiscover.selina.world` to the request. Save the text file output and copy the long string of code inside it.
2. **Install Certificate Authority Role:** On **DC1**, go to **Add Roles and Features** and install **Active Directory Certificate Services (AD CS)** along with *Certification Authority Web Enrollment*. Restart.
3. **Sign the Certificate:** Open a browser and go to `http://dc1/certsrv`. Select **Request a certificate** $\rightarrow$ **Submit an advanced request**. Paste your copied CSR text code block, change the template to **Web Server**, and click Submit. Download the resulting file as `certnew.cer`.
4. **Complete the Request:** Go back to the EAC Certificates tab, complete the pending request by uploading `certnew.cer`, and edit its properties to assign it to the **IIS** (web) service.

---

### Phase 4: Automating Trust with Group Policy

Even though the certificate is installed, client computers don't trust it yet. We use Group Policy to force all workstations to trust our root certificate automatically.

1. **Export Root Certificate:** Copy your CA's root certificate file and save it to a central folder on DC1 at `C:\Certs\RootCert.cer`.
2. **Configure Group Policy:** Open **Group Policy Management** $\rightarrow$ Right-click **Default Domain Policies** $\rightarrow$ Click **Edit**.
3. **Navigate to Public Keys:** Drill down into:
`Computer Configuration` → `Policies` → `Windows Settings` → `Security Settings` → `Public Key Policies` → `Trusted Root Certification Authorities`.
4. **Import Certificate:** Right-click in the empty space, select **Import**, and choose `C:\Certs\RootCert.cer`. Save and close.
5. **Force Update on Clients:** Go to your workstation (**PC2**), open the Command Prompt (`cmd`), and type the following command to update security rules instantly:
```cmd
gpupdate /force

```


6. **Final Verification:** Open a browser on PC2 and visit `https://mail.selina.world/owa`. You will see a clean green padlock with absolutely no security warnings. Success!

---

### Phase 5: Setting Up Outlook for End Users

1. **Share Office Media:** On **Mail (Server1)**, right-click your Office software installation folder, click *Properties*, and share it across the network.
2. **Install Office on Workstation:** On **PC2**, press `Win + R`, type your server path string: `\\mail\f`, and run `setup64.exe`.
3. **Launch Outlook:** Open Outlook. Thanks to the network's internal **Autodiscover** settings running safely over HTTPS, Outlook will automatically find the user's email profile and connect without any manual setup!

---
