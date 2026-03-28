# VICIdial Troubleshooting Guide: Every Error Message Explained With Fixes

**Something broke in your VICIdial system and you need it fixed ten minutes ago. This page covers the 20 errors you'll actually run into, with the exact commands and config changes to resolve each one.**

This isn't a theoretical list. Every error here came from a production VICIdial system — our own deployments, client escalations, or forum threads we've verified. Each one includes the symptom you'll see, what's actually wrong, the fix, and how to stop it from happening again.

Before you start, know where VICIdial keeps its logs:

```bash
# Asterisk telephony log (SIP, channels, calls)
/var/log/asterisk/full

# VICIdial application logs (hopper, keepalive, scripts)
/var/log/astguiclient/

# MySQL slow queries (often the real culprit)
/var/log/mysql/slow-query.log

# Apache/web interface errors
/var/log/httpd/error_log    # CentOS/ViciBox
/var/log/apache2/error.log  # Debian/Ubuntu
```

---

## Error 1: Hopper Not Loading Leads

**Symptom:** The real-time report shows "0 leads in hopper." Agents sit in READY status. Nothing dials.

**Root cause:** The `AST_VDhopper.pl` script isn't finding dialable leads. This script runs every minute via cron, queries `vicidial_list` for leads matching your campaign's dial statuses, call time restrictions, and filters, then loads them into `vicidial_hopper`.

Nine times out of ten, it's one of these:

**Fix steps:**

1. **Check the campaign has active dial statuses:**
   - Admin > Campaigns > your campaign > Dial Statuses
   - At minimum, `NEW` must be checked. If you've already called through the list once, you need the appropriate retry statuses (NA, B, etc.) checked too.

2. **Check the list is actually active:**
   ```bash
   # Quick check from the CLI
   mysql -u cron -p asterisk -e "SELECT list_id, list_name, active FROM vicidial_lists WHERE campaign_id='YOUR_CAMPAIGN';"
   ```
   The `active` column must be `Y`.

3. **Check Local Call Time isn't blocking everything:**
   - Admin > Campaigns > Local Call Time. Set it to `24hours` temporarily for testing. If the hopper loads, your call time window is the issue.

4. **Check Lead Filter isn't zeroing out your leads:**
   - Admin > Campaigns > Lead Filter. Set to `NONE` and see if the hopper loads. If it does, your filter's SQL has a bug.
   - Common filter SQL problem: a syntax error that fails silently. Check `/var/log/astguiclient/VDhopper*.log` for MySQL errors.

5. **Check for a crashed `vicidial_hopper` table:**
   ```bash
   mysql -u cron -p asterisk -e "CHECK TABLE vicidial_hopper;"
   # If it returns "Table is crashed", repair it:
   mysql -u cron -p asterisk -e "REPAIR TABLE vicidial_hopper;"
   ```

6. **Verify the cron job is running:**
   ```bash
   ps aux | grep VDhopper
   # Should show AST_VDhopper.pl running. If not:
   grep VDhopper /var/spool/cron/root
   ```

**Prevention:** Monitor hopper levels in your [real-time report](/blog/vicidial-realtime-report-guide/). Set up a cron alert when the count drops below your agent count. See our full [dial hopper guide](/blog/vicidial-dial-hopper-guide/) for the deep dive.

---

## Error 2: AMD Detecting Too Many Machines

**Symptom:** The vicidial_log shows 40-60% of answered calls classified as `AM` (answering machine). Agents complain they're barely getting live calls. Drop rate looks fine but conversions are in the toilet because actual humans are getting hung up on.

**Root cause:** VICIdial's stock AMD algorithm (`app_amd.c` in Asterisk) is a silence-and-word-count detector that hasn't changed since 2008. Out of the box, it's about 65% accurate. Short greetings from real people ("Yeah?", "Hello?") trip the `INITIALSILENCE` or `MAXWORDS` thresholds. Carrier-injected pre-connect audio (ringback tones sent as media) also throws it off.

**Fix steps:**

1. **Tune AMD parameters in the Asterisk dialplan:**
   Edit `/etc/asterisk/extensions.conf` (or wherever your VICIdial dialplan lives). Find the AMD() invocation and adjust:
   ```
   ; Stock VICIdial values — way too aggressive
   ; AMD(initialSilence,greeting,afterGreetingSilence,totalAnalysisTime,
   ;     minWordLength,betweenWordsSilence,maximumNumberOfWords,silenceThreshold)

   ; Better starting point for US calling:
   AMD(2500,2000,1200,5000,100,50,3,256)
   ```
   The key change: drop `maximumNumberOfWords` from 4 to 3 and increase `afterGreetingSilence` to 1200ms. This gives real humans more time to pause after "Hello?"

2. **Check what your carriers are doing:**
   ```bash
   # Watch live AMD decisions
   asterisk -rx "core set verbose 5"
   # Then in the Asterisk CLI, watch for AMD result lines
   ```
   If you see AMD returning `MACHINE` immediately on connect, the carrier is sending pre-answer audio. Switch carriers or add a pre-AMD delay.

3. **Use AMDSTATUS routing in campaign settings:**
   - Admin > Campaigns > AMD settings
   - Set `AMD Send to Agent` to `Y` for `NOTSURE` results. Better to send a questionable call to an agent than to drop a live person.

4. **Consider killing AMD entirely** if your accuracy is below 70%. Calculate your cost: agents wasting 8 seconds hanging up on voicemails vs. AMD dropping 30% of your live answers.

**Prevention:** Track AMD accuracy weekly by querying `vicidial_log`:
```bash
mysql -u cron -p asterisk -e "
  SELECT status, COUNT(*) as cnt
  FROM vicidial_log
  WHERE call_date > DATE_SUB(NOW(), INTERVAL 7 DAY)
  AND status IN ('AM','AL','AFAX')
  GROUP BY status;"
```
Read the full breakdown in our [AMD configuration guide](/blog/vicidial-amd-guide/).

