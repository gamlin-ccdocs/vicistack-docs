# VICIdial Call Recording: Storage, Compliance & Archival

**Last updated: March 2026 | Reading time: ~20 minutes**

Call recording in VICIdial is one of those features that seems simple until it isn't. You enable [campaign recording](/settings/campaign-recording/), calls get recorded, files land on disk. Easy. Then six months later you are staring at a server with 2 TB of WAV files, your compliance team is asking about PCI-DSS, a client wants recordings from 14 months ago that you already deleted, and you realize you never actually planned any of this.

This guide covers everything about VICIdial call recording that matters in a production environment: how the recording system works at the [Asterisk](/glossary/asterisk/) level, storage format decisions and their real-world implications, capacity planning with actual numbers, compliance requirements by jurisdiction and industry, archival strategies that scale, and the database tables that track it all.

Whether you are running 10 agents or 500, if you record calls, you need a recording strategy. Not having one is how operations end up with compliance violations, disk space emergencies at 2 AM, and the inability to produce recordings when a regulator or a lawyer asks for them.

**ViciStack handles recording infrastructure as part of every deployment** --- properly sized storage, automated archival pipelines, and compliance-ready configurations included. [Skip the storage headaches.](/pricing/)

---

## How VICIdial Call Recording Actually Works

Understanding the recording pipeline is essential before you can optimize, troubleshoot, or build compliance processes around it. Here is what happens when a call is recorded in VICIdial.

### The MixMon Application

VICIdial uses Asterisk's `MixMon()` application (or `Monitor()` on older installations) to capture call audio. When an agent connects to a call and recording is enabled, VICIdial's `vicidial_manager` process sends an Asterisk Manager Interface ([AMI](/glossary/asterisk-manager-interface/)) command that injects `MixMon()` into the active channel.

MixMon captures both sides of the conversation --- the agent's audio and the caller's audio --- and mixes them into a single file. This is called "mixed mode" recording. The alternative, "stereo" recording (agent on one channel, caller on the other), is available but rarely used in production because it doubles storage requirements and most QA workflows expect a single mixed file.

The recording command looks like this internally:

```
Action: Originate
Channel: Local/5555@default
Application: MixMonitor
Data: /var/spool/asterisk/monitor/MIXmonitor/20260318-093045_1234567890_6001_campaign1.wav,b
```

The `b` option tells MixMon to only begin recording when the call is bridged (when both parties are connected), which avoids recording ring time and IVR navigation.

### File Naming and Location

VICIdial stores recordings in a predictable directory structure:

```
/var/spool/asterisk/monitor/MIXmonitor/
├── 20260318-093045_1234567890_6001_campaign1.wav
├── 20260318-093112_5551234567_6002_campaign1.wav
└── ...
```

The default filename format is:
```
YYYYMMDD-HHMMSS_PHONENUMBER_AGENTEXTENSION_CAMPAIGNID.wav
```

This naming convention means files are naturally sorted chronologically and you can identify the phone number, agent, and campaign from the filename alone. VICIdial also tracks this information in the database (covered below), but the filename structure is a useful fallback for manual file browsing.

The default path `/var/spool/asterisk/monitor/MIXmonitor/` is configurable. On a [cluster deployment](/blog/vicidial-cluster-guide/), each telephony server writes recordings locally to its own disk. The web server needs access to these files for playback, which typically means either NFS mounts or a centralized recording server that pulls files from each dialer.

### The recording_log Table

Every recording that VICIdial creates gets an entry in the `recording_log` table in the `asterisk` database. This is the authoritative index of all recordings:

| Column | Purpose |
|--------|---------|
| `recording_id` | Auto-increment primary key |
| `channel` | The Asterisk channel that was recorded |
| `server_ip` | Which server made the recording |
| `extension` | Agent extension |
| `start_time` | When recording began |
| `start_epoch` | Unix timestamp of recording start |
| `end_time` | When recording ended |
| `end_epoch` | Unix timestamp of recording end |
| `length_in_sec` | Duration of the recording in seconds |
| `length_in_min` | Duration in minutes (decimal) |
| `filename` | Full filename of the recording file |
| `location` | Full filesystem path or URL to the recording |
| `lead_id` | Links to the `vicidial_list` lead record |
| `user` | The agent's user ID |
| `vicidial_id` | Links to the `vicidial_log` or `vicidial_closer_log` entry |

