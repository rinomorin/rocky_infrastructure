# Phase 1: the Infrastructure.
## Project Goal: The "Secure-by-Design" Cloud

### The Objective
The objective is to build a high-security, low-cost "Home Cloud" that replicates the enterprise experience of **OpenShift** or **AWS**. By using **Rocky Linux 10**, the infrastructure is designed to be **inherently secure** and **CIS-compliant** from the first second of boot. This ensures that any application developed or deployed within this environment is "production-ready" because it adheres to strict professional standards.

## The Mandate: "Security First" Philosophy

Our goal is to maintain a single, uncompromising security standard across every environment tier: Private Lab, Private Cloud, and Production. We operate under the principle of Environment Parity: if a security control is required for our most sensitive production environment, it is active in our labs from the first line of code. Development must always meet the same standards as production, whether we are coding to an API or deploying the applications we use. This includes full monitoring for every environment.

### The Knight Capital Lesson: The Cost of Prototyping Shortcuts
In 2012, Knight Capital Group evaporated $440 million and their entire reputation in just 45 minutes. "They didn't plan it;" they simply failed to maintain parity between their development and deployment states.

The Scenario: During a software update, they left "zombie" code from a decade-old prototype sitting on a single production server.

The Failure: Because the lab environment allowed this obsolete code to exist without modern safety checks, a simple "flag" triggered a catastrophic loop.

The Result: The system began buying and selling millions of shares in a blind loop, losing $172,000 every single second. This proves that "prototyping shortcuts" are just production vulnerabilities waiting for a trigger.

## The Strategy: Eliminating the "Prototyping-to-Secure" Trap
The traditional "Prototyping-to-Secure" transition is a fundamental flaw in engineering. We often tell ourselves we will "bolt on" security once the prototype works. This creates a "Security Shock" where hardened production settings break the application. Worse, it leaves the doors wide open for hackers to exploit; they don't just wait for a transition—they are always vigilant, searching for those doors to open.
 
### Hardening Against the Adversary: 
By starting in a secure environment, we aren't just passing audits; we are actively shrinking the attack surface. Hackers look for the very things we often overlook in prototypes: default passwords, nullok backdoors, and unmonitored root access. If these don't exist in the lab, they can't be exploited in the cloud.

### No Surprises: 
By hardening our authselect profiles and partition mount options now, we ensure our applications are born in the same "hostile-ready" environment where they will live.

### Start Secure, Stay Secure:
If we develop within a hardened baseline, we never have to "fix" security later. We move faster because we are already protected against both internal errors and external threats.

"We do not prototype in a vacuum. If it isn't secure in our Lab, it isn't ready for our Cloud. We don't 'add' security later; we build within it so that hackers find no foothold and deployment is a non-event."

The Practical Application
In the Lab and Cloud: We enforce high CIS levels (like even_deny_root and restricted fstab options) so we discover integration issues immediately, not during a high-stakes launch.

In the Cloud: Because we used production-level security in the lab, the transition to the Private Cloud is seamless.

### Against the Hacker: 
By eliminating "prototype-only" permissions and loose PAM configurations, we deny attackers the low-hanging fruit they rely on for initial access.

---
Our Network goals
<img width="2924" height="3700" alt="image" src="https://github.com/user-attachments/assets/b588d2ea-e477-4835-b3cd-2fbe89f6125a" />

---

## 1. Phase 1: Infrastructure Preparation

This stage focuses exclusively on establishing the core identity and security pillars before any Kubernetes nodes are provisioned. Standardizing these infrastructure components allows for **rapid recovery** in the event of a full system failure, as the entire environment can be redeployed to a known-secure state using Ansible playbooks.

### The Sequence of Trust: `ipa01` → `ipa02` → `helper`

Provisioning follows a strict order to build a chain of identity:

* **Step 1: `ipa01` (Root CA & Kerberos)**: Establishes the primary source of truth, providing the **Kerberos** realm for ticket-based authentication and the **Root Certificate Authority (CA)**.
* **Step 2: `ipa02` (Intermediate CA)**: Creates a redundant, enterprise-grade CA hierarchy by deploying a **ca-intermediate**. This isolates certificate issuance and mirrors professional data center architectures.
* **Step 3: `helper` (Cloud DNS & Provisioning)**: Manages the `cloud.home.example.com` zone and bridges the networks to ensure reliable service discovery.

### Core Security Features