> **Tired of Tuning AMD Parameters That Haven't Changed Since 2008?**
> ViciStack's AI-powered AMD hits 92-96% accuracy on the same VICIdial system. No dialplan edits. [See How It Works](/free-audit/)

---

## Error 3: Calls Not Recording

**Symptom:** Agents report calls aren't recording. The recording log page shows missing entries. Playback links return 404 or an empty audio file.

**Root cause:** Multiple possible causes — campaign recording setting is wrong, the recording directory is full, file permissions are broken, or the MixMon/Monitor application isn't starting.

**Fix steps:**

1. **Confirm the campaign recording setting:**
   - Admin > Campaigns > Recording. Must be set to `ALLCALLS`, `ALLFORCE`, or `ONDEMAND`. If set to `NEVER`, nothing records.

2. **Check disk space on the recording partition:**
   ```bash
   df -h /var/spool/asterisk/monitor/
   # If >95% full, recordings silently fail
   # Move old recordings or expand the partition
   ```

3. **Verify the recording directory permissions:**
   ```bash
   ls -la /var/spool/asterisk/monitor/
   # Should be owned by asterisk:asterisk with 755 permissions
   chown -R asterisk:asterisk /var/spool/asterisk/monitor/
   chmod 755 /var/spool/asterisk/monitor/
   ```

4. **Check that the recording conversion script is running:**
   ```bash
   ps aux | grep AST_audio_store
   # Also check:
   ls -lt /var/spool/asterisk/monitor/MIX/ | head -20
   # Should show recent .wav files being created
   ```

5. **Verify the Asterisk MixMonitor application is loaded:**
   ```bash
   asterisk -rx "core show application MixMonitor"
   # If "not registered", the module didn't load:
   asterisk -rx "module load app_mixmonitor.so"
   ```

**Prevention:** Set up a nightly cron that counts today's recordings vs. today's calls. Alert if the ratio drops below 90%. See our [call recording storage guide](/blog/vicidial-call-recording/) for archival and compliance details.

---

## Error 4: Agent Screen Frozen / Not Updating

**Symptom:** The agent's browser shows a lead but the countdown timer stopped. Buttons don't respond. The real-time report shows the agent in a status they definitely aren't in. Refreshing the page doesn't help — or it kicks them out entirely.

**Root cause:** The VICIdial agent interface relies on AJAX polling every second (the `vdc_form_display.php` loop). When this polling breaks — usually because of network latency, Apache maxing out, or the browser's connection limit — the screen freezes.

**Fix steps:**

1. **Check Apache's connection count:**
   ```bash
   # How many active Apache processes?
   ps aux | grep httpd | wc -l
   # Compare against MaxClients in httpd.conf
   grep MaxClients /etc/httpd/conf/httpd.conf
   ```
   If you're at 95%+ of MaxClients, bump it up or move the agent screens to a separate web server.

2. **Check MySQL response time:**
   ```bash
   # Run a quick benchmark
   time mysql -u cron -p asterisk -e "SELECT count(*) FROM vicidial_live_agents;"
   # If this takes >1 second, your database needs attention
   ```

3. **Clear the agent's session:**
   - Admin > Users > find the agent > Force Logout
   - Have them clear browser cache and log back in
   - If that doesn't work, check `vicidial_live_agents` for stale entries:
     ```bash
     mysql -u cron -p asterisk -e "
       SELECT user, status, last_update_time
       FROM vicidial_live_agents
       WHERE last_update_time < DATE_SUB(NOW(), INTERVAL 5 MINUTE);"
     ```

4. **Check for JavaScript errors in the browser console:**
   - Press F12 in the agent's browser, go to Console tab. If you see CORS errors or 502 responses, that's a server-side issue (Apache or the load balancer).

**Prevention:** Use a [real-time monitoring dashboard](/blog/vicidial-grafana-realtime-dashboard/) that alerts when any agent's `last_update_time` exceeds 30 seconds. That catches frozen screens before agents report them.

---

## Error 5: SIP Registration Failed

**Symptom:** The Asterisk CLI shows `UNREACHABLE` when you run `sip show peers`. The carrier trunk shows 0 registered. Outbound calls fail with "All circuits are busy now."

**Root cause:** SIP registration requires UDP port 5060 (or TCP/TLS on 5061) to reach the carrier, valid credentials, and DNS resolution of the carrier's SIP server. Any of those breaking kills registration.

**Fix steps:**

1. **Verify the trunk status:**
   ```bash
   asterisk -rx "sip show peers"
   # Look for your carrier trunk — it should say "OK" with a latency number
   # If it says "UNREACHABLE" or "Unregistered", registration failed
   ```

2. **Test network connectivity:**
   ```bash
   # Can you reach the SIP server?
   ping sip.yourcarrier.com
   # Check if port 5060 is open
   nc -zvu sip.yourcarrier.com 5060
   ```

3. **Check firewall rules:**
   ```bash
   iptables -L -n | grep 5060
   # Make sure UDP 5060 outbound is allowed
   # Also verify RTP port range (typically 10000-20000)
   iptables -L -n | grep 10000
   ```

4. **Verify credentials in `sip.conf`:**
   ```bash
   # Find your trunk definition
   grep -A 10 "\[your_trunk_name\]" /etc/asterisk/sip.conf
   # Confirm: host, username, secret, fromuser, fromdomain
   ```

5. **Force a re-register:**
   ```bash
   asterisk -rx "sip reload"
   # Wait 10 seconds, then check again:
   asterisk -rx "sip show registry"
   ```

**Prevention:** Monitor trunk registration with a cron job every 5 minutes. See our full guide on [SIP troubleshooting](/blog/vicidial-sip-troubleshooting/) and [trunk failover](/blog/vicidial-sip-trunk-failover/) for setting up automatic carrier switching when registration drops.

---

## Error 6: Dial Level Not Auto-Adjusting