This table is what powers recording search and playback in the VICIdial admin interface. When a manager searches for recordings by date range, phone number, or agent, VICIdial queries `recording_log` and returns links to the audio files.

### The recording_access Table

VICIdial 2.14+ includes a `recording_access` table that logs who accessed which recordings and when. This is critical for compliance --- particularly [HIPAA](/glossary/call-recording-compliance/) and PCI-DSS --- where you need an audit trail showing which personnel listened to which recordings.

```sql
-- Check who has accessed recordings for a specific lead
SELECT ra.user, ra.access_datetime, rl.filename
FROM recording_access ra
JOIN recording_log rl ON ra.recording_id = rl.recording_id
WHERE rl.lead_id = 12345
ORDER BY ra.access_datetime DESC;
```

---

## Recording Formats: WAV vs MP3

This is one of the most impactful decisions you will make about your recording system, and it is not as straightforward as "MP3 is smaller, use MP3."

### WAV (Uncompressed)

VICIdial records in WAV format by default. Specifically, 8 kHz, 16-bit, mono PCM WAV files. This is the native format of [G.711](/glossary/codec/) telephony audio.

**Storage per minute:** approximately 960 KB/minute (about 1 MB/minute for easy math).

**Advantages:**
- No CPU overhead during recording --- the audio is already in PCM format from the Asterisk channel
- Lossless quality --- you capture every bit of audio fidelity the telephone network provides
- Universally compatible --- every audio player, transcription service, and forensic tool handles WAV natively
- Required by some compliance frameworks that mandate "original quality" recordings

**Disadvantages:**
- Large file sizes --- a 5-minute call is approximately 5 MB
- Expensive to store long-term at scale

### MP3 (Compressed)

MP3 encoding can be enabled in VICIdial by installing the LAME encoder and configuring the recording format. MP3 at 32 kbps (which is more than adequate for speech-quality telephony audio) reduces file sizes by approximately 8-10x compared to WAV.

**Storage per minute:** approximately 120 KB/minute at 32 kbps, or 240 KB/minute at 64 kbps.

**Advantages:**
- Dramatically smaller files --- a 5-minute call is approximately 600 KB at 32 kbps
- Reduces storage costs by 80-90%
- Faster to transfer, download, and archive

**Disadvantages:**
- Requires LAME encoder installation and CPU overhead during recording
- Lossy compression --- some audio quality is permanently lost
- The CPU cost of real-time MP3 encoding is non-trivial: at 50 concurrent recordings, expect 5-10% additional CPU utilization on the telephony server
- Some compliance frameworks reject lossy-compressed recordings

### The Practical Recommendation

**Record in WAV. Convert to MP3 for archival.** This gives you the best of both worlds: original-quality recordings during the active period when QA, disputes, and compliance reviews happen, and compressed recordings for long-term storage.

Here is the conversion script we use:

```bash
#!/bin/bash
# convert_recordings.sh
# Convert WAV recordings older than 30 days to MP3
# Run via cron: 0 2 * * * /usr/local/bin/convert_recordings.sh

RECORDING_DIR="/var/spool/asterisk/monitor/MIXmonitor"
ARCHIVE_DIR="/var/spool/asterisk/monitor/ARCHIVE"
DAYS_BEFORE_CONVERT=30

mkdir -p ${ARCHIVE_DIR}

find ${RECORDING_DIR} -name "*.wav" -mtime +${DAYS_BEFORE_CONVERT} | while read wavfile; do
    mp3file="${ARCHIVE_DIR}/$(basename ${wavfile%.wav}.mp3)"
    lame --quiet -b 32 --resample 8 "${wavfile}" "${mp3file}"
    if [ $? -eq 0 ] && [ -f "${mp3file}" ]; then
        # Update the recording_log table with new location
        filename=$(basename "${wavfile}")
        mysql -u cron -p'your_cron_password' asterisk -e \
            "UPDATE recording_log SET location='${mp3file}' WHERE filename LIKE '%${filename%.*}%';"
        rm "${wavfile}"
        echo "Converted: ${wavfile} -> ${mp3file}"
    fi
done
```

