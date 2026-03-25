# VICIdial New Features: Everything You Need to Know

VICIdial has gone through a significant development cycle over the past eighteen months. Between SVN revisions 3848 and 3939, the codebase picked up ConfBridge support as a production-ready conferencing engine, a full PHP 8 compatibility overhaul, critical security patches, new API endpoints, ViciPhone v3.0 with modern WebRTC capabilities, two-factor authentication, and a MariaDB TIMESTAMP schema fix that touches 40 database tables.

If you are running a VICIdial deployment in production today, or planning an upgrade from an older revision, this article walks through every meaningful change: what actually improved, what broke, what you need to reconfigure, and how to upgrade without taking your dialer offline.

## What Changed in VICIdial: The Quick Summary

The changes across recent SVN revisions fall into seven categories. Here is the condensed version before we dig into each one.

| Category | What Changed | Key Revisions |
|---|---|---|
| **Security** | Two CVEs patched (KL-001-2024-011, KL-001-2024-012). Input variable filtering added across admin and agent web interfaces. | SVN 3848+ (July 2024) |
| **PHP 8 Compatibility** | Hundreds of files updated to eliminate deprecated function calls, implicit type conversions, and nullable parameter warnings. | SVN 3863+ (August 2024) |
| **ConfBridge** | Production-ready MeetMe replacement integrated into the codebase. New `vicidial_confbridges` table, `conf_engine` and `conf_update_interval` server fields, and a replacement screen session script. | SVN 3894 (January 2025) |
| **MariaDB TIMESTAMP Fix** | Explicit `DEFAULT` and `ON UPDATE` clauses added to TIMESTAMP fields across 40 database tables to comply with MariaDB's deprecated implicit behavior. | SVN 3898 |
| **API Expansion** | Seven new Non-Agent API functions and the `send_notification` Agent API function with confetti support. | Various, through SVN 3939 |
| **ViciPhone v3.0** | Complete rewrite on SIP.js v0.20.1 with browser audio processing controls, regional progress tones, and settings container integration. | Concurrent with SVN 3900+ |
| **Two-Factor Authentication** | Email, phone call, and SMS-based 2FA for both admin and agent login screens. | SVN 3800+ |

Beyond these headline items, the development team also shipped Call Quota Lead Ranking recycling improvements, cross-cluster communication enhancements, a database query fix in `vdc_db_query.php` (SVN 3900), and the groundwork for ViciBox 13.0 running on OpenSuSE 16.0.

If your system is below SVN revision 3848, stop reading and upgrade immediately. The security vulnerabilities patched in that revision allowed authenticated users to achieve full database enumeration, remote code execution, and administrative takeover. Every revision you remain behind that threshold is an active risk to your operation.

> **Running a VICIdial system you haven't touched in a while?** [Request a free audit](/free-audit/) and we will tell you exactly which revisions and configuration changes your deployment needs.

## AMD Improvements: What's Actually Different

Answering machine detection in VICIdial has historically been a source of frustration. The built-in Asterisk AMD module relies on energy thresholds and silence timing to distinguish human pickups from voicemail greetings, and the accuracy varies wildly depending on carrier audio quality, greeting length, and regional telecom behavior. Recent changes to VICIdial give operators more control over how AMD results are routed, and a growing ecosystem of third-party AI-based AMD solutions has emerged that integrates cleanly with the platform.

### AMD Agent Route Options

The most operationally significant AMD change in recent revisions is the maturation of the **AMD Agent Route Options** feature. When enabled in the Campaign Detail screen, this setting gives you granular control over which AMD detection results actually get routed to a live agent.

Here is how it works. In your campaign settings, you set the **Routing Extension** to `8369`, set the **AMD Type** to `AMD`, and enable **AMD Agent Route Options**. Then, in the associated **Settings Container** (type `AMD_AGENT_OPT_Campaign`), you define exactly which detection results should be sent to agents.

**Before AMD Agent Route Options:**

- AMD detected a human: call goes to agent.
- AMD detected a machine: call gets dispositioned as `AA` (Answering Machine) or plays the Answering Machine Message.
- AMD returned `NOTSURE` or `TOOLONG`: call still goes to agent, wasting agent time on ambiguous detections.

**After AMD Agent Route Options:**

- You specify exactly which results route to agents. Setting the container to `HUMAN,HUMAN` sends only confirmed human answers to agents.
- You can remove `NOTSURE` and `TOOLONG` from the routing list, keeping only definitive detections.
- You can include `MAXWORDS` in the agent route if you want long-greeting pickups to still reach agents.

This matters because the default behavior of routing `NOTSURE` calls to agents was one of the biggest complaints about VICIdial AMD. Agents would pick up and hear the tail end of a voicemail greeting, waste 10-15 seconds figuring out it was a machine, then disposition and move on. Multiply that across a 200-seat floor and you are hemorrhaging productive talk time.

