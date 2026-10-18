# PURA Drive — Secure Internal File Storage & Security Monitoring Platform

> **Zero Trust | Network Segmentation | IAM | Encryption | Malware Protection | SIEM | DevSecOps**

PURA Drive is a **secure internal file storage and collaboration platform** designed with a security-by-design approach.

The project combines **network segmentation, identity and access management, encrypted object storage, automated malware scanning, centralized logging, security monitoring, and DevSecOps controls** into a single architecture.

This repository documents the **architecture, security mechanisms, workflows, demonstrations, and design decisions** of the project.  
The source code is intentionally not published.

---

## 📌 Project Overview

Traditional file-sharing platforms can expose sensitive information to several risks, including:

- Unauthorized access
- Malware uploaded through files
- Excessive privileges
- Data exposure
- Lateral movement after a network compromise
- Insufficient auditability
- Lack of centralized security monitoring

PURA Drive addresses these risks by implementing security controls at multiple layers:

- **Network isolation** through dedicated VLANs
- **Centralized authentication** with Keycloak
- **Multi-factor authentication (TOTP)**
- **Role-Based Access Control (RBAC)**
- **Encrypted file storage**
- **Asynchronous antivirus scanning**
- **Centralized logs and security events**
- **Infrastructure and security monitoring**
- **Restricted administration through WireGuard**
- **Automated security checks in CI/CD**

---

## 🎯 Project Objectives

The main objectives of PURA Drive are to:

- Secure the storage of sensitive files
- Control user authentication and authorization
- Isolate application, data, and administration layers
- Encrypt files at rest
- Detect malicious files automatically
- Provide visibility into security and application events
- Monitor infrastructure and security metrics
- Protect administrative interfaces
- Integrate security controls into the development lifecycle

---

# 🏗️ Architecture

The platform follows a multi-layer architecture based on **network segmentation and security by design**.

![Global Architecture](architecture/global-architecture.png)

### Main architectural layers

| Layer | Main Components | Purpose |
|---|---|---|
| DMZ | Nginx | External entry point, TLS termination and traffic routing |
| Application | Next.js, NestJS, Keycloak, Redis | Application logic, authentication and asynchronous processing |
| Data | PostgreSQL, MinIO, KES, Vault | Application data, encrypted object storage and key management |
| Administration | WireGuard | Restricted access to administration services |

The architecture is divided into **four isolated VLANs** to enforce traffic boundaries between different security zones.

---

# 🌐 Network Segmentation

PURA Drive uses a **4-VLAN architecture** to reduce the attack surface and limit lateral movement.

![Network Segmentation](architecture/network-segmentation.png)

### VLAN 10 — DMZ

The DMZ contains the **Nginx reverse proxy**, which acts as the main entry point to the platform.

Responsibilities include:

- HTTPS termination
- HTTP → HTTPS redirection
- Security headers
- Request routing
- Traffic control

---

### VLAN 20 — Application Zone

The application layer contains services responsible for:

- Web interface
- Backend API
- Authentication integration
- Session management
- Asynchronous processing

Main technologies:

- Next.js
- NestJS
- Keycloak
- Redis
- Bull Queue

---

### VLAN 30 — Data Zone

The data zone contains sensitive storage and cryptographic services.

Main components:

- PostgreSQL
- MinIO
- KES
- HashiCorp Vault

The separation of this zone prevents direct exposure of storage and key-management services to the external network.

---

### VLAN 99 — Administration Zone

Administrative access is restricted through a **WireGuard VPN tunnel**.

This provides an additional security boundary for administrative operations and monitoring interfaces.

---

# 🔐 Identity & Access Management

PURA Drive uses **Keycloak** as the central Identity and Access Management component.

![Authentication Flow](architecture/authentication-flow.png)

### Authentication

The platform uses:

- OpenID Connect
- Single Sign-On
- TOTP-based Multi-Factor Authentication
- Role-Based Access Control

The authentication flow separates identity management from the application itself.

### Authorization

Keycloak roles are used to control access to platform resources and administrative capabilities.

---

## 🔑 Multi-Factor Authentication