### Installing LAME for MP3 Support

If you want native MP3 recording (skipping WAV entirely):

```bash
# AlmaLinux 9 / Rocky Linux 9
dnf install epel-release
dnf install lame lame-devel

# ViciBox (openSUSE)
zypper install lame libmp3lame0

# Verify LAME is available
lame --version
```

Then in VICIdial Admin, navigate to **Admin > System Settings > Recording** and set the recording format to MP3. Asterisk will use LAME to encode recordings in real-time.

---

## Storage Capacity Planning

This is where operations get surprised. Call recordings consume storage faster than most people expect, and running out of disk space on a production VICIdial server is a critical failure --- Asterisk will stop recording and may drop active calls.

### The Math

Here are the actual numbers for a 50-agent outbound operation running 8 hours/day:

| Metric | Value |
|--------|-------|
| Agents | 50 |
| Shift duration | 8 hours |
| Average [talk time](/glossary/talk-time/) per hour | 40-48 minutes |
| Average call duration (connected calls) | 2-4 minutes |
| Calls per agent per hour | 12-20 (with [predictive dialing](/glossary/predictive-dialing/)) |
| Percentage of calls recorded | 100% (recommended) |

**WAV storage per day:**
- 50 agents x 8 hours x 45 min avg talk time = 18,000 minutes of recordings/day
- 18,000 minutes x 0.96 MB/min = **17.3 GB/day**

**WAV storage per month (22 working days):**
- 17.3 GB x 22 = **380 GB/month**

**MP3 storage per month (at 32 kbps):**
- 18,000 min x 22 days x 0.12 MB/min = **47.5 GB/month**

**Annual storage:**
- WAV: ~4.5 TB/year
- MP3: ~570 GB/year

These numbers scale linearly with agent count. A 100-agent operation generates double. A 200-agent operation generates quadruple. If you are running multi-shift or 24/7 operations, multiply accordingly.

### Storage Sizing Recommendations

| Agent Count | Daily WAV | Monthly WAV | Recommended Local Disk | Archive Strategy |
|-------------|-----------|-------------|----------------------|-----------------|
| 10 agents | 3.5 GB | 77 GB | 500 GB SSD | Monthly offsite |
| 25 agents | 8.6 GB | 190 GB | 1 TB SSD | Weekly offsite |
| 50 agents | 17.3 GB | 380 GB | 2 TB SSD/RAID | Daily offsite |
| 100 agents | 34.6 GB | 760 GB | 4 TB RAID | Daily offsite + S3 |
| 200+ agents | 69+ GB | 1.5+ TB | Dedicated NAS/SAN | Real-time replication |

**Critical rule:** Never let your recording partition exceed 80% utilization. Set up monitoring alerts at 70% and 80%. When you hit 80%, either archive old recordings or add storage. At 95%, Asterisk recording will start failing silently.

### Disk Space Monitoring

Add this to your crontab for daily disk space alerts:

```bash
#!/bin/bash
# check_recording_disk.sh
THRESHOLD=80
PARTITION="/var/spool/asterisk/monitor"
USAGE=$(df -h ${PARTITION} | awk 'NR==2 {print $5}' | tr -d '%')

if [ ${USAGE} -gt ${THRESHOLD} ]; then
    echo "WARNING: Recording partition at ${USAGE}% usage on $(hostname)" | \
    mail -s "VICIdial Recording Disk Alert" admin@yourcompany.com
fi
```

Schedule it: `0 */4 * * * /usr/local/bin/check_recording_disk.sh`

> **Never Worry About Recording Storage Again.**
> ViciStack deployments include automated archival pipelines, disk monitoring, and compliance-ready retention policies. [Learn More.](/pricing/)

---

## Archival Strategies

Keeping all recordings on your primary VICIdial server's local disk is not sustainable. Here are the archival approaches, from simplest to most robust.

### VICIdial's Built-In ARCHIVE Flag