For a detailed walkthrough of AMD configuration including `cpd.conf` tuning and WaitForSilence optimization, see our [VICIdial AMD guide](/blog/vicidial-amd-guide/).

### CPD (Sangoma Call Progress Analysis) Settings

VICIdial also supports the **CPD AMD Action** setting in the Campaign Detail screen, which provides an alternative detection path using Sangoma's Call Progress Analysis. When set to `DISPO`, detected machines are automatically dispositioned as `AA` (Answering Machine) or `AFAX` (Fax Machine) and hung up before reaching an agent. When set to `MESSAGE`, the call is routed to the defined Answering Machine Message instead.

The recommended production settings for ViciBox 11 and 12 deployments are:

- **WaitForSilence**: `2000,2,30`
- **AMD Type**: `AMD`
- **AMD Agent Route**: Enabled

### AI-Based AMD Integration

The third-party AMD landscape has improved significantly. Solutions like AMDY.ai and VoiceDetect AMD now offer drop-in VICIdial integration that replaces Asterisk's built-in AMD module with speech-recognition-based detection. These systems transcribe the first few seconds of audio and classify the text as human or machine, achieving accuracy rates above 98% in testing while maintaining latency low enough for live transfers (typically under 3 seconds).

The integration path uses WebSocket connections and JSON payloads, which means the AI AMD engine runs as a separate service that your Asterisk dialers communicate with during call setup. This is a cleaner architecture than patching Asterisk's AMD module directly, and it means you can upgrade your AMD detection independently of your VICIdial SVN revision.

For operations where AMD accuracy directly impacts revenue, which is most outbound campaigns, the move from rule-based energy detection to language-based classification represents the single largest quality improvement available right now. If you want to explore what AI-based AMD looks like on your specific traffic, check our [AMD optimization feature page](/features/amd-optimization/).

> **Not sure if your AMD settings are costing you connects?** [Get a free audit](/free-audit/) and we will benchmark your current detection rates against optimized configurations.

## WebRTC Updates: Better Browser Phone Performance

ViciPhone v3.0 represents a complete rewrite of VICIdial's WebRTC softphone, and the improvements are substantial enough that any deployment still running ViciPhone v2.x should prioritize the upgrade.

### SIP.js v0.20.1 and PJSIP

The core change is the migration from an older SIP.js library to **SIP.js v0.20.1**. This is not a minor version bump. The newer SIP.js library handles WebSocket reconnection more gracefully, improves SRTP key negotiation, and resolves several edge cases around oEarly media that caused one-way audio on specific browser versions.

ViciPhone v3.0 requires PJSIP on the Asterisk side, which is the correct direction. PJSIP handles multiple contacts per AOR (Address of Record) better than legacy chan_sip, which means agents can have multiple browser tabs or devices registered without the SIP stack getting confused about where to route audio.

**Critical configuration requirement:** Your PJSIP webphone template must include `rtcp_mux=yes`. Without this setting, ViciPhone v3.0 will fail to establish media. This was not required in v2.x and is the most common upgrade issue we see.

The full PJSIP template requirements for WebRTC are:

- `encryption=yes`
- `avpf=yes`
- `rtcp_mux=yes`
- Transport: WSS on port 8089
- DTLS for encryption key negotiation
- Media profile: SAVPF (Secure Audio-Video Profile with Feedback, the only profile Chrome and Firefox accept for WebRTC)

### Browser Audio Processing Controls

ViciPhone v3.0 adds the ability to enable or disable browser-level audio processing features through a Settings Container:

- **Echo Cancellation**: Toggle on/off per deployment. Useful for environments where agents use headsets (echo cancellation can actually degrade quality with good headsets) versus open speakers.
- **Automatic Gain Control (AGC)**: Toggle on/off. AGC normalization can cause problems in noisy call center environments where background noise gets amplified during agent silence.

These controls are configured through a ViciPhone Settings Container in VICIdial's admin panel, which means you can adjust them without touching ViciPhone code. The Settings Container approach also means future ViciPhone features can be added without requiring VICIdial code changes.

### Regional Progress Tones

A small but meaningful addition: ViciPhone v3.0 can play regional progress audio instead of generic ringing when placing outbound calls. The audio file is configurable through the Settings Container with presets for North America, Europe, and the UK. For operations with agents in multiple countries, this reduces the disorientation of hearing unfamiliar ring tones.

### Auto-Conference on Registration

ViciPhone v3.0 automatically joins the agent conference upon successful SIP registration, eliminating the manual step that caused agents to occasionally forget to join their conference channel. This resolves a common support ticket where agents appear logged in but cannot receive calls.

### Common WebRTC Issues and NAT Traversal

The most persistent WebRTC issue across all VICIdial deployments remains NAT traversal. SIP signaling succeeds over WSS on port 8089, but RTP audio packets get trapped behind symmetric NATs. If you are experiencing one-way audio after upgrading to ViciPhone v3.0, the problem is almost certainly firewall or NAT configuration rather than a ViciPhone bug. Ensure your STUN/TURN server configuration is correct and that your firewall allows the RTP port range bidirectionally.

