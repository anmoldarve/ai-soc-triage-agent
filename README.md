# Junior SOC Analyst — AI-Powered Threat Triage Agent

## 1. Project Overview

This project extends a prior SOC AI Agent Pipeline build by wiring a live Python/tshark network monitor into a purpose-built AI triage agent hosted on the Airia platform. The goal was to simulate a lightweight, junior-analyst-style SOC workflow: capture raw traffic, detect a simple volumetric anomaly, package it as a structured alert, and hand it off to an LLM-based agent that produces a professional triage report — classification, MITRE ATT&CK mapping, confidence level, and recommended next actions.

**Tech Stack**

- **Airia Studio** — AI agent orchestration platform (project, model, agent, publishing, API keys)
- **OpenAI GPT-5 Nano** — underlying LLM for the triage agent (`gpt-5-nano-2025-08-07`)
- **Python 3** — `soc_monitor.py`, capture-to-alert automation script
- **tshark / pcap** — live traffic capture and CSV conversion
- **Kali Linux** (attacker/monitoring host) + Wazuh host — lab environment
- **REST API** (curl / requests) — alert delivery to the Airia agent endpoint

## 2. Pipeline Architecture

The pipeline has two halves that meet at a single HTTPS call: a local sensor/analysis script, and a cloud-hosted AI triage agent.

| Stage | Description |
|---|---|
| **1. Capture** | `soc_monitor.py` captures live traffic on a chosen interface for a fixed duration using tshark, saving a `.pcap` file. |
| **2. Parse** | The `.pcap` is converted to a CSV of packet records for analysis. |
| **3. Score** | Packets are grouped per source IP with `collections.Counter`; any source exceeding a configurable `THRESHOLD` is flagged as suspicious. |
| **4. Alert** | A structured JSON alert (source IP, packet count, destination host/IP, metadata) is written to `alert.json`. |
| **5. Send** | The script POSTs the alert JSON to the published Airia agent endpoint, authenticated with an `X-API-KEY` header. |
| **6. Triage** | The Airia agent (GPT-5 Nano, SOC-analyst system prompt) returns a structured triage report: classification, confidence, MITRE mapping, reasoning, and recommended actions. |

## 3. Building the AI Agent in Airia Studio

- **3.1 Create a Project** — a new Airia Studio project ("Junior Soc Analyst") was created to house the agent, model, and related configuration.
- **3.2 Add the Model** — GPT-5 Nano was selected as the underlying model, chosen for its low cost ($0.05/1M input tokens, $0.40/1M output tokens) and large 400K context window, well suited to a lightweight triage task.
- **3.3 Configure the Agent** — given a system prompt establishing it as an enterprise SOC Triage Analyst AI, tasked with analyzing structured JSON alert data and producing a professional triage report against a defined SOC playbook.
- **3.4 Publish the Agent** — published privately to the Airia Catalogue with an API Endpoint interface, making it callable from an external script rather than only from the Studio chat UI.
- **3.5 Retrieve API Credentials** — the API interface exposes a ready-made curl example for the Agent Execution endpoint; an Airia API key was generated to authenticate `soc_monitor.py`'s outbound requests.

## 4. The Monitoring Script — soc_monitor.py

A single Python script handles the full local half of the pipeline: capture, parse, score, alert, and send. Interface, capture duration, alert threshold, and Airia credentials are set as top-of-file configuration constants.

At runtime the script:

1. Captures live traffic for `CAPTURE_DURATION` seconds on `INTERFACE` using tshark, saving to `PCAP_FILE`
2. Converts the capture to `CSV_FILE`
3. Tallies packets per source IP with `collections.Counter`
4. Flags any source IP whose count exceeds `THRESHOLD`
5. Writes the resulting alert to `ALERT_FILE`
6. POSTs that alert to `AIRIA_API_URL` with the API key in the request header, printing the AI agent's JSON response to the terminal

## 5. Test Run & Results

**5.1 Generating test traffic:** 50 ICMP echo requests were sent to a target host on the lab network — comfortably above the 40-packet threshold configured in the script (`ping 192.168.1.46 -c 50`, round-trip times ~1 ms).

**5.2 Running the monitor:** `soc_monitor.py` was launched with root privileges (required for live packet capture) on interface `enp0s3`, capturing for 100 seconds while the ping traffic was generated in parallel.

**5.3 Detection & AI triage output:** the capture correctly tallied 50 packets from `192.168.1.46`, exceeded the 40-packet threshold, wrote the alert JSON, and forwarded it to the Airia agent, which responded with HTTP 200 and a structured triage report:

| Field | Value |
|---|---|
| **Source IP flagged** | 192.168.1.46 (50 packets vs. 40-packet threshold) |
| **Alert ID** | SOC-1F19D3B2 |
| **Threat classification** | Suspicious |
| **Confidence level** | Medium |
| **MITRE mapping** | Tactic: Uncertain (technique not determined from ICMP volume alone) |
| **Recommended actions** | Monitor → Enrich with threat intelligence → Review authentication/related logs |
| **HTTP response** | 200 OK from Airia agent endpoint |

## 6. Key Takeaways

- Chained a local, tool-based detection pipeline (tshark → CSV → threshold logic) directly into a cloud LLM agent via an authenticated REST call — a realistic pattern for AI-assisted SOC tooling.
- Configured and published a production-style Airia agent: project → model → agent → system prompt → publish → API key, mirroring how a real AI-augmented detection service would be stood up.
- Validated the agent's reasoning quality: it correctly distinguished "elevated but inconclusive" ICMP volume from a confirmed attack, and mapped its confidence and recommended actions accordingly rather than over-alerting.
- Threshold-based detection (Counter over per-source packet counts) is intentionally simple — a solid foundation to later extend with protocol-aware baselines, rate windows, or multiple detection rules feeding the same agent.

## 7. Next Steps

- Add additional detection rules (port-scan, SYN-flood, DNS tunneling heuristics) feeding the same Airia agent.
- Persist AI triage reports to a log/SIEM sink (e.g. forward into the existing Wazuh stack) for historical tracking.
- Expand the MITRE ATT&CK mapping logic with richer alert metadata so the agent can commit to a technique, not just a tactic.
- Add a feedback loop: allow an analyst to confirm/reject the AI classification to measure triage accuracy over time.