VICIdial includes a built-in recording archival mechanism. In **Admin > System Settings**, the **Archive** settings control an automated process that moves recordings from the active directory to an archive directory after a configurable number of days.

The system works through the `AST_audio_archive.pl` script that runs via cron. It reads from `recording_log`, moves files to the archive path, and updates the `location` column in the database to reflect the new path. This is the simplest archival option and works well for single-server deployments with attached archive storage.

### rsync to Archive Server

For [multi-server clusters](/blog/vicidial-cluster-guide/) where each telephony server generates recordings locally:

```bash
#!/bin/bash
# sync_recordings.sh - Run on each telephony server
# Syncs recordings to a central archive server
ARCHIVE_SERVER="10.0.2.50"
ARCHIVE_PATH="/recordings/$(hostname)"
LOCAL_PATH="/var/spool/asterisk/monitor/MIXmonitor/"

rsync -avz --remove-source-files \
    --include="*.wav" --include="*.mp3" --exclude="*" \
    ${LOCAL_PATH} ${ARCHIVE_SERVER}:${ARCHIVE_PATH}/
```

Schedule this hourly or daily depending on your storage headroom. The `--remove-source-files` flag frees up local disk after successful transfer. Make sure SSH key authentication is configured between servers.

### S3/MinIO for Long-Term Storage

For operations that need years of recording retention (common in financial services, healthcare, and insurance), object storage is the most cost-effective solution.

**AWS S3:**

```bash
#!/bin/bash
# s3_archive.sh - Archive recordings older than 90 days to S3
RECORDING_DIR="/var/spool/asterisk/monitor/MIXmonitor"
S3_BUCKET="s3://company-vicidial-recordings"
DAYS_OLD=90

find ${RECORDING_DIR} -name "*.wav" -o -name "*.mp3" -mtime +${DAYS_OLD} | while read file; do
    # Organize by year/month in S3
    filedate=$(stat -c %Y "${file}")
    year=$(date -d @${filedate} +%Y)
    month=$(date -d @${filedate} +%m)

    aws s3 cp "${file}" "${S3_BUCKET}/${year}/${month}/$(basename ${file})" \
        --storage-class STANDARD_IA

    if [ $? -eq 0 ]; then
        # Update database with S3 URL
        filename=$(basename "${file}")
        s3_url="${S3_BUCKET}/${year}/${month}/${filename}"
        mysql -u cron -p'password' asterisk -e \
            "UPDATE recording_log SET location='${s3_url}' WHERE filename LIKE '%${filename%.*}%';"
        rm "${file}"
    fi
done
```

**S3 Storage Costs (per month, for 1 TB of recordings):**

| Storage Class | Cost/TB/Month | Use Case |
|--------------|--------------|----------|
| S3 Standard | $23.00 | Active recordings (<90 days) |
| S3 Standard-IA | $12.50 | Infrequently accessed (90-365 days) |
| S3 Glacier Instant Retrieval | $4.00 | Archived (1-3 years) |
| S3 Glacier Deep Archive | $0.99 | Long-term retention (3+ years) |

**MinIO for on-premises S3-compatible storage:**

If you do not want recordings leaving your infrastructure (a valid concern for compliance-sensitive operations), MinIO provides S3-compatible object storage that runs on your own hardware. Deploy it on a separate server with large, cheap storage (spinning drives are fine for archived recordings):

```bash
# Install MinIO on an archive server
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
./minio server /data/recordings --console-address ":9001"
```

Then use the same `aws s3` commands with the `--endpoint-url` flag pointing to your MinIO instance.

### NFS Mounts for Cluster Recording Access

In a cluster, the web server needs to access recordings from all telephony servers for playback. The standard approach is NFS:

**On each telephony server (NFS export):**
```bash
# /etc/exports
/var/spool/asterisk/monitor 10.0.2.0/24(ro,sync,no_subtree_check)
```

**On the web server (NFS mount):**
```bash
# /etc/fstab
10.0.2.10:/var/spool/asterisk/monitor /var/spool/asterisk/monitor/server1 nfs ro,soft,timeo=5 0 0
10.0.2.11:/var/spool/asterisk/monitor /var/spool/asterisk/monitor/server2 nfs ro,soft,timeo=5 0 0
```