**Symptom:** You set the campaign to Adaptive predictive dialing, but the dial level stays pinned at 1.0 (or whatever you set manually). Agent wait times are climbing but the system won't dial faster.

**Root cause:** Adaptive dialing requires several conditions to be true simultaneously: enough leads in the hopper, enough agents logged in, the algorithm having enough call history to calculate the right ratio, and the campaign's dial level limitations not capping it.

**Fix steps:**

1. **Confirm the campaign dial method:**
   - Admin > Campaigns > Dial Method. Must be set to `ADAPT_HARD_LIMIT`, `ADAPT_TAPERED`, or `ADAPT_AVERAGE`. If it's set to `RATIO`, the system dials at whatever fixed ratio you entered — it won't adapt.

2. **Check the auto-dial level limitations:**
   - Admin > Campaigns > Auto Dial Level. Check that `Maximum Dial Level` isn't set to 1.0. Also check `Minimum Hopper Level` — if your hopper can't maintain that level, adaptive dialing throttles down.

3. **Verify enough call history exists:**
   ```bash
   # The adaptive algorithm needs recent calls to calculate ratios
   mysql -u cron -p asterisk -e "
     SELECT COUNT(*) FROM vicidial_log
     WHERE campaign_id='YOUR_CAMPAIGN'
     AND call_date > DATE_SUB(NOW(), INTERVAL 30 MINUTE);"
   ```
   If this returns <50 calls, the algorithm doesn't have enough data. It needs at least 20-30 minutes of calling history.

4. **Check that agents are logged in and in READY status:**
   The algorithm won't increase dial level with fewer than 2 available agents. More agents = more data = better predictions.

5. **Review the `Dial Level Difference Target`:**
   - This number (typically -1 to 1) controls how aggressively the system overshoots. A setting of 0 means the system tries to keep zero agents waiting. Negative numbers reduce the dial level.

**Prevention:** Track your dial level over time using the [Grafana dashboard](/blog/vicidial-grafana-dashboards/). If it flatlines, you'll see it before agents start complaining. Our [auto dial level tuning guide](/blog/vicidial-auto-dial-level-tuning/) covers the algorithm internals.

---

## Error 7: Callbacks Not Firing

**Symptom:** Agents set CALLBK dispositions with scheduled times, but the callback never comes. The lead doesn't pop at the scheduled time. Customers call back angry because nobody followed up.

**Root cause:** Callbacks in VICIdial depend on the `AST_VDauto_dial.pl` script finding them in the `vicidial_callbacks` table and loading them into the hopper at the right time. If the lead's status was overwritten, the callback time is in a timezone VICIdial isn't configured for, or the agent who owns it isn't logged in (for USERONLY callbacks), it won't fire.

**Fix steps:**

1. **Check that the callback actually exists in the database:**
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT callback_id, lead_id, campaign_id, status, callback_time, recipient
     FROM vicidial_callbacks
     WHERE lead_id = YOUR_LEAD_ID
     ORDER BY callback_time DESC LIMIT 5;"
   ```
   The `status` should be `ACTIVE`. If it's `INACTIVE` or `DEAD`, something already processed or killed it.

2. **Check the callback recipient type:**
   - `ANYONE` — any agent in the campaign will get it when the time hits. This works most reliably.
   - `USERONLY` — only the specific agent who set it will get it. If that agent isn't logged in at callback time, it just sits there.
   - `USERONLY_CLOSER` — same problem. The named agent must be active.

3. **Verify the callback time and timezone:**
   ```bash
   # Is the server time matching what the agent intended?
   date
   mysql -u cron -p asterisk -e "SELECT NOW();"
   # These should be within 1 second of each other
   ```

4. **Check if the lead status was overwritten:**
   If another process (list reset, lead recycle, another agent calling the same lead) changed the lead's status away from `CALLBK`, the callback won't trigger. Query the lead directly:
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT lead_id, status, last_local_call_time
     FROM vicidial_list
     WHERE lead_id = YOUR_LEAD_ID;"
   ```

**Prevention:** Use `ANYONE` callbacks unless the agent truly needs to own the relationship. Schedule a daily report that counts unresolved callbacks older than 24 hours. Our full [callback automation guide](/blog/vicidial-callback-automation/) covers the setup.

---

## Error 8: DNC List Not Filtering

**Symptom:** Agents are calling numbers that are on your Do Not Call list. You loaded the DNC file, but VICIdial dials those numbers anyway.

**Root cause:** VICIdial checks DNC lists at hopper load time, not at dial time. If the DNC list wasn't loaded correctly, the campaign isn't configured to use it, or the list was loaded after the leads already entered the hopper, those numbers will still get dialed.

**Fix steps:**

1. **Confirm DNC checking is enabled on the campaign:**
   - Admin > Campaigns > Campaign Detail > `Use Internal DNC List` must be set to `Y` or `AREACODE`.
   - If you're using the campaign-specific DNC, check `Use Campaign DNC List` too.