For a comprehensive setup walkthrough including cluster-specific WebRTC considerations, see our [VICIdial cluster guide](/blog/vicidial-cluster-guide/).

## API Changes: What Breaks and What to Update

The VICIdial API received meaningful additions across both the Agent API and Non-Agent API. If you have external integrations, CRM connectors, or automation scripts that talk to VICIdial's API layer, review this section carefully.

### New Non-Agent API Functions

Seven new functions were added to the Non-Agent API (`/vicidial/non_agent_api.php`) between 2023 and 2025:

| Function | Added | Purpose | Required User Level |
|---|---|---|---|
| `container_list` | June 2023 | Lists Settings Container information in the system | 7+ |
| `lead_dearchive` | November 2023 | Moves leads from the archive table back to the active `vicidial_list` table | 8+, modify_leads |
| `delete_fpg_phone` | January 2024 | Removes a phone number from a Filter Phone Group | 8+, modify_lists |
| `delete_dnc_phone` | March 2024 | Removes a phone number from a DNC list | 8+, modify_lists + Delete DNC |
| `update_remote_agent` | July 2024 | Updates remote agent settings (status, campaign_id, lines) | 8+, modify_remote_agents |
| `user_details` | August 2024 | Returns detailed information about a single user | 7+, view_reports |
| `hopper_bulk_insert` | July 2025 | Adds a group of lead_ids directly to the `vicidial_hopper` table | 8+, modify_campaigns |

The `hopper_bulk_insert` function is particularly notable. Previously, the only way to prioritize specific leads was to manipulate `rank` values or use the hopper's natural ordering. Now you can inject a batch of `lead_ids` directly into the hopper via API, which opens up use cases like real-time lead scoring integration where an external system identifies high-priority leads and pushes them to the front of the dial queue.

The `lead_dearchive` function addresses a long-standing operational pain point. VICIdial archives old leads to keep the `vicidial_list` table performant, but recovering archived leads previously required direct database access. Now it is an API call.

### New Agent API Function: send_notification

The Agent API gained the `send_notification` function, which sends customizable messages to agents by User, User Group, or Campaign. What makes this function unusual is the **confetti animation** support, with configurable parameters:

- `show_confetti`: Y/N to enable particle animation
- `duration`: Length of animation in seconds (default 2, max 10)
- `maxParticleCount`: Number of particles (default 2350, max 9999)
- `particleSpeed`: Animation speed (default 60, max 100)
- Text styling: configurable size, font, weight, and color

A corresponding campaign-level **Show Confetti** setting allows confetti to trigger automatically when agents disposition a call as `SALE` or any callback-flagged status. This is a gamification feature aimed at agent morale and engagement.

Beyond confetti, the `send_notification` function provides a clean way for external systems to push alerts to agents without going through the agent screen's built-in alert mechanisms. If you run workforce management or quality assurance tools alongside VICIdial, this function lets those tools communicate directly with agents.

### Agent API Additions for Cluster Operations

The `force_fronter_leave_3way` and `force_fronter_audio_stop` functions now support cluster-aware routing with three modes:

- `LOCAL_ONLY`: Action applies only to the local server.
- `LOCAL_AND_CCC`: Action applies locally and propagates via Cross-Cluster Communication.
- `CCC_REMOTE`: Action applies only to remote CCC-connected clusters.

This is significant for multi-cluster deployments where fronter and closer agents may be on different VICIdial installations. Previously, forcing a fronter out of a 3-way call only worked reliably when both agents were on the same cluster.

### What Might Break

If your integrations use hardcoded API response parsing, be aware that some existing functions received additional output fields. The `update_lead` and `add_lead` functions now include expanded webform variable support (`webform_one` through `webform_three`), which may change response payload sizes.

Additionally, the PHP 8 compatibility changes across the web interface mean that any custom PHP scripts you have deployed alongside VICIdial need to be tested against PHP 8.2, which is the version shipped with ViciBox 12. Common breakage points include implicit nullable parameters, deprecated `utf8_encode()`/`utf8_decode()` calls, and changes to string-to-number comparison behavior.

> **Have custom API integrations with VICIdial?** [Request a free audit](/free-audit/) to make sure your integrations will survive the upgrade without breaking your workflows.

## Database Schema Changes: Migration Considerations

The database changes in recent SVN revisions are more extensive than a typical update cycle. There are three distinct categories of schema modifications you need to handle.

### MariaDB TIMESTAMP Field Overhaul (SVN 3898)

This is the most impactful schema change. MariaDB deprecated and then disabled the implicit `ON UPDATE CURRENT_TIMESTAMP` behavior for TIMESTAMP fields. VICIdial relied on this implicit behavior extensively, meaning TIMESTAMP fields that previously auto-updated when a row changed would silently stop updating after a MariaDB upgrade.

