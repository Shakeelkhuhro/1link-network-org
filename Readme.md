# 1LINK Network Core & Security Provisioning Pipeline

This repository hosts the end-to-end GitOps automation engine for the **1LINK Network Automation & Change Management Pipeline**. The system monitors local iTop ITSM tickets, processes human-in-the-loop approvals via AWX, validates network states, and provisions multi-layer configurations across Cisco infrastructure before closing out database records safely.

---

## 🛠️ Repository Architecture & Playbooks

The pipeline is split into modular playbooks to ensure clean execution bounds within the AWX Workflow Canvas:

* **`playbooks/itop_ingestion.yml` (Phase 1):** Connects to the local iTop REST API, pulls down tickets marked as `resolved`, parses raw description strings using regex, extracts variables, and bridges data downstream using `set_stats`.
* **`playbooks/validate_network_state.yml` (Phase 2):** Connects to infrastructure to verify state reality. It safely aborts execution if a user requests a resource that already exists or tries to delete a resource that is missing.
* **`playbooks/cisco_switch_execution.yml` (Phase 3 & 4):** The heavy-lifting configuration engine. Handles VLAN modification, port manipulation, interface enablement, static routes, ACL injection, OSPF/BGP dynamic routing, and global baseline compliance.
* **`playbooks/itop_closure.yml` (Phase 4):** Serializes operational runtime logs into a native JSON structure via the `| to_json` filter and pushes updates back to iTop to permanently close the tickets.
* **`playbooks/send_notification.yml` (Phase 6):** Assembles a clean HTML table summarizing the execution parameters and emails it out to the Network Operations team.

---

## 🧭 How the iTop Ingestion Engine Parses Keywords

The ingestion engine enforces strict architectural boundaries to avoid accidental network disruptions caused by typos or messy formatting:

1.  **Keyword Precedence Guardrail:** The parser looks for the keyword `delete` before analyzing the word `vlan`. This stops a request like *"delete vlan 100"* from accidentally triggering a creation routine.
2.  **Contiguous Search Pattern:** The engine scans the text block using a strict contiguous sequence (`regex_search('[0-9]+')`). This keeps variations like `vlan 702`, `vlan   702`, or `vlan#702` from breaking the automation string logic.
3.  **Interface Text Mapping:** A lowercase translation regex looks for shorthand keywords such as `gi1/0/1`, `gigabitethernet1/0/5`, or `fastethernet0/2` to automatically locate the correct hardware interface.

---

## 📋 Simple Use Case Testing Guide

To test the entire range of operations, create standard `UserRequest` tickets in your iTop instance with a status of `resolved` using these exact string templates:

### Use Case 1: Standard VLAN Creation & Port Map
* **iTop Ticket Description:** `"Please deploy network change to create vlan 702 and assign to access port gi1/0/1."`
* **How Engine Processes It:** Extracts `change_action: create`, `target_vlan_id: 702`, and `target_interface: GigabitEthernet1/0/1`.
* **Result:** VLAN 702 is built on the core switch, port Gi1/0/1 is opened, and it is converted into an active access port for VLAN 702.

### Use Case 2: Structural Network Asset Teardown
* **iTop Ticket Description:** `"Decommission old segment. Run routine to delete vlan 702 immediately."`
* **How Engine Processes It:** Matches the `delete` keyword first. Sets `change_action: delete` and identifies the target ID as `702`.
* **Result:** The script strips VLAN 702 from the configuration database and returns the affected ports to baseline configurations.

### Use Case 3: Interface Shut Routine (Maintenance)
* **iTop Ticket Description:** `"Emergency lock down required. Shut interface gi1/0/1 on vlan 720 due to malicious activity."`
* **How Engine Processes It:** Finds the interface designator, matches the word `shut` while verifying that `no shut` is missing, and flips the target state tracker to disabled.
* **Result:** Interface `GigabitEthernet1/0/1` is given an administrative shutdown command, and an automated tracking note is attached to the interface configuration.

### Use Case 4: Interface Revival Routine (Bring Up)
* **iTop Ticket Description:** `"Maintenance window closed. Bring up interface gi1/0/1 and run no shut."`
* **How Engine Processes It:** Matches the target interface and catches the phrase `no shut` to override the basic shutdown logic, marking the state tracker as enabled.
* **Result:** The interface execution module issues an administrative `no shutdown` to restore active transit paths.

### Use Case 5: Infrastructure Hardening & Dynamic Routing Sync
* **iTop Ticket Description:** `"Run system compliance sweeps and ensure hardening profiles are active on vlan 705."`
* **How Engine Processes It:** Sets standard creation variables for safety parameters, parses internal dynamic routing maps, and configures peer boundaries.
* **Result:** The playbook synchronizes the device back to baseline standards—re-enforcing AAA profiles, correcting centralized syslog servers, checking OSPF/BGP neighbor groups, and ensuring idle shells close automatically.