2. **Verify the DNC list loaded correctly:**
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT COUNT(*) FROM vicidial_dnc WHERE phone_number = '5551234567';"
   # Replace with a number that should be blocked
   # If count is 0, the number wasn't loaded
   ```

3. **Verify the phone number format matches:**
   DNC entries and lead phone numbers must be in the same format. If your DNC has `15551234567` (with leading 1) but your leads have `5551234567` (10 digits), they won't match.
   ```bash
   # Check what format your DNC entries use
   mysql -u cron -p asterisk -e "
     SELECT phone_number FROM vicidial_dnc LIMIT 10;"
   # Compare against your leads
   mysql -u cron -p asterisk -e "
     SELECT phone_number FROM vicidial_list WHERE list_id=YOUR_LIST LIMIT 10;"
   ```

4. **Flush and reload the hopper** after adding DNC entries:
   ```bash
   mysql -u cron -p asterisk -e "DELETE FROM vicidial_hopper WHERE campaign_id='YOUR_CAMPAIGN';"
   # The hopper will repopulate on the next cron cycle (within 1 minute)
   # and DNC numbers will be excluded
   ```

**Prevention:** Load DNC lists before importing leads. Schedule weekly DNC updates via cron. See our [DNC management guide](/blog/vicidial-dnc-management/) for handling federal, state, and internal DNC lists with proper audit trails.

---

## Error 9: Real-Time Report Blank or Not Updating

**Symptom:** Admin opens the real-time report and sees nothing — no agents, no calls, no stats. Or the report shows data but the numbers haven't changed in minutes.

**Root cause:** The real-time report pulls from `vicidial_live_agents`, `vicidial_auto_calls`, and `server_updater` tables. If the cron scripts that populate these tables stopped running, or MySQL is too slow to respond, the report goes blank.

**Fix steps:**

1. **Check that the keepalive scripts are running:**
   ```bash
   # The most important cron scripts
   ps aux | grep -E "keepalive|AST_manager"
   # You should see:
   # ADMIN_keepalive_ALL.pl
   # AST_manager_listen.pl
   # AST_manager_send.pl
   ```
   If any are missing, restart them:
   ```bash
   /usr/share/astguiclient/ADMIN_keepalive_ALL.pl --cu3way
   ```

2. **Check the `server_updater` table:**
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT server_ip, last_update FROM server_updater;"
   # The last_update should be within the last 30 seconds
   # If it's stale, the keepalive isn't updating
   ```

3. **Check MySQL performance:**
   ```bash
   mysqladmin -u cron -p processlist | wc -l
   # If there are hundreds of connections, you have a query pileup
   # Check for table locks:
   mysql -u cron -p asterisk -e "SHOW OPEN TABLES WHERE In_use > 0;"
   ```

4. **Check the vicidial_live_agents table for corruption:**
   ```bash
   mysql -u cron -p asterisk -e "CHECK TABLE vicidial_live_agents;"
   # If crashed:
   mysql -u cron -p asterisk -e "REPAIR TABLE vicidial_live_agents;"
   ```

**Prevention:** Set up the [Grafana real-time dashboard](/blog/vicidial-grafana-realtime-dashboard/) as a second monitoring layer. If the VICIdial report blanks out, Grafana will still show you what's happening in the database.

---

## Error 10: API Authentication Failed

**Symptom:** Your CRM or external integration sends API calls to VICIdial and gets back `ERROR: Invalid Username/Password` or `ERROR: auth USER DOES NOT HAVE PERMISSION TO USE THE API`. Scripts that worked yesterday suddenly stop.

**Root cause:** The VICIdial Non-Agent API (`/agc/api.php` and `/vicidial/non_agent_api.php`) requires a user account with API access enabled and the correct API permissions. Password changes, IP restrictions, or API setting changes on the user account will break existing integrations.

**Fix steps:**

1. **Verify the API user exists and has API access:**
   - Admin > Users > find the API user
   - `API Access` must be set to `1`
   - `API Access Level` should be `1` (or higher for modify access)
   - Check that the `modify_same_user_level` permission allows API access

2. **Test the API directly:**
   ```bash
   curl "http://YOUR_SERVER/vicidial/non_agent_api.php?source=test&user=APIUSER&pass=APIPASS&function=version"
   # Should return: VERSION: 2.x-xxx
   # If it returns an error, the credentials or permissions are wrong
   ```

3. **Check for IP restrictions:**
   - Admin > Users > `API Allowed IPs`. If this field is populated, only listed IPs can use the API. An empty field means all IPs are allowed.

4. **Verify the API function permissions:**
   Each API function requires specific permissions. For example, `add_lead` requires `modify_leads=1` on the user account. Check the admin interface for the specific permission related to the failing function.

5. **Check Apache logs for the actual error:**
   ```bash
   tail -100 /var/log/httpd/error_log | grep api
   ```

**Prevention:** Create a dedicated API user account — never share it with agent logins. Document the credentials in a secure location. Monitor API response codes in your integration. See our [API integration guide](/blog/vicidial-api-integration/) for complete examples.

---

## Error 11: "There Is a Time Synchronization Problem"

**Symptom:** Agents see a red banner at the top of the agent screen: "There is a time synchronization problem on this server." Sometimes the screen works fine despite the warning; other times the agent screen partially freezes.

**Root cause:** VICIdial's agent interface checks the `server_updater` table and compares the dialer server time, web server time, and database time. If any pair differs by more than 3 seconds, you get the warning. Common cause: NTP isn't running, or the system is under such heavy load that cron scripts fall behind.

**Fix steps:**

1. **Check the time difference between components:**
   ```bash
   # Server system time
   date +%s
   # MySQL time
   mysql -u cron -p asterisk -e "SELECT UNIX_TIMESTAMP();"
   # Compare — they should differ by <1 second
   ```

2. **Fix NTP synchronization:**
   ```bash
   # Check if NTP is running
   systemctl status ntpd      # CentOS 7 / ViciBox
   systemctl status chronyd   # CentOS 8+

   # If not running, start and enable it:
   systemctl start ntpd && systemctl enable ntpd

   # Force an immediate sync:
   systemctl stop ntpd
   ntpdate -u pool.ntp.org
   systemctl start ntpd
   ```
   Do NOT run `ntpdate` while `ntpd` is already running — they fight over the same socket.

3. **Check system load:**
   ```bash
   uptime
   # If load average is above (2 x CPU cores), the system is overloaded
   # and cron scripts can't keep up, causing the time drift warning
   top -bn1 | head -20
   ```

4. **For multi-server clusters, sync ALL servers to the same NTP source:**
   ```bash
   # On each server in the cluster
   grep ^server /etc/ntp.conf
   # All should point to the same NTP pool or your own internal NTP server
   ```

**Prevention:** Add a cron job on every VICIdial server to check NTP drift:
```bash
# Add to root's crontab
*/5 * * * * /usr/sbin/ntpdate -u pool.ntp.org > /dev/null 2>&1
```

---