TOTP-based MFA adds an additional authentication factor beyond the user's password.

The objective is to reduce the impact of compromised credentials.

**Demo:**

![Login](demos/01-login/login.png)

![MFA](demos/01-login/mfa.png)

---

# 🔒 Secure File Storage

One of the core security mechanisms of PURA Drive is **server-side encryption using SSE-KMS**.

The encryption architecture combines:

- **MinIO**
- **KES**
- **HashiCorp Vault**

![Encryption Architecture](architecture/encryption-architecture.png)

---

## 🔐 Envelope Encryption

PURA Drive uses an envelope encryption model with two levels of keys:

### Key Encryption Key — KEK

The KEK is stored in **HashiCorp Vault**.

It is used to protect the Data Encryption Keys.

The KEK never directly encrypts the file content.

### Data Encryption Key — DEK

KES generates a unique DEK for each file.

The DEK encrypts the file using:

**AES-256-GCM**

The plaintext DEK exists only temporarily in memory during the cryptographic operation.

---

## 🔗 MinIO → KES → Vault

The encryption chain can be summarized as:

```text
User File
    │
    ▼
  MinIO
    │
    │ SSE-KMS
    ▼
   KES
    │
    │ mTLS
    ▼
 HashiCorp Vault
    │
    ▼
   KEK
```

MinIO stores the encrypted object while cryptographic key management is delegated to KES and Vault.

Communication between MinIO and KES is protected using **mutual TLS (mTLS)**.

---

# 📤 Secure File Upload

PURA Drive uses a direct-to-object-storage upload mechanism.

![File Upload Flow](architecture/file-upload-flow.png)

The general workflow is:

```text
Browser
   │
   │ Request upload URL
   ▼
NestJS API
   │
   │ Generate presigned URL
   ▼
Browser
   │
   │ Upload file
   ▼
MinIO
   │
   │ Request encryption operation
   ▼
KES
   │
   ▼
Vault
```

The file does not need to pass through the NestJS backend during the actual object upload.

This reduces unnecessary data transfer through the application layer.

---

# 🦠 Malware Detection

Uploaded files are processed through an **asynchronous antivirus scanning pipeline**.

The solution uses:

- ClamAV
- Bull Queue
- Redis
- MinIO

![Antivirus Flow](architecture/antivirus-flow.png)

### Scanning workflow

```text
User Upload
     │
     ▼
   MinIO
     │
     ▼
 Scan Job
     │
     ▼
 Bull Queue / Redis
     │
     ▼
   ClamAV
     │
     ├───────────────┐
     │               │
   CLEAN          INFECTED
     │               │
     ▼               ▼
 File available    File deleted
     │               │
     ▼               ▼
 Database status   Security event
                     │
                     ▼
                 Admin alert
```

The scan is performed asynchronously so that antivirus processing does not block the upload experience.

---

# 🧪 Malware Detection Demo

The repository includes screenshots demonstrating the complete detection workflow.

### 1. Upload

![Upload Infected File](demos/03-malware-detection/upload-infected-file.png)

A file is uploaded through the user interface.

### 2. Scan Status

![Scan Status](demos/03-malware-detection/scan-status.png)

The platform displays the scanning status of the uploaded file.

### 3. ClamAV Detection

![ClamAV Detection](demos/03-malware-detection/clamav-detection.png)

ClamAV analyzes the file and identifies the malicious content.

### 4. File Removal & Alert

When a file is detected as infected:

- The file is removed from storage
- Its database status is updated
- The user is notified
- An administrative security alert is generated

![Admin Alert](demos/03-malware-detection/admin-alert.png)

> **Note:** No real malware sample is included in this repository. The demonstration only documents the detection workflow and its results.

---

# 📁 File Management

PURA Drive provides a user interface for managing stored files.

Supported operations include:

- File upload
- File download
- File deletion
- Folder management
- File search
- Filtering
- List view
- Grid view

### File Dashboard

![Dashboard](demos/02-file-management/dashboard.png)

### File List

![File List](demos/02-file-management/file-list.png)

### File Grid