The fix, landed in SVN revision 3898, adds explicit `DEFAULT` and `ON UPDATE` clauses to TIMESTAMP fields across **40 database tables**. Both `MySQL_AST_CREATE_tables.sql` and `upgrade_2.14.sql` were updated.

**What this means in practice:**

- If you are creating a **new VICIdial database** on MariaDB 10.11+ (as shipped with ViciBox 12), you must use the updated `MySQL_AST_CREATE_tables.sql` or add `explicit_defaults_for_timestamp = Off` to your MariaDB configuration.
- If you are **upgrading an existing database**, run `upgrade_2.14.sql` to apply the explicit clauses. Existing databases imported from older servers generally work fine because MariaDB preserves the original column definitions during import.
- If you encounter issues, the workaround is to add `explicit_defaults_for_timestamp = Off` to `/etc/my.cnf.d/general.cnf` and restart MariaDB. This re-enables the legacy behavior globally.

The `server_updater` table's `db_time` field is particularly sensitive to this change. If it stops auto-updating, VICIdial's keepalive processes may incorrectly mark servers as offline.

### ConfBridge Tables and Server Fields

The ConfBridge integration adds:

- **New table**: `vicidial_confbridges` -- mirrors the structure of `vicidial_conferences` but tracks ConfBridge-specific conference state including extension, server IP, and 3-way call state. You must insert 300 conference entries (9600000 through 9600299) for each dialer server.
- **New server fields**: `conf_engine` (ENUM: MEETME or CONFBRIDGE) and `conf_update_interval` (default 60 seconds) added to the `servers` table.

The `vicidial_confbridges` table entries are not created automatically by the SQL upgrade scripts in all cases. You may need to manually insert them using the SQL provided in the ConfBridge documentation.

### DB Schema Version

After running `upgrade_2.14.sql`, your DB Schema Version should read **1729** or higher (visible in System Settings). If the admin interface shows a "WARNING: Code expects different schema" message after an SVN update, run the upgrade SQL again:

```
mysql -p -f --database=asterisk < /usr/src/astguiclient/trunk/extras/upgrade_2.14.sql
```

The `-f` flag forces the script to continue past errors from already-applied statements, which is safe for re-runs.

### Additional Column Changes

Beyond the headline items, recent revisions added columns for:

- Two-factor authentication state tracking on user records
- AMD Agent Route Options settings on campaign records
- Cross-Cluster Communication routing parameters
- Call Quota Lead Ranking recycling fields
- Agent DID reporting columns
- GDPR compliance fields for lead management

Review the full `upgrade_2.14.sql` file for the complete list. If you are jumping multiple schema versions, run the upgrade SQL sequentially and check the `system_settings` table's `db_schema_version` field after each run to confirm it applied correctly.

## Performance Improvements: Benchmarks Before and After

Performance changes in the recent VICIdial revisions come from three sources: the ConfBridge migration, PJSIP adoption, and the MariaDB TIMESTAMP fix. The picture is nuanced -- some things got better, some require careful tuning, and one area demands attention.

### ConfBridge vs. MeetMe: Audio Quality and Resource Usage

ConfBridge performs audio mixing in **user space** rather than kernel space (which is how MeetMe works through the DAHDI `dahdi_transcode` module). This architectural difference has performance implications in both directions.

**Improvements:**

- ConfBridge eliminates the "bloop" sound that customers hear when joining conference calls under MeetMe. This was a longstanding complaint from operations running warm transfer workflows where the customer could hear the conference join tone.
- ConfBridge supports `dsp_drop_silence` across user profiles, which reduces CPU usage during quiet portions of calls by not mixing silent audio frames.
- ConfBridge does not require DAHDI hardware for basic operation, simplifying virtualized deployments.

**Trade-offs:**

- Because ConfBridge mixes audio in user space, it is more sensitive to CPU contention and virtualization overhead than MeetMe. Physical servers or VMs with dedicated CPU cores are recommended for high-density dialer servers.
- The `mixing_interval` setting (default 40ms in some configurations, recommended 20ms) directly affects audio quality. Lower values (10-20ms) prevent RTP packet ordering issues with SIP endpoints but increase CPU usage.

**Timing configuration is critical.** By default, ConfBridge uses `res_timing_timerfd.so`, but Asterisk's AGI script execution (which VICIdial uses heavily) causes timing issues with this module. The recommended configuration disables the default timing modules in `/etc/asterisk/modules.conf`:

```
noload => res_timing_timerfd.so
noload => res_timing_kqueue.so
noload => res_timing_pthread.so
```

This forces Asterisk to use DAHDI for timing, which provides kernel-space timing accuracy even though the audio mixing itself happens in user space.

### PJSIP vs. chan_sip: CPU Overhead

Forum reports from ViciBox 12 deployments show that 100 extensions with 150 simultaneous auto-dial calls consumed over 96 CPU units on Asterisk 18 with PJSIP. However, experienced operators report that the performance difference between PJSIP and legacy chan_sip is negligible in practice once the system is properly tuned.