## Error 12: Database Table Crashed

**Symptom:** Random VICIdial functions stop working. The hopper won't load. Agents can't disposition calls. The admin GUI shows partial data or PHP errors. The MySQL error log shows: `Table './asterisk/vicidial_live_agents' is marked as crashed and should be repaired`.

**Root cause:** MyISAM tables (which VICIdial uses heavily) crash when MySQL gets killed mid-write — power failure, OOM kill, forced restart, or running out of disk space. The most commonly crashed tables: `vicidial_live_agents`, `vicidial_auto_calls`, `vicidial_hopper`, `vicidial_list`, and `vicidial_log`.

**Fix steps:**

1. **Identify which tables are crashed:**
   ```bash
   mysqlcheck -u root -p --check asterisk
   # This checks every table in the asterisk database
   # Crashed tables show "Table is marked as crashed"
   ```

2. **Repair specific tables:**
   ```bash
   mysql -u root -p asterisk -e "REPAIR TABLE vicidial_live_agents;"
   mysql -u root -p asterisk -e "REPAIR TABLE vicidial_auto_calls;"
   mysql -u root -p asterisk -e "REPAIR TABLE vicidial_hopper;"
   ```

3. **If REPAIR TABLE fails, try with USE_FRM:**
   ```bash
   mysql -u root -p asterisk -e "REPAIR TABLE vicidial_list USE_FRM;"
   # Warning: USE_FRM can lose data from the last few rows written before the crash
   ```

4. **Nuclear option — repair all tables at once:**
   ```bash
   mysqlcheck -u root -p --auto-repair --optimize asterisk
   # This checks, repairs, and optimizes every table
   ```

5. **Check disk space (the usual cause):**
   ```bash
   df -h
   # If any partition is >95% full, that's likely why MySQL crashed
   # Common space hogs: call recordings, CDR logs, vicidial_log table
   ```

**Prevention:** Schedule nightly table checks:
```bash
# Add to root's crontab at 2 AM (low-traffic time)
0 2 * * * mysqlcheck -u root -pYOUR_PASSWORD --auto-repair asterisk > /var/log/mysqlcheck.log 2>&1
```
Consider converting high-write tables to InnoDB (crash-safe). See our [database maintenance guide](/blog/vicidial-database-maintenance/) and [MySQL optimization guide](/blog/vicidial-mysql-optimization/) for the full story.

---

## Error 13: One-Way Audio on Calls

**Symptom:** Agent can hear the customer but the customer can't hear the agent (or the reverse). Internal calls between extensions work fine. The problem appears only on outbound calls through a SIP trunk.

**Root cause:** Almost always NAT. The SDP body in the SIP INVITE contains a private IP address (10.x.x.x, 172.16.x.x, 192.168.x.x). The carrier sends RTP media to that private IP, which is unreachable from the public internet.

**Fix steps:**

1. **Check the SDP for private IPs:**
   ```bash
   # Capture a live call's SIP traffic
   sngrep -d eth0 port 5060
   # Look at the INVITE message, find the "c=" line in the SDP body
   # BAD: c=IN IP4 10.0.0.50
   # GOOD: c=IN IP4 203.0.113.50
   ```

2. **Fix NAT in sip.conf:**
   ```ini
   ; In /etc/asterisk/sip.conf, [general] section:
   externip=YOUR.PUBLIC.IP.HERE
   localnet=10.0.0.0/255.0.0.0
   localnet=172.16.0.0/255.240.0.0
   localnet=192.168.0.0/255.255.0.0
   nat=force_rport,comedia
   ```

3. **Open RTP port range in your firewall:**
   ```bash
   # Check current RTP port range
   grep rtpstart /etc/asterisk/rtp.conf
   grep rtpend /etc/asterisk/rtp.conf
   # Default: 10000-20000. Open these in the firewall:
   iptables -A INPUT -p udp --dport 10000:20000 -j ACCEPT
   ```

4. **Reload SIP after changes:**
   ```bash
   asterisk -rx "sip reload"
   asterisk -rx "rtp set debug on"
   # Make a test call and watch for RTP packets flowing in both directions
   ```

**Prevention:** Use `sngrep` regularly to spot-check SDP bodies. For the full SIP debugging workflow with packet captures, see our [SIP troubleshooting guide](/blog/vicidial-sip-troubleshooting/).

---

## Error 14: "All Circuits Are Busy Now"

**Symptom:** Outbound calls fail immediately. The Asterisk CLI shows `CHANUNAVAIL` or `CONGESTION`. Agents hear a fast busy or silence.

**Root cause:** Three possibilities: (1) the SIP trunk isn't registered, (2) you've exceeded the carrier's channel limit, or (3) the carrier rejected the call with a SIP 503/480/486 response.

**Fix steps:**

1. **Check trunk registration first (see Error 5):**
   ```bash
   asterisk -rx "sip show peers" | grep -i your_trunk
   ```

2. **Check current channel usage vs. capacity:**
   ```bash
   asterisk -rx "core show channels" | tail -1
   # Shows "X active channels"
   # Compare against your carrier's channel limit
   ```

3. **Look at the specific SIP response code:**
   ```bash
   # Watch live Asterisk CLI output during a failed call
   asterisk -rvvv
   # Look for lines like:
   # "Got SIP response 503 Service Unavailable"
   # "Got SIP response 486 Busy Here"
   ```
   - **503** = carrier overloaded or your account is suspended
   - **486** = specific number is busy
   - **404** = number doesn't exist
   - **480** = temporarily unavailable

4. **If you're hitting channel limits, add a second carrier:**
   Set up [SIP trunk failover](/blog/vicidial-sip-trunk-failover/) so calls automatically route to a backup carrier when the primary is full.

5. **Check for carrier-side issues:**
   ```bash
   # Quick test — try dialing a known-good number manually
   asterisk -rx "originate SIP/your_trunk/15551234567 application Playback hello-world"
   ```

