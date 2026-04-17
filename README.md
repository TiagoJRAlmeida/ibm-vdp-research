# IBM VDP — Critical Vulnerability Research Report

A documented real-world vulnerability discovery carried out as part of the **Bug Bounty** final project for the *Theory and Practice of Security Attacks* course, Master's in Information Security at FCUP — Universidade do Porto.

The finding was submitted to IBM's Vulnerability Disclosure Programme (VDP) and acknowledged on HackerOne. This repository contains the full technical report detailing the discovery, exploitation chain, impact assessment, and remediation recommendations.

> **Note:** All credentials, keys, and internal endpoints present in the original report have been redacted prior to publication. No sensitive IBM data is reproduced here.

---

## Summary

During a structured bug bounty research exercise, an internet-facing IBM **webMethods Microservices Runtime** instance was identified with default administrative credentials left active on a publicly accessible development environment.

Initial access was gained through informed reasoning based on exposed operational metadata and known product defaults — not brute force. From there, a sequence of post-authentication steps ultimately led to the reading of `/proc/1/environ` via an inconsistency in the platform's file access control enforcement, exposing plaintext runtime secrets including AWS credentials, keystore passwords, and internal cluster configuration.

The vulnerability was assigned a **CVSS v3.1 score of 9.8 (Critical)**.

---

## Attack Chain Overview

```
1. Reconnaissance
   └── Subdomain enumeration (~352,000 IBM subdomains discovered)
       └── Identified high-signal target: IBM webMethods API Gateway (dev environment)

2. Initial Access
   └── Unauthenticated /metrics and /health endpoints confirmed service + stack
       └── Default credentials (Administrator / manage) granted full admin access

3. Post-Authentication Exploration
   ├── Dashboard enumeration: server properties, JVM metrics, package/service tree
   ├── Attempted OS command execution (pub.utils:executeOSCommand) — blocked by allowlist
   ├── Attempted runtime security policy modification via Extended Settings — reverted on restart
   ├── Attempted API Gateway REST interface access via pub.client:http — 401 Unauthorized
   └── Partial filesystem access via pub.file:getFile (JKS keystore read — binary, not extractable)

4. Critical Finding
   └── File access control enforcement inconsistency:
       pub.file:getFile bypassed path restrictions for /proc/1/environ
       └── Returned plaintext container environment variables including:
           ├── AWS access key + secret key
           ├── Administrative API Gateway credentials
           ├── Keystore and truststore passwords
           └── Internal cluster endpoints and Kubernetes metadata

5. Impact
   └── AWS credentials enabled authenticated access to IBM cloud infrastructure
       (S3 buckets, EC2 instances, OpenSearch, SageMaker, CloudWatch, and more)
   └── Testing stopped here — no further exploitation performed
   └── Disclosed responsibly under IBM's Safe Harbor Policy
```

---

## Vulnerability Details

| Field | Details |
|---|---|
| **Target** | IBM webMethods Microservices Runtime (internet-facing dev instance) |
| **Programme** | IBM Vulnerability Disclosure Programme (VDP) via HackerOne |
| **Root Cause** | Default administrative credentials + inconsistent file access control enforcement |
| **Attack Vector** | Network (no authentication required for initial access) |
| **CVSS v3.1 Score** | 9.8 — Critical |
| **CVSS Vector** | `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| **Disclosure** | Responsible disclosure; submitted to IBM VDP; Safe Harbor Policy followed |

---

## Key Technical Findings

**Default credential exposure.** The webMethods instance was reachable from the public internet and accepted the documented default `Administrator / manage` credential pair. This was not a brute force attack — the exposed `/metrics` endpoint confirmed the product and version, and the default credentials are publicly documented for this platform.

**Inconsistent file access control enforcement.** The Integration Server enforces filesystem restrictions through a path-based allowlist. However, this mechanism was not applied uniformly across all built-in file services. While `pub.file:copyFile` and `pub.file:moveFile` correctly blocked access to sensitive paths, `pub.file:getFile` did not — allowing the `/proc/1/environ` pseudo-file to be read despite lying outside any configured allowlist.

**Runtime secret exposure via `/proc/1/environ`.** In a containerised deployment, PID 1's environment variables contain all secrets injected at runtime. Reading this file yielded plaintext AWS credentials, API Gateway admin passwords, keystore and truststore passwords, and full Kubernetes pod metadata — collapsing the security boundary between the application layer and the underlying cloud infrastructure.

---

## Reconnaissance Methodology

The IBM program uses a wildcard scope (`*.ibm.com`), which required large-scale subdomain enumeration. Tools used across the reconnaissance phase included:

- **Subdomain enumeration:** `subfinder`, `findomain`, `assetfinder`, `chaos`, `waybackurls` — combined output of ~352,000 unique subdomains
- **Live host filtering:** `httpx` — reduced to ~2,000 responsive hosts
- **Technology fingerprinting:** `httpx`, `nuclei`, `nmap`, `Wappalyzer`
- **Endpoint discovery:** `katana` (web crawler), `ffuf` (directory fuzzing)
- **Traffic visualization:** Burp Suite site map (populated via `httpx` proxy)

A key lesson from earlier programs (UKG, Kong) was that narrow scopes reduce attack surface and that aggressive filtering during host triage can eliminate valuable targets. Switching to a broader wildcard scope and clustering rather than discarding redirect chains proved decisive.

---

## Impact Assessment

A single file-read operation against `/proc/1/environ` yielded credentials capable of:

- Authenticating directly to AWS-managed services (OpenSearch, S3, EC2)
- Enumerating and accessing dozens of S3 buckets (backups, DR replicas, analytics, image registries, logging pipelines)
- Identifying production, staging, and development compute resources
- Decrypting internal TLS communications via exposed keystore passwords
- Impersonating the API Gateway to internal cluster components

No customer data was accessed, modified, or exfiltrated. Testing was halted upon confirming the impact and all findings were disclosed to IBM under their Safe Harbor Policy.

---

## Report

The full technical report is included in this repository: [`report.pdf`](./report.pdf)

It covers the complete methodology, reconnaissance process, exploitation steps, CVSS assessment, and detailed remediation recommendations.

---

## Authors

- Tiago Almeida — [github.com/TiagoJRAlmeida](https://github.com/TiagoJRAlmeida)
- Gabriel Ferreira

*Supervised by Prof. André Baptista and Miguel Regala, FCUP.*