Use `soft` and `timeo=5` mount options so that a telephony server going down does not cause the web server to hang. Read-only mounts (`ro`) on the web server prevent accidental deletion from the admin interface.

---

## Compliance Requirements by Jurisdiction and Industry

Recording calls is not just an operational decision --- it is a legal one. The rules about when you can record, how you must store recordings, and how long you must keep them vary dramatically by jurisdiction and industry.

### Federal vs. State Consent Laws

**Federal law (one-party consent):** Under the federal Electronic Communications Privacy Act (ECPA, 18 U.S.C. 2511), you only need one party's consent to record a phone call. Since your agent is a party to the call and your company has authorized recording, federal law is satisfied in most outbound dialing scenarios.

**But state laws can be stricter.** Twelve states plus the District of Columbia require **all-party consent** (also called "two-party consent," though it applies to all parties on the call):

| All-Party Consent States |
|--------------------------|
| California, Connecticut, Delaware, Florida, Illinois, Maryland, Massachusetts, Michigan, Montana, New Hampshire, Pennsylvania, Washington |

**What this means for VICIdial operations:** If you are dialing into all-party consent states, you must inform the called party that the call is being recorded and obtain their consent before the recording begins. The standard approach is a pre-recorded announcement:

*"This call may be monitored or recorded for quality assurance purposes."*

In VICIdial, you can implement this through:

1. **Agent script with verbal disclosure** --- train agents to state the recording notice at the beginning of every call
2. **[IVR](/glossary/ivr/) announcement** --- use a [call menu](/glossary/call-menu/) to play a recording disclosure before connecting to the agent
3. **Selective recording by state** --- use VICIdial's campaign-level recording controls to disable recording for leads in all-party consent states (not recommended --- you lose QA visibility)

The safest approach is universal disclosure on all calls regardless of the destination state. This covers you even when leads have moved or provided incorrect area codes.

### PCI-DSS Compliance

If your agents take credit card payments over the phone, PCI-DSS Section 3.4 requires that you do not store cardholder data (including recording of card numbers spoken aloud) unless it is encrypted with strong cryptography.

**The practical solution: pause/resume recording.**

VICIdial supports recording pause and resume through the agent interface. When an agent is about to collect payment information, they click the "Pause Recording" button. When the payment portion of the call is complete, they click "Resume Recording." The result is a recording file with the payment conversation redacted.

Configure this in VICIdial Admin:

1. **Admin > Campaigns > [Campaign] > Recording** --- set `Campaign Recording` to `ALLCALLS` or `ALLFORCE`
2. **Admin > Campaigns > [Campaign] > Agent Screen** --- ensure `Recording Buttons` is set to `IN_HEADER` or `IN_SCRIPT`
3. Train agents on the exact workflow: pause before payment, resume after payment

**Important limitation:** Pause/resume relies on agent behavior. If an agent forgets to pause, you have a PCI compliance violation on tape. For high-volume payment processing, consider a dedicated IVR payment system that removes the agent from the call during payment collection. VICIdial can transfer calls to an external payment IVR and then reconnect the agent afterward.

```sql
-- Audit query: Find recordings that may contain payment data
-- Look for calls to payment-focused campaigns that were NOT paused
SELECT rl.recording_id, rl.filename, rl.start_time, rl.length_in_sec,
       rl.user, vl.phone_number
FROM recording_log rl
JOIN vicidial_list vl ON rl.lead_id = vl.lead_id
WHERE rl.start_time >= '2026-03-01'
  AND rl.vicidial_id IN (
    SELECT uniqueid FROM vicidial_log
    WHERE campaign_id = 'PAYMENT_CAMPAIGN'
  )
ORDER BY rl.start_time DESC;
```

### HIPAA Compliance

Healthcare-related call recordings fall under [HIPAA](/glossary/call-recording-compliance/) if they contain Protected Health Information (PHI). HIPAA requires:

- **Access controls** --- only authorized personnel can listen to recordings containing PHI
- **Audit trails** --- you must log who accessed which recordings and when (the `recording_access` table handles this)
- **Encryption at rest** --- recordings stored on disk must be encrypted
- **Encryption in transit** --- recordings transferred between servers or accessed via web must use TLS/HTTPS
- **Retention and destruction** --- HIPAA does not specify an exact retention period, but requires a policy and consistent destruction when the period expires

**Encryption at rest for VICIdial recordings:**

The simplest approach is filesystem-level encryption using LUKS on the recording partition:

```bash
# Create encrypted partition for recordings
cryptsetup luksFormat /dev/sdb1
cryptsetup luksOpen /dev/sdb1 recordings_crypt
mkfs.ext4 /dev/mapper/recordings_crypt
mount /dev/mapper/recordings_crypt /var/spool/asterisk/monitor

# Add to /etc/crypttab for automatic mount at boot
echo "recordings_crypt /dev/sdb1 /root/recordings_key luks" >> /etc/crypttab
```

This encrypts all recordings at the filesystem level with no changes needed to VICIdial or Asterisk.

### State Retention Requirements

There is no single federal mandate for how long you must retain call recordings, but state laws and industry regulations create effective minimums:

| Industry/Regulation | Minimum Retention |
|--------------------|-------------------|
| General business (no specific regulation) | 90 days recommended |
| TCPA-related litigation protection | 4-5 years (statute of limitations) |
| Financial services (SEC Rule 17a-4) | 3 years minimum, some records 6 years |
| Insurance (varies by state) | 3-7 years depending on state |
| Healthcare (HIPAA) | 6 years from creation or last effective date |
| Debt collection (FDCPA) | 3-5 years recommended |
| Government contracts | Per contract terms, often 5-7 years |

**Practical advice:** Keep recordings for at least the statute of limitations on any claim that could involve the recorded call. For most outbound dialing operations, that means 4-5 years for [TCPA](/glossary/tcpa/) protection and 3-7 years for industry-specific requirements. The cost of storing compressed recordings for 5 years in S3 Glacier is trivial compared to the cost of being unable to produce a recording when litigation demands it.

---

## Recording Playback and Search in VICIdial Admin

VICIdial provides several built-in tools for finding and listening to recordings.

### Admin Recording Search

Navigate to **Reports > Recording Lookup** in the VICIdial admin interface. You can search by:

- Date range
- Phone number
- Agent user ID
- Lead ID
- Campaign ID

The search queries the `recording_log` table and returns a list of matching recordings with playback links. Each playback link opens an HTML5 audio player in the browser (or triggers a download, depending on your Apache configuration).

### Agent-Side Recording Access

Agents can access recordings for their own calls through the agent interface if you enable it. In **Admin > User Groups**, the `recording_access` permission controls whether agents in that group can listen to recordings. Options:

- **ALL** --- agent can listen to all recordings (not recommended for most operations)
- **OWN** --- agent can only listen to their own recordings
- **NONE** --- agent cannot access recordings

For QA-focused operations, setting this to `OWN` lets agents review their own calls for self-improvement without exposing other agents' recordings.

### API-Based Recording Retrieval

VICIdial's non-agent API includes a recording retrieval endpoint. This is useful for integrating recordings with external CRM systems, QA platforms, or compliance tools:

```
https://your-vicidial-server/vicidial/non_agent_api.php?
  source=recording_api&
  user=API_USER&
  pass=API_PASS&
  function=recording_lookup&
  lead_id=12345&
  date=2026-03-18
```

The API returns the recording URL(s) and metadata, which your external system can then fetch and process.

---

## Recording Quality Assurance Workflows

Recording calls is only useful if someone listens to them. Here are the QA workflows that production VICIdial operations typically implement.

### Random Sampling

The simplest approach: QA managers listen to a random sample of calls each day. A 5% sample rate is common for operations with 50+ agents. For a 50-agent operation generating 4,000-6,000 calls/day, that means 200-300 calls reviewed.