**Prevention:** Monitor active channels and set alerts at 80% of your carrier limit. Always have at least two carriers configured. Review our [carrier selection guide](/blog/vicidial-carrier-selection/) for picking reliable providers.

---

## Error 15: Agents Getting Logged Out Randomly

**Symptom:** Agents get kicked to the login screen mid-shift. No warning, no error — the page just reloads to the login form. Sometimes it happens to all agents at once; sometimes just one.

**Root cause:** Three common causes: (1) the `ADMIN_keepalive_ALL.pl` script triggered a mass logout at the end-of-day time, (2) Apache session timeout expired, or (3) PHP's `session.gc_maxlifetime` is too short and the session was garbage collected.

**Fix steps:**

1. **Check if it was a timeclock auto-logout:**
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT timeclock_end_of_day FROM system_settings;"
   # If this is set (e.g., 0100 for 1 AM), all agents get logged out at that time
   ```
   The `timeclock_auto_logout.pl` script runs once daily based on this setting.

2. **Check PHP session settings:**
   ```bash
   php -i | grep session.gc_maxlifetime
   # Default is 1440 seconds (24 minutes). For an 8-hour shift:
   # Set to at least 28800 in /etc/php.ini:
   # session.gc_maxlifetime = 28800
   ```
   Restart Apache after changing: `systemctl restart httpd`

3. **Check Apache's timeout setting:**
   ```bash
   grep -i timeout /etc/httpd/conf/httpd.conf
   # KeepAliveTimeout and Timeout should be reasonable (300+)
   ```

4. **Check for memory pressure causing Apache to kill processes:**
   ```bash
   dmesg | grep -i "out of memory" | tail -5
   # If OOM killer is running, you need more RAM or fewer Apache processes
   ```

**Prevention:** Set `timeclock_end_of_day` to a time when no agents are working (or leave it empty to disable). Increase PHP session lifetime. Monitor Apache process count vs. available memory.

---

## Error 16: Calls Dropping After a Few Seconds

**Symptom:** Calls connect (you hear the remote party answer) but disconnect within 2-10 seconds. No error on the agent screen — the call just ends. The Asterisk CDR shows very short `billsec` values.

**Root cause:** Usually a codec mismatch or the carrier's billing system rejecting the call after it answers. Less commonly, a NAT issue where the ACK never arrives at the carrier and it tears down the dialog.

**Fix steps:**

1. **Check the hangup cause:**
   ```bash
   # Query the last 20 short calls
   mysql -u cron -p asterisk -e "
     SELECT uniqueid, src, dst, billsec, hangupcause
     FROM asterisk.cdr
     WHERE calldate > DATE_SUB(NOW(), INTERVAL 1 HOUR)
     AND billsec < 10 AND billsec > 0
     ORDER BY calldate DESC LIMIT 20;"
   ```
   Common hangup causes:
   - **16** = Normal clearing (carrier hung up — check if your SIP session timers are too aggressive)
   - **21** = Call rejected (carrier rejected after answer — usually a billing issue)
   - **31** = Normal unspecified (often codec negotiation failure)
   - **38** = Network out of order (NAT/firewall issue)

2. **Check codec negotiation:**
   ```bash
   asterisk -rx "sip show peer your_trunk"
   # Look at "Codecs" line. Ensure both sides support the same codec
   # Common fix: allow only ulaw and alaw
   ```
   In `/etc/asterisk/sip.conf` for the trunk:
   ```ini
   disallow=all
   allow=ulaw
   allow=alaw
   ```

3. **Check SIP session timers:**
   ```bash
   grep session-timer /etc/asterisk/sip.conf
   # If set to "originate" or "accept", try changing to:
   session-timers=refuse
   ```

4. **Verify your carrier account is in good standing** — prepaid accounts with low balance often answer and then disconnect after a few seconds when the billing system kicks in.

**Prevention:** Track average call duration daily. If the mean drops below 30 seconds, investigate immediately. Alert on `billsec < 5` spikes.

---

## Error 17: Campaign Not Dialing Automatically

**Symptom:** Campaign is active, agents are logged in and in READY, hopper has leads, but nothing dials. The auto-dial system acts like it's turned off.

**Root cause:** The `AST_VDadapt.pl` and `AST_VDauto_dial.pl` scripts control automatic dialing. If they're not running, nothing will auto-dial regardless of campaign settings.

**Fix steps:**

1. **Check that auto-dial scripts are running:**
   ```bash
   ps aux | grep -E "AST_VDadapt|AST_VDauto_dial"
   # Both should appear. If not, check the crontab:
   grep -E "VDadapt|VDauto_dial" /var/spool/cron/root
   ```

2. **Confirm the campaign dial method:**
   - Admin > Campaigns > Dial Method. If set to `MANUAL`, agents must click to dial each call. For auto-dialing, set to `RATIO` or one of the `ADAPT_*` modes.

3. **Check if the campaign dial level is set to 0:**
   - Admin > Campaigns > Auto Dial Level. If the dial level is 0, nothing dials. Set it to 1.0 minimum.

4. **Verify agents are truly in READY status:**
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT user, status, campaign_id
     FROM vicidial_live_agents
     WHERE campaign_id='YOUR_CAMPAIGN';"
   # Agents must show status 'READY' — not 'PAUSED', 'INCALL', etc.
   ```