![File Grid](demos/02-file-management/file-grid.png)

### Folder Management

![Folders](demos/02-file-management/folders.png)

---

# 👥 Team Collaboration

PURA Drive also provides team workspaces for collaborative file management.

Team functionality includes:

- Team creation
- Member management
- File sharing
- Owner / Member roles
- Team workspace
- Per-file comments

![Teams](demos/04-team-workspace/teams.png)

![Members](demos/04-team-workspace/members.png)

---

# 📊 Security Monitoring & SIEM

PURA Drive integrates the **ELK Stack** to centralize and visualize application, audit, and security logs.

The monitoring architecture provides visibility into:

- User activities
- Security events
- Authentication events
- File-related events
- Malware detections
- Administrative activities

![Logging Architecture](architecture/logging-architecture.png)

---

## 🔎 Kibana

Kibana is used to search and visualize centralized logs.

### Kibana Dashboard

![Kibana Dashboard](demos/06-kibana/kibana-dashboard.png)

### Security Events

![Security Events](demos/06-kibana/security-events.png)

### Audit Logs

![Audit Logs](demos/06-kibana/audit-logs.png)

The objective is to provide centralized visibility for investigation and security monitoring.

---

# 📈 Infrastructure Monitoring

Prometheus and Grafana are used for monitoring infrastructure and application metrics.

![Monitoring Architecture](architecture/monitoring-architecture.png)

The monitoring layer provides dashboards for:

- Infrastructure health
- Application metrics
- Performance indicators
- Security-related metrics

### Grafana

![Infrastructure Dashboard](demos/07-grafana/infrastructure.png)

![Security Metrics](demos/07-grafana/security-metrics.png)

---

# 🛡️ Administration & Security

The platform includes an administration dashboard for managing users and monitoring security events.

![Admin Dashboard](demos/05-admin-dashboard/overview.png)

Administrative access is additionally restricted through the **WireGuard VPN**.

This creates an additional security boundary around sensitive administration functions.

---

# ⚙️ DevSecOps

Security controls are integrated into the development lifecycle through a GitHub Actions CI/CD pipeline.

![DevSecOps Pipeline](architecture/devsecops-pipeline.png)

The security pipeline includes:

### 🔍 Gitleaks

Used for detecting exposed secrets in the codebase.

### 🧹 Semgrep

Used for static security analysis.

### 🕷️ OWASP ZAP

Used for dynamic application security testing.

The objective is to identify security issues as early as possible in the development lifecycle.

---

# 🧰 Technology Stack

### Application

- Next.js
- NestJS
- TypeScript

### Identity & Access Management

- Keycloak
- OpenID Connect
- TOTP
- RBAC

### Storage & Data Protection

- MinIO
- KES
- HashiCorp Vault
- AES-256-GCM
- PostgreSQL

### Security

- ClamAV
- WireGuard
- Nginx
- TLS
- mTLS

### Observability

- ELK Stack
- Kibana
- Prometheus
- Grafana

### Infrastructure

- Docker
- Docker Compose
- Redis
- Bull Queue

### DevSecOps

- GitHub Actions
- Gitleaks
- Semgrep
- OWASP ZAP

---

# 🔄 Security Workflows

## Authentication

```text
User
 │
 ▼
Nginx
 │
 ▼
Keycloak
 │
 ├── OIDC
 ├── TOTP MFA
 └── RBAC
 │
 ▼
PURA Drive
```

## File Upload

```text
Browser
   │
   ▼
NestJS
   │
   ▼
Presigned URL
   │
   ▼
MinIO
   │
   ▼
KES + Vault
   │
   ▼
Encrypted Object
```

## Malware Detection

```text
MinIO
  │
  ▼
Bull Queue
  │
  ▼
ClamAV
  │
  ├── Clean ───► File available
  │
  └── Infected ─► Delete + Alert
```

## Security Monitoring

```text
Applications
     │
     ▼
   Logs
     │
     ▼
    ELK
     │
     ▼
  Kibana
     │
     ▼
Security Investigation
```

---

# 🧩 Security Principles

The architecture follows several security principles:

### Defense in Depth

