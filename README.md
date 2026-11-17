# Enterprise Web & Security Operations Monitoring Dashboard (Splunk SIEM)

An end-to-end **Splunk Enterprise** log analysis and SIEM monitoring dashboard built to track multi-server web traffic health, e-commerce revenue telemetry, and SSH brute-force authentication threats.

---

## Dashboard Preview

### Operational Health & Revenue Analytics
![HTTP Status and Revenue Panels](Panel_1&2.png)

### Web Traffic Trends & SSH Threat Detection
![Web Traffic and Failed SSH Logins](Panel_3&4.png)

---

## Architecture & Data Sources
* **SIEM Platform:** Splunk Enterprise 10.4 (Classic Simple XML Dashboard)
* **Log Sources:** Multi-tier Apache web access logs (`access_combined_wcookie`), Linux authentication logs (`secure.log`), and vendor sales records spanning `www1`, `www2`, `www3`, and `mailsv` (Splunk Enterprise Tutorial Dataset).
* **Data Enrichment:** File-based CSV lookup table (`Prices_lookup` mapped to `prices.csv`) correlating `productId` fields with product names and unit pricing.
* **Interactive Controls:** Dynamic dropdown input token (`$Server_token$`) enabling real-time filtering across individual web servers (`www1`–`www3`) or aggregated infrastructure traffic.

---

## Dashboard Panels & SPL Queries

### 1. HTTP Status Code Distribution (Operational Health)
Tracks web server HTTP response codes across 39,500+ events to monitor service availability (`200 OK`: 34,282) and isolate client/server anomalies (`503`: 952, `408`: 756, `500`: 733, `404`: 690).

    index=buttercup_games sourcetype=access_combined_wcookie
    | stats count by status
    | sort - count

### 2. Top Selling Products & Revenue (CSV Lookup Enrichment)
Enriches successful purchase transactions (`action=purchase status=200`) with external CSV pricing metadata via the `lookup` command to calculate unit volume and formatted revenue.

    index=buttercup_games sourcetype=access_combined_wcookie action=purchase status=200
    | lookup Prices_lookup productId OUTPUT product_name price
    | stats count as units_sold, sum(price) as total_revenue by product_name
    | eval total_revenue="$" . tostring(round(total_revenue, 2), "commas")
    | sort - units_sold

### 3. Hourly Web Traffic by Server (Regex Extraction & Token Filtering)
Extracts server host identifiers (`www1`, `www2`, `www3`) from file paths using regular expressions (`rex`) and visualizes hourly request volume using an interactive token-driven area chart.

    index=buttercup_games sourcetype=access_combined_wcookie
    | rex field=source "(?<web_server>www\d+)"
    | search web_server="$Server_token$"
    | timechart span=1h count by web_server

### 4. Failed SSH Logins by Source IP (SOC Threat Detection)
Parses Linux `secure.log` authentication failures via regex extraction to identify top brute-force attacker source IPs, applying conditional red color-scale thresholds to prioritize high-severity threats.

    index=buttercup_games source="*secure.log" "Failed password"
    | rex "from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
    | stats count as failed_attempts by src_ip, host
    | sort - failed_attempts

---

## Key Security & Operational Findings
* **SSH Brute-Force Activity:** Detected concentrated failed SSH login attempts targeting the infrastructure, led by source IP `87.194.216.51` (**948 attempts**), `211.166.11.101` (**743 attempts**), and `128.241.220.82` (**622 attempts**).
* **Service Reliability:** Maintained an ~86.7% HTTP `200` success rate across web nodes while surfacing `503 Service Unavailable` and `500 Internal Server Error` spikes for operational triage.
* **Revenue Attribution:** Identified top revenue-generating products (*Manganiello Bros.* at **$8,357.91** and *Dream Crusher* at **$8,237.94**) through automated lookup correlation.

---

## Repository Contents
* `README.md` — Project architecture, SPL documentation, and key findings.
* `siem_operations_dashboard.xml` — Exported Splunk Classic Simple XML dashboard definition.
* `spl_queries.spl` — Standalone SPL queries used across all four dashboard panels.
* `prices.csv` — Product catalog lookup table used for transaction enrichment.
* `Panel_1&2.png` & `Panel_3&4.png` — Dashboard visualization screenshots.

---

## How to Reproduce in Splunk
1. Create a dedicated index named `buttercup_games` in **Settings > Indexes**.
2. Ingest the multi-server web and authentication log archive into `buttercup_games`.
3. Upload `prices.csv` under **Settings > Lookups > Lookup table files** and configure a file-based lookup definition named `Prices_lookup`.
4. Create a new Classic Simple XML dashboard, switch to **Source**, paste the contents of `siem_operations_dashboard.xml`, and click **Save**.
