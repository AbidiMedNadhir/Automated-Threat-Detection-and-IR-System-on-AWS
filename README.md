<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0d1117,40:0a1628,100:003566&height=200&section=header&text=Automated%20Incident%20Response%20on%20AWS&fontSize=28&fontColor=ffffff&fontAlignY=40&desc=Cloud%20SOAR%20Pipeline%20%7C%20Detect%20%E2%80%A2%20Isolate%20%E2%80%A2%20Forensics%20%E2%80%A2%20Notify&descAlignY=62&descSize=14&descColor=4cc9f0&animation=fadeIn" width="100%" />

<br/>

![University](https://img.shields.io/badge/TEK--UP%20University-Tunis,%20Tunisia-4cc9f0?style=flat-square&logo=graduation-cap&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-2ea44f?style=flat-square&logo=checkmarx&logoColor=white)
![Type](https://img.shields.io/badge/Type-Cloud%20Security%20%2F%20SOAR-e63946?style=flat-square)
![Cloud](https://img.shields.io/badge/Cloud-Amazon%20Web%20Services-FF9900?style=flat-square&logo=amazon-aws&logoColor=white)
![License](https://img.shields.io/badge/Purpose-Educational%20Only-0078D4?style=flat-square)

</div>

---

## 📌 Overview

In modern cloud environments, the speed of reaction to an intrusion is a critical security factor. A **manual** incident response — detection, analysis, decision, action — can take anywhere from tens of minutes to several hours. During that window, an attacker can exfiltrate sensitive data, pivot laterally across the network, or compromise additional resources.

This project implements a **fully automated Incident Response (IR) pipeline on AWS**, following a **SOAR approach** (Security Orchestration, Automation and Response): detect a threat, contain the compromised resource, collect forensic evidence and alert the security team — **end-to-end, without human intervention, in under 90 seconds.**

Built as part of the Cloud Security module (Sécurité des Systèmes Informatiques et des Réseaux) — TEK-UP University, Class **SSIRF-5-A**.

---

## 💡 Why This Project Matters

Reactive, manual security operations don't scale — analysts can't watch every alert in real time, and attackers exploit exactly that delay. This project demonstrates how a small set of native AWS services can be wired together into an **autonomous containment system**, turning security monitoring into security *action*. It reflects the direction modern SOC and Cloud Security Engineering roles are heading: less manual triage, more orchestrated, code-defined response.

**Key engineering principles applied:**
- 🎯 **Event-driven design** — every step is triggered by the previous one, no polling, no manual steps
- 🔐 **Least privilege by default** — every IAM role is scoped to exactly what its component needs
- 🧊 **Forensics without re-exposure** — evidence is collected *after* isolation, over an API channel (SSM), never by reopening network access
- 📦 **Evidence integrity** — reports and snapshots are encrypted at rest and stored in a locked-down, private bucket

---

## 🎯 Objectives

| Goal | Detail |
|---|---|
| 🧠 **Detect** | Continuously monitor instance activity for anomalous behavior |
| 🔒 **Contain** | Automatically isolate the compromised EC2 instance at the network level |
| 🧪 **Investigate** | Collect forensic evidence (processes, connections, modified files, users) without opening SSH |
| 💾 **Preserve** | Snapshot the compromised disk for post-mortem / legal-grade analysis |
| 📊 **Report** | Generate a structured, timestamped incident report |
| 📧 **Alert** | Notify the security team in real time |
| ⏱️ **Perform** | Keep the full detect-to-contain cycle under 90 seconds |

---

## 🏗️ Architecture

<p align="center">
  <img src="architecture-diagram.png" alt="Automated Incident Response Architecture on AWS" width="100%" />
</p>

### How the components interact

The pipeline is organized in four layers that hand off to one another automatically:

**1️⃣ Telemetry & Detection**
The target EC2 instance generates three independent log streams — **VPC Flow Logs** (network traffic), **CloudTrail** (API activity) and **DNS Logs** (domain resolution). These feed into **CloudWatch Log Groups**, where **Metric Filters** continuously scan for suspicious patterns and feed a **CloudWatch Alarm**. The moment a threshold is breached, the alarm fires — this is the system's trigger point.

**2️⃣ Orchestration**
The alarm publishes an event to **Amazon EventBridge**, which acts as the control plane of the whole pipeline. EventBridge's only job is routing: it decouples *what was detected* from *what responds to it*, so the response logic can evolve independently of the detection logic.

**3️⃣ Automated Response (AWS Lambda)**
EventBridge invokes a single **Lambda function**, which runs three actions in sequence, each scoped to a dedicated IAM permission:
- `ec2:ModifySecurityGroups` → swaps the instance's Security Group for an **empty one** (no inbound, no outbound), cutting all network access instantly
- `ssm:SendCommand` → runs forensic collection scripts on the now-isolated instance **through the AWS API**, with no SSH port ever opened
- `ec2:CreateSnapshot` → captures an **EBS snapshot** of the compromised disk, freezing the exact state of the system at the moment of compromise

**4️⃣ Evidence, Notification & Audit**
The Lambda function writes the incident report, the forensic data and the snapshot reference to a **private S3 bucket**, encrypted at rest via **AWS KMS** with all public access blocked. It then publishes to an **SNS topic**, sending the security team an email with the instance ID, the detected threat, and a direct link to the report. Throughout the entire process, **CloudWatch** logs every action taken, turning the response itself into an audit trail.

```
EC2 Target ──► Flow Logs / CloudTrail / DNS Logs
                        │
                        ▼
         CloudWatch Log Groups + Metric Filters
                        │
                        ▼
              CloudWatch Alarm (trigger)
                        │
                        ▼
                   EventBridge
                        │
                        ▼
                  AWS Lambda (IR Engine)
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
  SG Isolation     SSM Forensics     EBS Snapshot
        │               │               │
        └───────────────┼───────────────┘
                        ▼
         S3 (private, KMS-encrypted) ──► SNS Email Alert
                        │
                        ▼
              CloudWatch Audit & Monitoring
```

---

## 🧩 Component Reference

| Component | Role |
|---|---|
| **Amazon EC2** (`t2.micro`, Amazon Linux 2) | Target instance, SSM Agent preinstalled |
| **VPC Flow Logs / CloudTrail / DNS Logs** | Raw telemetry feeding the detection layer |
| **Amazon CloudWatch** (Log Groups, Metric Filters, Alarms) | Continuous detection engine and the system's trigger |
| **Amazon EventBridge** | Orchestration — routes alarms to the response function |
| **AWS Lambda** (Python) | Core response engine: isolation, forensics, snapshotting |
| **AWS Systems Manager (SSM)** | Executes forensic commands over the AWS API — no SSH |
| **Amazon EBS Snapshot** | Freezes disk state for post-incident analysis |
| **Amazon S3** (private, SSE-KMS) | Stores incident reports, forensic data, snapshot references |
| **AWS KMS** | Encrypts/decrypts report data at rest |
| **Amazon SNS** | Real-time email alerts to the security team |
| **Amazon CloudWatch (Monitoring)** | End-to-end audit trail of every automated action |
| **AWS IAM** | Scoped, least-privilege roles per component |

---

## 🛡️ Security Controls

| Control | Purpose |
|---|---|
| Empty Security Group swap | Immediate, total network containment of the compromised host |
| SSM-only forensics (no SSH) | Evidence collection without ever reopening an attack surface |
| S3 Block Public Access + SSE-KMS | Confidentiality and integrity of all incident evidence |
| Least-privilege IAM per component | No wildcard policies, no shared roles |
| EventBridge-driven orchestration | Detection and response fully decoupled and independently testable |
| CloudWatch audit logging | Full traceability — every automated action is logged and reviewable |

---

## 🛠️ Tools & AWS Services

| Category | Services |
|---|---|
| **Compute** | Amazon EC2 |
| **Detection** | VPC Flow Logs, CloudTrail, DNS Logs, Amazon CloudWatch |
| **Orchestration** | Amazon EventBridge |
| **Response Engine** | AWS Lambda (Python) |
| **Forensics** | AWS Systems Manager (SSM RunCommand) |
| **Evidence Preservation** | Amazon EBS Snapshots |
| **Storage & Encryption** | Amazon S3, AWS KMS |
| **Notification** | Amazon SNS |
| **Monitoring & Audit** | Amazon CloudWatch |
| **Identity** | AWS IAM |

---

## 📈 Outcome

A compromised EC2 instance is detected, network-isolated, forensically captured and reported — **autonomously, in under 90 seconds**, with zero manual steps and a complete audit trail of every action taken.

---

## 📂 Repository Contents

- `architecture-diagram.png` — full pipeline architecture diagram
- `README.md` — this document

---

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/nadhir-abidi-95837325b)
[![Email](https://img.shields.io/badge/Gmail-Contact-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:abidimednadhir@gmail.com)
[![GitHub](https://img.shields.io/badge/GitHub-AbidiMedNadhir-161b22?style=for-the-badge&logo=github&logoColor=white)](https://github.com/AbidiMedNadhir)

---

<div align="center">
<img src="https://capsule-render.vercel.app/api?type=waving&color=0:003566,50:0a1628,100:0d1117&height=100&section=footer" width="100%" />
</div>
