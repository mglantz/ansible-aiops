
---
name: snow-aap-resolver
description: >
  Digital worker that automatically resolves high-priority ServiceNow incidents
  using Ansible Automation Platform (AAP). Use this skill whenever the user asks
  to process, triage, or resolve ServiceNow incidents using Ansible automation,
  or when building an agentic loop that bridges ITSM and automation execution.
  Triggers on: "resolve incidents", "process ServiceNow tickets", "automate
  incident remediation", "AAP incident automation", "run Ansible for incidents",
  or any workflow that combines ServiceNow and Ansible Automation Platform.
compatibility: "Requires ServiceNow MCP server and Ansible Automation Platform MCP server"
---
 
# ServiceNow → AAP Incident Resolver
 
A digital worker that fetches high-priority NEW incidents from ServiceNow,
matches them to appropriate Ansible Automation Platform job templates, and
executes automated remediation — then updates the incident with the outcome.
 
---
 
## Incident Filter Criteria
 
Only work incidents that meet **all three** conditions:
 
| Field   | Required value |
|---------|---------------|
| State   | `New` (1)     |
| Impact  | `1` (High)    |
| Urgency | `1` (High)    |
 
Do **not** process incidents with any other combination of state/impact/urgency.
 
---
 
## High-Level Workflow
 
```
1. FETCH   → Get qualifying NEW/P1 incidents from ServiceNow
2. TRIAGE  → For each incident, pick the best AAP job template
3. EXECUTE → Launch the job template with target_host set
4. UPDATE  → Write outcome back to the ServiceNow incident
5. REPEAT  → Continue until the queue is empty
```
 
Work incidents **one at a time** in order of `sys_created_on` (oldest first).
 
---
 
## Step-by-Step Instructions
 
### Step 1 — Fetch Qualifying Incidents
 
Use the ServiceNow MCP to query for incidents:
 
```
state = 1       (New)
impact = 1      (High)
urgency = 1     (High)
```
 
Retrieve these fields for each incident:
- `sys_id` — unique identifier (needed for updates)
- `number` — human-readable ID (e.g. INC0012345)
- `short_description`
- `description`
- `cmdb_ci` — Configuration Item (maps to the affected host)
- `caller_id.name` — for context
 
If no incidents are found, report that the queue is clear and stop.
 
---
 
### Step 2 — Determine the Target Host
 
For each incident, extract the **target host** from:
 
1. `cmdb_ci` field (preferred — the CI name/FQDN in the CMDB)
2. Failing that, parse the `description` for a hostname or IP address
3. If neither yields a host, **skip the incident** and add a work note:
   > "Automated remediation skipped — no target host could be determined. Manual review required."
 
---
 
### Step 3 — Select an AAP Job Template
 
List all available job templates from the AAP MCP. For each template:
 
- **Read the template description carefully.** The description explains what the
  template does, what problem it solves, and any prerequisites.
- Score the template against the incident `short_description` and `description`.
- Choose the template whose description best matches the incident's problem.
 
**Selection rules:**
- Prefer specificity: a template named "Restart Apache Service" beats a generic
  "Run Health Check" for an Apache incident.
- If multiple templates are plausible, pick the most targeted one.
- If **no template** is a reasonable match, skip automation for this incident
  and add a work note explaining why.
 
Record the chosen `template_id` and `template_name`.
 
---
 
### Step 4 — Launch the Job Template
 
Call the MCP service AAP-job_management to launch the selected job template with extra_vars defining the variable target_host as the sys
tem we are fixing.
 
The `target_host` variable is **always required** � every template supports it
to scope execution to the affected system.
 
Wait for job completion (poll if the MCP provides a job status endpoint).
Capture:
- `job_id`
- `status` (`successful` | `failed` | `error`)
- `stdout` or summary output (truncate to first 2000 chars if very long)
 
---
 
### Step 5 — Update the ServiceNow Incident
 
Use the ServiceNow MCP to update the incident (`sys_id`):
 
#### On job SUCCESS:
- Set **State** → `Resolved` (6)
- Set **Resolution code** → `Solved (Permanently)` (if the field exists)
- Set **Resolution notes** / **Close notes**:
  ```
  Automated remediation completed successfully.
  AAP Job Template: <template_name> (ID: <template_id>)
  AAP Job ID: <job_id>
  Target host: <target_host>
  
  Job output summary:
  <first 1500 chars of stdout>
  ```
- Add a **Work note** summarising what was done.
 
#### On job FAILURE or ERROR:
- Do **not** resolve the incident.
- Add a **Work note**:
  ```
  Automated remediation attempted but failed.
  AAP Job Template: <template_name> (ID: <template_id>)
  AAP Job ID: <job_id>
  Target host: <target_host>
  Status: <status>
  
  Job output summary:
  <first 1500 chars of stdout>
  
  Manual intervention required.
  ```
- Leave State as `New` (do not change it).
 
---
 
### Step 6 — Report and Continue
 
After each incident, output a one-line summary to the user:
 
```
✅ INC0012345 — Resolved via "Restart Apache Service" on web01.example.com
❌ INC0012346 — Job failed; work note added, manual review needed
⏭️  INC0012347 — Skipped; no target host found
```
 
Then move on to the next qualifying incident until the queue is empty.
 
---
 
## Error Handling
 
| Situation | Action |
|-----------|--------|
| ServiceNow MCP unavailable | Abort, report error to user |
| AAP MCP unavailable | Abort, report error to user |
| No matching job template | Skip incident, add work note, continue |
| No target host resolvable | Skip incident, add work note, continue |
| Job launch fails (API error) | Add work note with error, do not resolve, continue |
| Job times out | Treat as failure; add work note with timeout detail |
| Incident already updated mid-run | Check state before updating; skip if no longer New |
 
---
 
## Safety Rules
 
- **Never** set an incident to Resolved unless the AAP job returned `successful`.
- **Never** mutate incident fields other than State, Resolution notes,
  Close notes, and Work notes.
- **Never** guess a target host — only use values from `cmdb_ci` or explicit
  text in the incident description.
- **Always** record the AAP job ID in ServiceNow for auditability.
- Process at most **20 incidents per run** to avoid runaway execution.
  If more exist, report the count and ask the user to re-trigger.
 
---
 
## Example Summary Output
 
```
ServiceNow → AAP Incident Resolver — Run complete
═══════════════════════════════════════════════════
Incidents processed : 4
  ✅ Resolved        : 2
  ❌ Job failed      : 1
  ⏭️  Skipped         : 1
 
Details:
  ✅ INC0001234 — "Disk usage critical on db02" → "Disk Cleanup Automation" → RESOLVED
  ✅ INC0001235 — "Apache not responding on web03" → "Restart Apache Service" → RESOLVED
  ❌ INC0001236 — "DB replication lag on db04" → "Fix DB Replication" → JOB FAILED
  ⏭️  INC0001237 — "Network connectivity issue" → No target host found → SKIPPED
```