5. **Check the server's dial capacity:**
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT server_ip, max_vicidial_trunks, answer_transfer_agent
     FROM servers WHERE active='Y';"
   # If max_vicidial_trunks is set too low or is 0, dialing is restricted
   ```

**Prevention:** After making campaign changes, always test with a small list before going to production. Monitor the keepalive scripts with `ps aux | grep AST_VD` as part of your pre-shift checklist.

---

## Error 18: Recordings Not Downloading / 404 Error

**Symptom:** The admin GUI shows a recording entry, but clicking the play or download link returns a 404 Not Found error. The recording was made but the file can't be found.

**Root cause:** VICIdial records calls as `.wav` files in `/var/spool/asterisk/monitor/` and then the `AST_audio_store.pl` script converts and moves them. If that script isn't running, or the file was moved/archived/deleted before the link was updated, you get a 404.

**Fix steps:**

1. **Check if the raw recording file exists:**
   ```bash
   # The recording filename is typically in the vicidial_recording_log
   mysql -u cron -p asterisk -e "
     SELECT recording_id, filename, location
     FROM recording_log
     WHERE lead_id = YOUR_LEAD_ID
     ORDER BY start_time DESC LIMIT 5;"
   # Then check for the file:
   find /var/spool/asterisk/monitor/ -name "*RECORDING_FILENAME*" 2>/dev/null
   ```

2. **Check if the audio store script is running:**
   ```bash
   ps aux | grep AST_audio_store
   # If not running:
   /usr/share/astguiclient/AST_audio_store.pl
   ```

3. **Check the web server recording path configuration:**
   ```bash
   mysql -u cron -p asterisk -e "
     SELECT recording_web_link, alt_server_ip FROM servers WHERE active='Y';"
   # Verify this URL actually resolves to the recording directory
   ```

4. **If recordings exist on disk but aren't web-accessible:**
   ```bash
   ls -la /var/spool/asterisk/monitor/MIX/
   # Verify Apache has a valid Alias pointing to this directory
   grep -r "monitor" /etc/httpd/conf.d/
   ```

**Prevention:** Set up a nightly script that cross-checks `recording_log` entries against actual files on disk. Archive old recordings to a separate server or S3 bucket. Our [call recording storage optimization guide](/blog/vicidial-call-recording-storage-optimization/) covers this in detail.

---

## Error 19: Hotkeys Not Working in Agent Screen

**Symptom:** Agent presses a hotkey number (1-9) to disposition a call but nothing happens. They have to click the disposition button manually. This slows down their wrap-up time by 5-10 seconds per call.

**Root cause:** Hotkeys must be enabled at both the campaign level and correctly mapped in the campaign's Hotkey settings. If the agent screen focus is on a different input field (like the comments box), keystrokes go to that field instead of the hotkey handler.

**Fix steps:**

1. **Enable hotkeys in the campaign:**
   - Admin > Campaigns > Detail > `Hotkey Active` must be set to `1`
   - Admin > Campaigns > Hotkeys tab > Map each number (1-9) to a disposition status

2. **Verify the agent is clicking outside of text fields before pressing hotkeys:**
   Hotkeys only fire when the browser focus is on the main page, not inside a form field. Train agents to click the gray area of the screen before pressing the hotkey number.

3. **Check for JavaScript conflicts:**
   - If you have custom agent screen modifications (iFrame pop-ups, CRM integrations), they may capture keystroke events before VICIdial's hotkey handler sees them.
   - Press F12 in the agent browser, go to Console, and check for JavaScript errors.

4. **Test with a clean browser profile:**
   Extensions like password managers or accessibility tools can intercept keystrokes. Have the agent try in an incognito/private window.

**Prevention:** Include hotkey usage in agent training. Test hotkeys after any agent screen customization changes.

---

## Error 20: SSL/HTTPS Not Working on Agent Screen

**Symptom:** Agents get browser security warnings when accessing the VICIdial interface. Chrome shows "Your connection is not private" (ERR_CERT_AUTHORITY_INVALID). WebRTC phone won't register because the browser blocks mixed content.

**Root cause:** The SSL certificate is expired, self-signed, or the server's Apache configuration isn't set up for HTTPS. WebRTC (used by VICIdial's built-in webphone) requires a valid HTTPS connection — browsers won't allow microphone access over plain HTTP.

**Fix steps:**

1. **Check the current certificate:**
   ```bash
   # View certificate details
   openssl s_client -connect YOUR_SERVER:443 -servername YOUR_DOMAIN </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
   # Check the "notAfter" date
   ```

2. **Install or renew with Let's Encrypt:**
   ```bash
   # Install certbot if not present
   yum install certbot python3-certbot-apache   # CentOS
   apt install certbot python3-certbot-apache    # Debian

   # Get a certificate
   certbot --apache -d your-vicidial-domain.com

   # Auto-renew via cron
   echo "0 3 * * * certbot renew --quiet" >> /var/spool/cron/root
   ```

3. **Configure Apache for HTTPS:**
   ```bash
   # Verify the SSL virtual host is configured
   grep -r "SSLCertificateFile" /etc/httpd/conf.d/
   # The certbot command usually handles this automatically
   ```

4. **Update VICIdial to use HTTPS URLs:**
   - Admin > Servers > your server > `External Server IP` — change from `http://` to `https://YOUR_DOMAIN`
   - Admin > System Settings > verify `Web Server` and `Admin Web Server` fields use the HTTPS URL

5. **For WebRTC/webphone, verify the wss:// endpoint:**
   The webphone needs a secure WebSocket connection. Check that port 8089 (default for Asterisk HTTPS WebSocket) is open and uses the same certificate.

**Prevention:** Set certbot auto-renewal. Monitor certificate expiry dates — Let's Encrypt certs expire every 90 days. See our [WebRTC setup guide](/blog/vicidial-webrtc-setup/) for the full secure phone configuration.

---

## Quick Reference: Log Files and Diagnostic Commands

Every VICIdial troubleshooter needs these memorized:

| What You Need | Where to Find It |
|---|---|
| Asterisk call log | `/var/log/asterisk/full` |
| VICIdial application logs | `/var/log/astguiclient/` |
| Hopper debug log | `/var/log/astguiclient/VDhopper*.log` |
| Apache error log | `/var/log/httpd/error_log` |
| MySQL slow queries | `/var/log/mysql/slow-query.log` |
| MySQL error log | `/var/log/mysql/error.log` |
| Call recordings | `/var/spool/asterisk/monitor/MIX/` |
| SIP configuration | `/etc/asterisk/sip.conf` |
| Dialplan | `/etc/asterisk/extensions.conf` |
| RTP port config | `/etc/asterisk/rtp.conf` |
| VICIdial crontab | `/var/spool/cron/root` |