Security controls are distributed across multiple layers rather than relying on a single mechanism.

### Least Privilege

Access to services and cryptographic operations is restricted according to their role.

### Network Segmentation

Application, data, and administration services are isolated into dedicated network zones.

### Zero Trust Approach

Access is controlled through identity, authorization, network boundaries, and explicit security controls.

### Security by Design

Security requirements are considered throughout the architecture rather than being added only after implementation.

### Separation of Duties

Application services, object storage, and cryptographic key management are separated into distinct components.

---

# 📂 Repository Structure

```text
pura-drive-secure-storage/
│
├── README.md
│
├── architecture/
│   ├── global-architecture.png
│   ├── network-segmentation.png
│   ├── authentication-flow.png
│   ├── encryption-architecture.png
│   ├── file-upload-flow.png
│   ├── antivirus-flow.png
│   ├── logging-architecture.png
│   ├── monitoring-architecture.png
│   └── devsecops-pipeline.png
│
├── demos/
│   ├── 01-login/
│   ├── 02-file-management/
│   ├── 03-malware-detection/
│   ├── 04-team-workspace/
│   ├── 05-admin-dashboard/
│   ├── 06-kibana/
│   └── 07-grafana/
│
├── docs/
│   ├── 01-project-overview.md
│   ├── 02-requirements.md
│   ├── 03-architecture.md
│   ├── 04-network-security.md
│   ├── 05-identity-access-management.md
│   ├── 06-secure-storage.md
│   ├── 07-antivirus-pipeline.md
│   ├── 08-observability-siem.md
│   ├── 09-devsecops.md
│   └── 10-security-considerations.md
│
└── security/
    ├── threat-model.md
    ├── security-controls.md
    ├── attack-surface.md
    └── mitigations.md
```

---

# 🔒 Security & Privacy Notice

This repository intentionally **does not contain the application source code**.

The following must never be committed:

- Passwords
- API keys
- Access tokens
- Private keys
- Certificates containing private material
- `.env` files containing secrets
- Real user data
- Production logs
- Sensitive infrastructure credentials
- Real malware samples

Screenshots and diagrams should be sanitized before publication.

---

# 📚 Project Documentation

Detailed documentation is available in the [`docs/`](docs/) directory.

Recommended reading order:

1. [Project Overview](docs/01-project-overview.md)
2. [Requirements](docs/02-requirements.md)
3. [Architecture](docs/03-architecture.md)
4. [Network Security](docs/04-network-security.md)
5. [Identity & Access Management](docs/05-identity-access-management.md)
6. [Secure Storage](docs/06-secure-storage.md)
7. [Antivirus Pipeline](docs/07-antivirus-pipeline.md)
8. [Observability & SIEM](docs/08-observability-siem.md)
9. [DevSecOps](docs/09-devsecops.md)
10. [Security Considerations](docs/10-security-considerations.md)

---

# 📄 PFE Report

The complete academic project report provides the detailed technical design, implementation process, architecture decisions, security mechanisms, and project methodology.

**Full report:** `docs/PFE_Report.pdf`

---

# 🎓 Academic Project

**Project:** PURA Drive — Secure Internal File Storage & Collaboration Platform

**Institution:** ENET'COM — Sfax, Tunisia

**Field:** Telecommunications Engineering — Telecommunications Technologies and Services

**Project Focus:**

> Secure Storage · Network Segmentation · IAM · Encryption · Malware Detection · SIEM · Observability · DevSecOps

---

# 👩‍💻 Author

**Oumayma Selmi**

Telecommunications Engineering Graduate — Cybersecurity specialization

- LinkedIn: [Oumayma Selmi](https://www.linkedin.com/in/oumayma-selmi-74b3ab284/)
- GitHub: [aumayma](https://github.com/aumayma)

---

# ⭐ Project Focus

This project demonstrates how a secure file-storage platform can combine:

**Identity → Network Security → Data Protection → Malware Detection → Monitoring → DevSecOps**

into a defense-in-depth architecture.

The repository focuses on the **security architecture and technical implementation approach** rather than exposing the application source code.
