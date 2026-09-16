UAE Heat-Stress Compliance Automation (n8n)

An n8n workflow that automates monitoring and audit-logging for UAE's mandatory mid-day work ban (MOHRE), plus live WBGT-based heat-stress alerting beyond the fixed window. Built as a portfolio project demonstrating n8n as an orchestration layer around live data, human-in-the-loop escalation, and a rules-based core with an AI layer added only for reporting (not decision-making).

Why this project

UAE law bans outdoor work 12:30 PM–3:00 PM daily from June 15–September 15, with fines up to AED 50,000 per violation. Newer guidance also expects ongoing WBGT (wet bulb globe temperature) monitoring and kept records, not just adherence to the fixed window. Most public automation examples for the UAE market are generic (CRM, lead gen, WhatsApp bots); this project targets a genuinely under-built compliance niche, using only data sources the operator is authorized to use (public weather APIs, owned site/HR data) — no scraping of government or third-party portals.

Architecture

Main workflow: Heat Stress Compliance

Schedule Trigger — fires every 15 minutes, all day.

IF — working hours — restricts processing to ~6am–6pm via $now.hour check.

Set — seasonal config — banStart / banEnd dates (e.g. 06-15 / 09-15), editable yearly without touching logic.

Code — seasonal check — compares today's date against the configured window; passes an inSeason boolean.

IF — in season? — false branch ends the run quietly for the day.

Google Sheets — site list — reads siteName, latitude, longitude, supervisorEmail per site.

Split In Batches — loops through sites one at a time.

HTTP Request — weather calls OpenWeatherMap (/data/2.5/weather) per site, using decimal-degree coordinates and a Query Auth credential for the API key.

Code — WBGT calculation derives WBGT from temperature + humidity (simplified outdoor approximation; no solar/wind load, since the free weather tier doesn't provide it). Re-attaches siteName/supervisorEmail from the Split In Batches node, since the HTTP node overwrites item data.

IF — violation check (OR) — true if current time is within the fixed 12:30–3:00 window, OR WBGT exceeds the configured threshold.

False → Set node ("no violation") → straight to logging.

True → alert path below.

Send Email (Gmail SMTP) — alert to the site supervisor, with an acknowledge link built from {{$execution.resumeUrl}}&ack=true.

Wait node — Resume: On Webhook Call, 10-minute timeout.

IF — acknowledged? — checks $json.query.ack === 'true'.

True → Set node ("acknowledged").

False (timeout) → escalation email to HR/safety → Set node ("escalated - no response").

Google Sheets — append — all three paths (no violation / acknowledged / escalated) converge here, each via its own Set node producing an identical row shape: siteName, wbgt, checkedAt, triggerReason, ackStatus.


Secondary workflow: Daily Summary

Schedule Trigger (once daily, after the ban window closes) → Google Sheets read (today's rows) → Code node aggregates counts/breakdown → OpenAI node generates a plain-language summary → Send Email to HR/safety management.

AI is used only for the human-facing summary — the compliance-critical alerting/escalation logic above is entirely rules-based, deliberately avoiding any hallucination risk in the part that actually triggers alerts or gets logged as an audit record.

Setup

Credentials needed

OpenWeatherMap — free tier (1,000 calls/day). Stored as an n8n Query Auth credential (appid), not hardcoded in the URL.
Gmail SMTP — App Password (requires 2-Step Verification on a personal Gmail account; Workspace accounts may block this — use OAuth2 Gmail node instead if so).
Google Sheets — standard OAuth2 credential.
OpenAI (for the summary workflow) — API key from platform.openai.com.

Config

Site list sheet: siteName, latitude, longitude (decimal degrees, not DMS), supervisorEmail.
Seasonal window: banStart / banEnd in MM-DD format, update annually per MOHRE's published dates.
WBGT threshold: configurable in the violation IF node (commonly 32–33°C is treated as high-risk).


Coordinates must be decimal degrees. OpenWeatherMap rejects DMS strings (24°54'47.2"N) with a 400 "Nothing to geocode" error.
$execution.resumeUrl already includes a query string (a signature param). Appending ?ack=true breaks the URL — use &ack=true instead.
Field name/type mismatches fail silently. A typo'd config field (e.g. bandEnd vs banEnd) or a string/number mismatch makes date comparisons quietly return false with no error.
Twilio trial accounts restrict recipients to Verified Caller IDs and reject some optional API parameters — relevant if using SMS instead of email.
Pinned/mock data on the Wait node can make downstream IF nodes appear to ignore real execution data during testing — unpin before trusting a test run.
Compliance and data-source notes

All data sources used are either public APIs (weather) or data the operator already legitimately holds (site list, HR records). No government or third-party portal is scraped or accessed without authorization — UAE's Cybercrime Law (Federal Decree-Law No. 34 of 2021) treats unauthorized system access seriously, so this project intentionally avoids that category of data source entirely.

Possible extensions
Full WBGT calculation with solar radiation/wind (would need a paid weather API tier).
Multi-language alert templates (Arabic/Urdu/Hindi) for supervisor accessibility.
Slack escalation as an alternative/addition to email.
Weekly/monthly rollup reports alongside the daily summary.