**Essential diagnostic commands:**

```bash
# Check all running VICIdial processes
ps aux | grep -E "AST_|ADMIN_|VD" | grep -v grep

# SIP trunk status
asterisk -rx "sip show peers"
asterisk -rx "sip show registry"

# Active calls and channels
asterisk -rx "core show channels"
asterisk -rx "core show calls"

# Database health
mysqlcheck -u root -p --check asterisk

# System load and resources
uptime && free -h && df -h

# Live SIP packet capture
sngrep -d eth0 port 5060
```

---

## FAQ

**Q: Where are VICIdial log files located?**
A: Asterisk logs are in `/var/log/asterisk/full`. VICIdial application logs (hopper, keepalive, dialer scripts) are in `/var/log/astguiclient/`. Apache logs are in `/var/log/httpd/error_log` (CentOS) or `/var/log/apache2/error.log` (Debian). MySQL logs are in `/var/log/mysql/`.

**Q: How do I check if the VICIdial cron jobs are running?**
A: Run `ps aux | grep AST_` to see active processes. The essential ones are `ADMIN_keepalive_ALL.pl`, `AST_VDhopper.pl`, `AST_VDadapt.pl`, `AST_VDauto_dial.pl`, `AST_manager_listen.pl`, and `AST_manager_send.pl`. If any are missing, check `crontab -l` as root to see if they're scheduled.

**Q: My VICIdial system is slow. What should I check first?**
A: Run `uptime` to check load average (should be below 2x your CPU core count). Then `free -h` to check memory (if swap usage is high, you need more RAM). Then `df -h` to check disk space. Then `mysqladmin -u root -p processlist` to check for query pileups. Nine times out of ten, it's either disk space, RAM, or a crashed MySQL table.

**Q: How do I repair a crashed MySQL table in VICIdial?**
A: Run `mysqlcheck -u root -p --auto-repair asterisk` to check and repair all tables. For a specific table: `mysql -u root -p asterisk -e "REPAIR TABLE vicidial_live_agents;"`. If standard repair fails, try `REPAIR TABLE tablename USE_FRM;` — but know this can lose the last few rows written before the crash.

**Q: Can I use VICIdial without AMD (Answering Machine Detection)?**
A: Yes. Many high-volume centers turn AMD off entirely because stock AMD is only about 65% accurate. Without AMD, agents hear every answered call — including voicemails — but you eliminate false positives where live humans get dropped. The trade-off depends on your call volume and agent cost. Read our [AMD guide](/blog/vicidial-amd-guide/) for the full analysis.

**Q: Why does the agent screen show "DEAD" call status?**
A: A DEAD call means the audio channel was lost — the remote party hung up, the carrier dropped the call, or there was a network interruption. If you're seeing excessive DEAD calls, check your SIP trunk stability with `asterisk -rx "sip show peers"` and review the Asterisk CLI output during the calls for SIP BYE messages or CANCEL messages.

**Q: How do I change the maximum number of calls VICIdial dials simultaneously?**
A: Admin > Servers > your server > `Max VICIdial Trunks`. This limits the total outbound channels. Also check Admin > Campaigns > `Auto Dial Level` and `Maximum Dial Level` which control per-campaign limits. For the adaptive algorithm internals, see our [predictive dialer settings guide](/blog/vicidial-predictive-dialer-settings/).

**Q: VICIdial keeps calling the same lead multiple times. How do I stop it?**
A: Check three things: (1) the lead's status after each call — if it's being set back to a dialable status (like `NEW`) by a script or list reset, it'll get called again. (2) Lead Recycling settings — if enabled, it can re-insert recently called leads. (3) Callback entries — old unresolved callbacks can trigger re-dials. Query the lead in `vicidial_list` and check `vicidial_callbacks` for active entries. See our [lead recycling guide](/blog/vicidial-lead-recycling/) for proper configuration.

**Q: How do I back up my VICIdial system?**
A: Back up the MySQL database: `mysqldump -u root -p --all-databases | gzip > /backup/vicidial-$(date +%Y%m%d).sql.gz`. Back up configuration files: `tar czf /backup/asterisk-config-$(date +%Y%m%d).tar.gz /etc/asterisk/`. Back up recordings: rsync them to a remote server nightly. Our [disaster recovery guide](/blog/vicidial-disaster-recovery/) covers automated backup schedules and failover procedures.

**Q: Why do I see "Phone login not active" when an agent tries to log in?**
A: The phone extension (SIP phone) the agent selected isn't configured or isn't active. Admin > Phones > find the phone entry > ensure `Active` is set to `Y` and the `Server IP` matches the server the agent is connecting to. Also verify the SIP extension exists in Asterisk: `asterisk -rx "sip show peer EXTENSION"`. See our error reference at [/errors/phone-login-not-active/](/errors/phone-login-not-active/).

**Q: How many agents can a single VICIdial server handle?**
A: A properly configured VICIdial server with 8 CPU cores and 16GB RAM typically handles 50-80 concurrent agents. At 100+ agents, you should split into a [multi-server cluster](/blog/vicidial-cluster-guide/) with separate database, web, and telephony servers. CPU is usually the bottleneck — Asterisk's call processing is single-threaded per channel.

**Q: Can I run VICIdial in Docker?**
A: It's possible but not recommended for production by the VICIdial developers. Asterisk requires direct hardware access (or DAHDI timing) for conference bridges and recording. Check our [Docker deployment guide](/blog/vicidial-docker-deployment/) for the caveats and configuration required.

---

> **Your VICIdial System Needs More Than Troubleshooting.**
> ViciStack optimizes the entire stack — from Asterisk to the database to your carrier routing. Most clients see a 50% conversion increase in two weeks. [Get Your Free Audit](/free-audit/)

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-troubleshooting-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