```sql
-- Pull random sample of today's recordings for QA review
SELECT rl.recording_id, rl.filename, rl.user, rl.length_in_sec,
       vl.phone_number, vl.first_name, vl.last_name
FROM recording_log rl
JOIN vicidial_list vl ON rl.lead_id = vl.lead_id
WHERE DATE(rl.start_time) = CURDATE()
  AND rl.length_in_sec > 60  -- Skip short calls
ORDER BY RAND()
LIMIT 100;
```

### Targeted Review

Rather than random sampling, focus QA time on the calls most likely to have issues:

- **Long calls** (>10 minutes) --- possible agent stalling or off-script behavior
- **Short calls** (<30 seconds) --- possible hang-ups, wrong numbers, or agent rudeness
- **Calls resulting in specific dispositions** --- review all "SALE" dispositions to verify the sale was legitimate
- **Calls from new agents** --- higher review rate during the first 2 weeks
- **Complaint-flagged leads** --- any lead that called back to complain

### Integration with External QA Platforms

Several third-party QA platforms can ingest VICIdial recordings for AI-powered analysis, automated scoring, and compliance checking. The integration typically works through:

1. A scheduled script that queries `recording_log` for new recordings
2. Upload of recording files to the QA platform via API
3. The QA platform processes recordings (speech-to-text, keyword detection, sentiment analysis)
4. Results are pushed back to VICIdial or displayed in the QA platform's dashboard

This is an area where [AI-powered quality control](/blog/ai-call-center-quality-control/) is rapidly advancing. Automated transcription and keyword flagging can screen 100% of calls and surface the ones that need human review.

> **QA at Scale, Without the Manual Effort.**
> ViciStack integrates recording management with monitoring and compliance workflows. [See How It Works.](/free-audit/)

---

## Advanced Recording Configuration

### Dual-Channel (Stereo) Recording

For operations that need to distinguish between agent audio and caller audio (common in QA scoring and transcription), VICIdial supports dual-channel recording:

In **Admin > Campaigns > [Campaign] > Recording**, set `Dual Channel Recording` to `Y`. This creates stereo WAV files where the left channel is the agent and the right channel is the caller.

**Warning:** Dual-channel recordings are twice the size of mono mixed recordings. Plan your storage accordingly. The trade-off is worth it if you use speech analytics that benefit from channel separation, but it is unnecessary for basic QA listening.

### On-Demand Recording

Not every call needs to be recorded. VICIdial supports several recording modes configured at the campaign level via the [campaign recording](/settings/campaign-recording/) setting:

| Recording Mode | Behavior |
|---------------|----------|
| ALLCALLS | Every call is recorded automatically |
| ALLFORCE | Every call is recorded, agents cannot stop recording |
| ONDEMAND | Recording starts only when the agent clicks the record button |
| NEVER | No recording |

**ALLFORCE** is the recommended setting for compliance-sensitive operations. It removes the agent's ability to selectively stop recording, which prevents the scenario where an agent stops recording before saying something inappropriate.

**ONDEMAND** is useful for high-volume operations where only a fraction of calls need recording (e.g., recording only sales calls, not survey calls). It reduces storage requirements but creates compliance gaps.

### Recording Permissions by User Group

Control which [user groups](/glossary/user-group/) can access recordings through **Admin > User Groups > [Group] > Allowed Recordings**. This is essential for multi-tenant or multi-client operations where different teams should only hear their own recordings.

Permissions include:
- Which campaigns' recordings the group can access
- Whether the group can download recordings (vs. only stream playback)
- Whether the group can delete recordings

For [BPO operations running multi-tenant VICIdial](/blog/vicidial-multi-tenant/), recording isolation is one of the most important permission configurations. A client should never be able to hear recordings from another client's campaigns.

---

## Troubleshooting Common Recording Issues

### Recordings Not Being Created

**Check 1:** Is the campaign recording setting enabled?
```
Admin > Campaigns > [Campaign] > Recording > Campaign Recording = ALLCALLS or ALLFORCE
```

**Check 2:** Is the recording directory writable?
```bash
ls -la /var/spool/asterisk/monitor/MIXmonitor/
# Should be owned by asterisk:asterisk with 755 permissions
```

**Check 3:** Is there disk space?
```bash
df -h /var/spool/asterisk/monitor/
```