The key PJSIP performance considerations:

- PJSIP creates more threads per endpoint than chan_sip. Set `system/threadpool_initial_size` and `system/threadpool_auto_increment` in `pjsip.conf` to match your expected agent count.
- PJSIP's transport layer handles WebSocket connections natively, which means `res_pjsip_transport_websocket.so` must be loaded for ViciPhone but adds no meaningful overhead beyond the connection itself.

### MariaDB Query Performance

The `vdc_db_query.php` fix in SVN 3900 resolved an issue where non-numeric `start_epoch` values caused HTTP 500 errors during call completion logging. While not a performance improvement per se, this fix eliminated a failure mode that could cascade into dropped call records and agent screen freezes during high-volume periods.

For general MySQL/MariaDB tuning, the standard VICIdial recommendations remain:

- Set `Auto Trim Hopper` to **N** in System Settings.
- Keep the hopper size only as large as needed for a few minutes of dialing per campaign.
- Monitor the `vicidial_log` table size and archive aggressively.

For detailed dialer tuning approaches that account for these performance changes, see our [dialer tuning feature page](/features/dialer-tuning/).

> **Want to know if your servers can handle the upgrade without a performance hit?** [Request a free audit](/free-audit/) and we will profile your current resource usage and identify bottlenecks before you touch SVN.

## New Reporting Features and the Data They Surface

VICIdial's reporting engine did not receive a ground-up redesign in recent revisions, but several additions expand what you can track and how you surface it.

### Agent DID Reporting

New columns in the call logging tables allow reports to include the specific DID that an inbound call arrived on, associated with the agent who handled it. This is particularly useful for operations that assign dedicated DIDs to individual agents or small teams and need to track per-DID conversion rates.

Previously, DID reporting was available at the campaign and in-group level but not cleanly associated with the handling agent in historical reports. The new fields close that gap.

### Call Quota Lead Ranking Reporting

The Call Quota Lead Ranking feature set now includes logging and reporting for its recycling operations. You can track how leads progress through their quota cycles, which time periods they were dialed in, and whether they met the configured contact thresholds.

Important operational detail: Call Quota logs are **archived after 7 days**. All recycling operations for a given lead must complete within 7 days, or the system loses the historical data needed to make ranking decisions. If your recycle cycles are longer than 7 days, this feature will not work correctly.

The reporting for this feature is available through the existing VICIdial admin reports interface, and the underlying data lives in dedicated log tables that can be queried directly for custom reporting.

### Real-Time Report Enhancements

The real-time campaign display screens continue to provide mobile-accessible summary views across all campaigns' live stats. The AJAX functions that power these displays can be treated as lightweight APIs for building custom dashboards, since they accept the same authentication parameters and return structured data.

A community patch addresses the long-standing issue where agents marked as `LAGGED` while on an active call appear as `DISPO` in the real-time report, giving administrators false readings about agent availability. If you run a large floor and rely on the real-time report for staffing decisions, check whether your SVN revision includes this fix or apply the patch manually.

### Cross-Cluster Communication Reporting

For multi-cluster deployments using VICIdial's built-in Cross-Cluster Communication features, the system now returns richer status data from remote clusters including:

- Number of agents waiting, in-call, paused, and total
- Number of calls waiting in queue
- Calls today and drops today
- Drop percentage

This data enables smarter routing decisions when distributing calls across clusters based on real-time capacity rather than static configuration.

### CDR and Export Improvements

The `AST_inbound_export.pl` script (added as part of the 2.14 feature set) enables automated export of in-group call data. This is useful for operations that feed VICIdial call records into external business intelligence tools, data warehouses, or compliance archival systems.

## Configuration Settings Added or Modified

This section catalogs the specific settings that were added or changed. If you are upgrading, walk through this list and verify each setting in your admin panel after applying the update.

### System Settings

| Setting | Type | Purpose |
|---|---|---|
| Two-Factor Admin Auth Hours | Numeric | Hours before admin 2FA re-authentication required (0 = disabled) |
| Two-Factor Agent Auth Hours | Numeric | Hours before agent 2FA re-authentication required (0 = disabled) |
| Two-Factor Auth Config Container | Text | Name of the 2FA_SETTINGS type Settings Container |

### Server Settings

| Setting | Type | Purpose |
|---|---|---|
| Conferencing Engine | ENUM (MEETME/CONFBRIDGE) | Selects the conferencing application for this server |
| Conf Update Interval | Numeric (default 60) | Seconds between conference state updates |

### Campaign Settings

| Setting | Type | Purpose |
|---|---|---|
| AMD Agent Route Options | Enable/Disable | Controls granular AMD result routing to agents |
| AMD Type | ENUM | Selects AMD detection method (AMD, CPD, etc.) |
| Show Confetti | Campaign option | Triggers confetti animation on SALE/CALLBACK dispositions |
| Call Quota Lead Ranking | Multiple fields | Controls lead recycling based on contact quotas per time period |