* **Kerberos**: Enables ticket-based authentication, removing the need for static SSH passwords or keys between nodes.
* **LDAP (389/636)**: Centralizes the user directory via encrypted channels.
* **Active Directory (AD) Integration**: Allows for enterprise-style group policy and Role-Based Access Control (RBAC).
* **DNS (Integrated)**: Provides the backbone for service resolution within the future cluster.

---

## 2. Security Standards & "Immutable-ish" Expandability

A fundamental requirement of this project is that **SELinux remains enabled** at all times to maintain a hardened posture.

### Partitioning for CIS Level 2 Compliance

The LVM configuration utilizes a standardized template specifically designed to meet **CIS Level 2 security best practices**:

* **Dedicated Audit Volume**: A dedicated partition is allocated for `/var/log/audit` to ensure system events are captured without risking the root filesystem.
* **Service Isolation**: Dedicated volumes for `/tmp`, `/var/tmp`, and `/usr` prevent a single compromised service or runaway log from impacting the entire OS.

### Expandability & High Availability

The project treats infrastructure as **"Immutable-ish"** to support seamless expandability and maintenance:

* **Standardized Templates**: By using consistent, hardened templates, the system supports adding additional KVM hosts to share the load.
* **Preventative Maintenance & HA**: This design ensures that virtualized infrastructure nodes (IPA, Helper, Load Balancers) can move easily across KVM hosts. This mobility is critical for high availability, allowing VMs to be shifted for hardware patching, load balancing, or to recover quickly from a host failure without breaking the security or identity chain.

---

## 3. Infrastructure Task Summary

The initial playbooks act as both a builder and a security auditor:

* **Secure Baseline**: Every node starts with an **OpenSCAP** scan against the CIS Level 2 profile; failure to meet this baseline stops the build immediately.
* **Identity Enrollment**: Nodes are enrolled into the IPA realm to receive unique host principals and the CA trust chain.
* **Network Lockdown**: `firewalld` is locked to strictly necessary ports, and PAM is reconfigured with **faillock** to prevent brute-force attacks.

---
## 🚀 Playbook Overview
**setup_environment.yml:** The primary orchestrator. We use this to clear the environment, build the golden images, and prepare the helper nodes.

**config_infra.yml:** The post-deployment configuration, focusing on identity management (IPA) and site-specific service hardening.

## 🛠️ Role Breakdown
### 1. rocky_infra_golden
**pre-cleanup.yml:** Flushes "stuck" DHCP leases and destroys existing zombie VMs to ensure a clean workspace for the acme build.

**00_create_golden.yml:** Provisions the "Golden Image" VM from the Rocky ISO using customized Kickstart templates.

**01_wait_for_shutdown.yml:** Monitors installation progress and waits for the VM to power off (preventing the "fucking clone" hang issue).

**02_secure_golden.yml:** Applies security hardening and ensures FIPS-compliant configurations are baked into the base image.

**03_sysprep.yml:** Strips machine-specific IDs and logs, making the image generic and ready for cloning.

**04_write_json.yml:** Records metadata for the new image in our local state tracking.

### 2. rocky_server_deploy
**01_determine_os_identity.yml:** Matches target host requirements to the correct golden image version.

**pre_clone.yml:** Validates host storage and networking before starting the replication process.

**02_clone_to_guest.yml:** Executes the cloning of the golden image to create active infrastructure guests.

**set_guest_nic.yml:** Injects unique IP, MAC, and Hostname data into the cloned guest.

**post_clone.yml:** Finalizes the guest state and ensures it is reachable via the network.

**write_json.yml:** Updates the deployment report to reflect the new guest status.

### 3. rocky_end
**install_ipa.yml:** Orchestrates the installation of FreeIPA/IdM packages on the designated nodes.

**ipa_master.yml / ipa_replica.yml:** Configures the primary identity master and sets up high-availability replicas.

**ipa_service_check.yml:** Validates that Kerberos and LDAP services are fully operational.

**connections.yml:** Manages the internal routing and firewall rules required for IPA communication.

### 4. rocky_helper_prep
**pre_prep.yml:** Initial validation of the helper environment and host-level dependency checks.

**01_prep_nodes_ssh.yml:** Bootstraps SSH keys across the helper nodes to ensure seamless Ansible orchestration.

**02_prep_helper_python.yml:** Finalizes the setup by ensuring the Python environment is ready for the acme infra modules.

# 📂 Execution & State
**env_build.sh:** The master shell script used to trigger the entire end-to-end workflow.

ansible-execute: A wrapper script that passes consistent variables to our playbooks.

state/deployment_report.json: Our central database for tracking active VMs and safe-to-delete lists.