**Check 4:** Is the MixMon application loaded in Asterisk?
```bash
asterisk -rx "module show like mix"
# Should show app_mixmonitor.so as running
```

### Recordings Are Silent or One-Sided

This is almost always a [NAT traversal](/glossary/nat-traversal/) issue. MixMon records the audio streams as they arrive at Asterisk. If one side's audio is not reaching Asterisk due to NAT problems, the recording will be silent on that side.

Verify [RTP](/glossary/rtp/) is flowing in both directions:
```bash
tcpdump -i eth0 udp portrange 10000-20000 -c 100
```

You should see UDP packets going both to and from your carrier's IP addresses.

### Recordings Are Truncated

If recordings cut off before the call ends, the most common cause is Asterisk running out of file descriptors. Each active recording holds an open file descriptor. Check and increase the limit:

```bash
# Check current limit
ulimit -n

# Increase in /etc/security/limits.conf
asterisk soft nofile 65536
asterisk hard nofile 65536
```

Restart Asterisk after changing limits.

---

## Frequently Asked Questions

### How much disk space does VICIdial call recording use per agent per month?

For a single agent working an 8-hour shift with [predictive dialing](/glossary/predictive-dialing/), expect approximately 7-8 GB/month in WAV format or 900 MB-1 GB/month in MP3 format. This assumes roughly 40-45 minutes of talk time per hour, which is typical for a well-tuned outbound campaign. Multiply by your agent count for total monthly storage requirements.

### Can I record only specific campaigns or inbound groups?

Yes. Recording is configured per campaign in **Admin > Campaigns > Recording** and per [inbound group](/glossary/inbound-group/) in **Admin > Inbound Groups > Recording**. You can have one campaign recording all calls while another records none. This is useful for saving storage by not recording surveys, IVR interactions, or other non-sensitive call types.

### How do I move recordings between VICIdial servers?

Use rsync to transfer the recording files, then update the `recording_log` table to reflect the new file locations. The `location` column in `recording_log` is the path VICIdial uses for playback. After moving files, run an UPDATE query to change the path prefix to the new server's path.

### Is it legal to record calls without telling the other party?

Under federal law (one-party consent), yes --- as long as one party to the call (your agent or your company) consents. However, twelve states require all-party consent. If you are dialing into those states, you must disclose that the call is being recorded. The safest practice is to disclose on every call regardless of the destination state.

### How do I automatically delete old recordings?

Create a cron job that deletes recordings older than your retention policy:

```bash
# Delete WAV recordings older than 365 days
find /var/spool/asterisk/monitor/MIXmonitor/ -name "*.wav" -mtime +365 -delete

# Delete corresponding database entries
mysql -u cron -p'password' asterisk -e \
    "DELETE FROM recording_log WHERE start_time < DATE_SUB(NOW(), INTERVAL 365 DAY);"
```

Always verify your retention policy with legal counsel before implementing automated deletion.

### Can VICIdial recordings be used as evidence in court?

Yes, VICIdial recordings are regularly used as evidence in [TCPA](/glossary/tcpa/) litigation, consumer disputes, and regulatory proceedings. To maximize evidentiary value: maintain chain of custody documentation, use tamper-evident storage (write-once S3 Glacier with Object Lock), retain recordings in their original WAV format, and ensure your `recording_log` database entries provide complete metadata (timestamps, agent ID, phone number, lead ID).

### How do I encrypt recordings at rest?

The simplest approach is filesystem-level encryption using LUKS (Linux Unified Key Setup) on the partition where recordings are stored. This is transparent to VICIdial and Asterisk --- they read and write files normally, and the encryption/decryption happens at the filesystem level. For S3-based archival, enable server-side encryption (SSE-S3 or SSE-KMS) on the S3 bucket.

### Does call recording affect VICIdial performance?

On modern hardware, the performance impact of WAV recording is negligible --- Asterisk is writing raw PCM data that already exists in memory. MP3 recording adds measurable CPU overhead due to real-time encoding. At 50 concurrent recordings with MP3 encoding, expect 5-10% additional CPU utilization on the telephony server. If you are running near CPU capacity, stick with WAV and convert to MP3 offline.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-call-recording).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