### Settings Containers (New Container Types)

| Container Type | Purpose |
|---|---|
| `AMD_AGENT_OPT_Campaign` | Defines which AMD detection results route to agents (e.g., `HUMAN,HUMAN` or `HUMAN,NOTSURE,MAXWORDS`) |
| `2FA_SETTINGS` | Configures two-factor authentication parameters including code length, expiration, delivery method (email/phone/SMS), and lockout thresholds |
| ViciPhone Settings Container | Controls browser audio processing (echo cancellation, AGC), regional progress tones, auto-conference behavior, and translation settings |
| `DNCDOTCOM` | DNC.com API integration settings |

### astguiclient.conf Changes

The `VARactive_keepalives` variable in `/etc/astguiclient.conf` now accepts a `C` flag to enable the ConfBridge screen session. If you switch to ConfBridge, you must:

1. Add `C` to the `VARactive_keepalives` line.
2. Comment out the old conference validator crontab entry.
3. The new `AST_conf_update_screen.pl` script replaces both `AST_conf_update.pl` and `AST_conf_update_3way.pl`.

### User Settings

| Setting | Type | Purpose |
|---|---|---|
| Two Factor Auth Override | ENUM (NOT ACTIVE/Disabled) | Per-user override to disable 2FA when system-wide 2FA is enabled |

### Asterisk Configuration Files

For ConfBridge deployments, you need these Asterisk configuration files:

- **confbridge-vicidial.conf**: Defines one bridge profile and eight user profiles (customer, agent, admin, monitoring, recording, barge, DTMF, audio playback) with specific `hear_own_join_sound` settings per profile.
- **extensions.conf**: Eight new extension patterns using the 9600XXX range prefix scheme.
- **manager.conf**: A new `[confcron]` user with read/write command and reporting permissions, including event filters for both MeetMe and ConfBridge events.
- **modules.conf**: Disable `res_timing_timerfd.so`, `res_timing_kqueue.so`, and `res_timing_pthread.so` to force DAHDI timing.

For the full setup walkthrough covering all of these settings in context, see our [VICIdial setup guide](/blog/vicidial-setup-guide/).

## Known Issues and Workarounds

Every upgrade cycle surfaces issues. Here are the ones that matter, with solutions.

### FastAGI Binding Failure on Fresh Installations

**Issue:** On fresh ViciBox 12 installations, the FastAGI service fails to bind to IPv6 port 4577, preventing AGI scripts from executing.

**Workaround:** Force FastAGI to bind to IPv4 only by modifying the FastAGI startup configuration. In most cases, adding `-host 0.0.0.0` to the FastAGI startup arguments resolves the issue. If your server has IPv6 disabled at the kernel level, ensure the AGI configuration explicitly references the IPv4 loopback address.

### ViciBox 12 Firewall Configuration

**Issue:** The initial ViciBox 12.0.0 release did not enable the firewall by default, creating a security gap. Subsequent releases (12.0.1, 12.0.2) corrected this but introduced YaST/firewalld interaction issues where the `external.xml` zone file contained incorrect settings.

**Workaround:** Verify your firewall is active with `firewall-cmd --state`. If YaST firewall configuration throws errors, directly edit the firewalld zone files in `/etc/firewalld/zones/` rather than using the YaST interface.

### ConfBridge Monitoring (Listen/Barge) Broken Out of the Box

**Issue:** The real-time report's listen and barge functionality does not work with ConfBridge because `non_agent_api.php` line 3295 references the `vicidial_conferences` table instead of `vicidial_confbridges` when the server is configured for ConfBridge.

**Workaround:** Apply the patch documented in the CyburDial/Dialer.one ConfBridge guide, or manually update the table reference in `non_agent_api.php` to check the `conf_engine` setting and query the appropriate table. This may already be fixed in your SVN revision -- check the file directly.

### ConfBridge Conferences Not Auto-Created

**Issue:** The `vicidial_confbridges` table entries (300 conferences per server, numbered 9600000-9600299) are not always created by the upgrade SQL scripts. If they are missing, agents cannot be placed into conferences and calls will fail silently.

**Workaround:** Manually insert the conference entries using the SQL provided in the ConfBridge documentation. Run this for each dialer server IP in your cluster.

### MariaDB TIMESTAMP Regression on New Databases

**Issue:** Newly created databases on MariaDB 10.11+ will have TIMESTAMP fields that do not auto-update, even after running `upgrade_2.14.sql`, because MariaDB's default behavior changed.

**Workaround:** Add `explicit_defaults_for_timestamp = Off` to `/etc/my.cnf.d/general.cnf` and restart MariaDB. Alternatively, dump the database, drop it, recreate it with proper character set and collation, and reimport:

```bash
mysqldump -B asterisk | gzip > asterisk-backup.sql.gz
mysql -u root -p -e "DROP DATABASE asterisk;"
mysql -u root -p -e "CREATE DATABASE asterisk DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;"
pv asterisk-backup.sql.gz | gunzip | mysql -u root -p asterisk
```

### PHP 8 Deprecation Warnings in Custom Code

**Issue:** VICIdial's own PHP codebase has been updated for PHP 8 compatibility (SVN 3863+), but any custom scripts, reports, or integrations you have deployed alongside VICIdial may throw deprecation warnings or fatal errors under PHP 8.2.

**Workaround:** Test all custom PHP code against PHP 8.2 before upgrading your web server. Common issues include: `utf8_encode()`/`utf8_decode()` (replaced with `mb_convert_encoding()`), implicit nullable parameters, changed string-to-number comparison behavior, and removed `create_function()` calls.

### vdc_db_query.php HTTP 500 Errors (Fixed in SVN 3900)

**Issue:** The `vdc_db_query.php` script could throw HTTP 500 errors during call completion if the `start_epoch` field contained non-numeric characters.

**Workaround:** Upgrade to SVN 3900+. The fix overwrites `start_epoch` only if it has not been populated or contains non-digits.

### zypper Update Issues on ViciBox

**Issue:** Users running ViciBox on OpenSuSE 15.6 have reported recurring `zypper up` failures.

**Workaround:** Apply the specific patch: `zypper in -t patch openSUSE-SLE-15.6-2024-4321=1 SUSE-2024-4321=1`

> **Encountering issues during or after your upgrade?** [Request a free audit](/free-audit/) and our team will diagnose and resolve configuration problems across your entire cluster.

## Upgrade Path: How to Upgrade Safely

Upgrading VICIdial is not a one-button operation. The process involves SVN source code updates, database schema migrations, Perl script deployment, and Asterisk configuration changes. Here is the step-by-step path that minimizes risk.

### Pre-Upgrade Checklist

Before you touch anything:

1. **Check your current SVN revision and DB schema version.** In the admin panel, go to System Settings and note both values. You can also run `svn info /usr/src/astguiclient/trunk` on the command line.
2. **Read the UPGRADE file.** It lives at `/usr/src/astguiclient/trunk/UPGRADE` and is updated with every release. The instructions can change between revisions, so always read the version you are upgrading to, not the one you have.
3. **Verify your Asterisk version.** If you are migrating to ConfBridge, you need Asterisk 16 or higher. ViciBox 12 ships Asterisk 18. VICIdial does not support Asterisk "short-term" releases (19, 21, 23) because they only receive one year of Sangoma support. Asterisk 20 is the recommended LTS target for new deployments.
4. **Check PHP version compatibility.** ViciBox 12 ships PHP 8.2. If you have custom PHP code, test it first.
5. **Confirm MariaDB version.** ViciBox 12 ships MariaDB 10.11.9. Note whether you need the TIMESTAMP fix.

### Step 1: Full Backup

Run the backup script on every server that hosts the database:

```bash
perl /usr/share/astguiclient/ADMIN_backup.pl --debugX
```

For multi-server deployments, run the backup **without** database and web components on dialer-only servers (they do not host the database).

Back up your Asterisk configuration files separately:

```bash
cp -r /etc/asterisk /etc/asterisk.backup.$(date +%Y%m%d)
```

### Step 2: Download the Latest SVN

```bash
cd /usr/src/astguiclient
svn checkout svn://svn.eflo.net:3690/agc_2-X/trunk
```

Or if you already have a checkout:

```bash
cd /usr/src/astguiclient/trunk
svn update
```

Note the revision number reported by SVN. Verify it is 3848 or higher (for security patches) and ideally 3898+ (for the TIMESTAMP fix).

### Step 3: Upgrade the Database First

Always upgrade the database before deploying code. Run the appropriate upgrade SQL:

```bash
mysql -p -f --database=asterisk < /usr/src/astguiclient/trunk/extras/upgrade_2.14.sql
```

The `-f` flag lets the script continue past statements that have already been applied. Check the DB Schema Version in System Settings after running.

If you are jumping from an older version (e.g., 2.12 or 2.13), you need to run the intermediate upgrade SQL files in order. Check the `extras/` directory for `upgrade_2.12.sql`, `upgrade_2.13.sql`, etc.

### Step 4: Deploy Code to Each Server

On each server in your cluster:

```bash
cd /usr/src/astguiclient/trunk
perl install.pl
```

The installer backs up any customized `bin/` and `agi/` scripts before overwriting them. Review the backup directory afterward if you have custom modifications.

### Step 5: Rebuild Configuration Files

In the admin panel, go to **Admin > Servers**, click on each server, and set **Rebuild conf files = Y**. This regenerates the Asterisk dialplan and SIP/PJSIP configuration from the database.

### Step 6: Update Phone Codes (If Needed)

If your area code or country code data is outdated:

```bash
/usr/share/astguiclient/ADMIN_area_code_populate.pl --purge-table --debug
```

### Step 7: ConfBridge Migration (Optional)

If you are switching from MeetMe to ConfBridge:

1. Ensure Asterisk 16+ is installed on your dialers.
2. Deploy the `confbridge-vicidial.conf` file to `/etc/asterisk/`.
3. Update `extensions.conf` with the 9600XXX dial patterns.
4. Add the `[confcron]` user to `manager.conf`.
5. Disable timing modules in `modules.conf`.
6. Insert 300 conference entries into the `vicidial_confbridges` table per server.
7. Add `C` to `VARactive_keepalives` in `/etc/astguiclient.conf`.
8. Comment out the old conference validator crontab entry.
9. In the admin panel, go to **Admin > Servers**, change **Conferencing Engine** to `CONFBRIDGE`.
10. Restart Asterisk and verify conferences are functional.

### Step 8: Post-Upgrade Verification

After all servers are updated:

1. Check the admin panel for schema warnings.
2. Place test calls through each dialer to verify audio path.
3. Test agent login, call handling, and disposition.
4. Verify real-time reports are updating.
5. If using WebRTC, confirm ViciPhone connects and media flows bidirectionally.
6. Check the Asterisk CLI for errors: `asterisk -rx "core show channels"`.
7. Monitor `/var/log/astguiclient/` for errors in the keepalive and dial scripts.

### What Not to Do

- **Do not skip the database upgrade.** Running new code against an old schema causes unpredictable failures.
- **Do not upgrade your database server's MariaDB version and VICIdial code at the same time.** Do them separately so you can isolate issues.
- **Do not install stock Asterisk from source.** VICIdial requires custom Asterisk patches (for AMD stats, IAX/SIP peer status, and dial timeout resets). Use the patched Asterisk from the VICIdial download repository at `download.vicidial.com`.
- **Do not upgrade a production cluster during business hours.** Even with careful planning, unexpected issues arise. Schedule upgrades during a maintenance window.

For operations running multi-server clusters with complex routing, see our [VICIdial cluster guide](/blog/vicidial-cluster-guide/) for cluster-specific upgrade considerations.

> **Want a hands-off upgrade?** [Request a free audit](/free-audit/) and our team will plan, execute, and verify your upgrade with zero downtime.

## ViciStack Compatibility With VICIdial

ViciStack maintains full compatibility with the latest VICIdial SVN revisions. Here is what that means in practice.

### Current Compatibility Status

ViciStack deployments are tested against every SVN revision as it lands in trunk. Our managed systems are currently running SVN 3939 with DB Schema 1729 on ViciBox 12, and all ViciStack optimization layers, monitoring dashboards, and automated tuning features work without modification.

### ConfBridge Support

ViciStack fully supports ConfBridge deployments. Our [dialer tuning](/features/dialer-tuning/) automation accounts for the different resource profile of ConfBridge vs. MeetMe, automatically adjusting `mixing_interval` and monitoring user-space CPU utilization patterns that differ from kernel-space MeetMe mixing.

### AMD Optimization Compatibility

ViciStack's [AMD optimization](/features/amd-optimization/) works with both the built-in AMD Agent Route Options and third-party AI AMD solutions. Our optimization layer tunes the campaign-level AMD settings, WaitForSilence thresholds, and routing extension configuration to maximize connect rates on your specific traffic patterns.

### WebRTC and ViciPhone v3.0

ViciStack deployments using WebRTC agents are fully compatible with ViciPhone v3.0. Our monitoring includes WebSocket connection health, SRTP negotiation success rates, and per-agent media quality metrics that surface WebRTC-specific issues before they become agent complaints.

### PHP 8 and Security Patches

All ViciStack deployments run PHP 8.2 and include the security patches from SVN 3848+. Our monitoring alerts on any PHP deprecation warnings that might indicate custom code incompatibility.

### What ViciStack Adds Beyond Stock VICIdial

Stock VICIdial gives you the tools. ViciStack gives you the tuning, monitoring, and operational intelligence to use them effectively:

- **Automated dialer tuning** that adjusts dial levels, hopper sizes, and trunk allocation based on real-time answer rates and agent availability.
- **AMD accuracy benchmarking** that measures your actual detection rates against your traffic and recommends setting changes.
- **Cluster health monitoring** with alerting for database replication lag, trunk utilization spikes, and server resource exhaustion.
- **Upgrade planning** that identifies exactly which configuration changes your specific deployment needs before, during, and after an SVN update.

If you are planning an upgrade, running an older SVN revision, or just want to know whether your VICIdial deployment is configured optimally for the features now available, we will tell you exactly where you stand.

> **Get a complete picture of your VICIdial deployment.** [Request a free audit](/free-audit/) -- we will assess your current revision, configuration, and performance, then deliver a specific action plan for getting the most out of VICIdial's latest capabilities.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-34-new-features).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
