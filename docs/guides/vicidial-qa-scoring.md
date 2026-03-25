# VICIdial Quality Assurance Scoring with Call Recordings

Quality assurance is where good call centers separate from great ones. You can optimize your dialer settings, tune your lead lists, and hire the best agents, but without a systematic QA process that scores calls, identifies coaching opportunities, and tracks improvement over time, you are flying blind. VICIdial includes robust call recording capabilities out of the box, but turning those recordings into an actionable QA workflow requires deliberate configuration and process design.

This guide covers the full QA pipeline: setting up recording in VICIdial, designing scorecards that actually improve performance, efficiently searching and reviewing recordings, automating quality flagging, building a coaching process around scores, and meeting compliance recording requirements.

## Setting Up Call Recording in VICIdial

VICIdial supports multiple recording methods, each with different trade-offs for quality, storage, and resource usage.

### Recording Methods

VICIdial offers three recording approaches, configured at the campaign level:

**1. Asterisk Native Recording (MONITOR)**

```
Campaign > Recording: ALLCALLS or ALLFORCE
Recording Method: MONITOR
```

The MONITOR method uses Asterisk's `Monitor()` application, which records the call into two separate audio channels (agent and caller) and then mixes them into a single file. This gives you the ability to analyze each side independently, which is valuable for QA scoring.

**2. MixMon Recording**

```
Campaign > Recording: ALLCALLS or ALLFORCE
Recording Method: MIXMON
```

MixMon (`MixMonitor()`) records both channels into a single file in real time. It uses less CPU than MONITOR because it skips the post-call mixing step. The trade-off is that you lose the ability to isolate agent vs. caller audio.

**3. Server-Side Trunk Recording**

For environments where agent-side recording is not feasible (remote agents with unreliable connections), you can record at the trunk level using Asterisk's built-in recording on the inbound/outbound trunk channels.

### Recommended Configuration

For a QA-focused deployment, use these settings:

```
Campaign Settings:
  Recording: ALLFORCE          # Records every call, agents cannot disable
  Recording Method: MIXMON     # Lower CPU, single file
  Recording Format: WAV        # Uncompressed for quality; convert to MP3 for storage later
```

In VICIdial Admin, navigate to **Campaigns > [Your Campaign] > Detail** and set:

| Setting | Value | Why |
|---------|-------|-----|
| Campaign Recording | ALLFORCE | Ensures 100% recording -- agents cannot toggle it off |
| Campaign Rec Exten | sip-vicidial | Default recording extension |
| Recording Filename | FULLDATE_AGENT_CUSTPHONE | Makes searching recordings easier |

### Storage Planning

Call recordings consume significant storage. Plan for it:

| Codec | Per Minute | Per Hour | 25 Agents x 6hr Talk/Day |
|-------|-----------|----------|--------------------------|
| WAV (16-bit, 8kHz) | 960 KB | 57 MB | 8.5 GB/day |
| MP3 (32 kbps) | 240 KB | 14 MB | 2.1 GB/day |
| GSM | 200 KB | 12 MB | 1.8 GB/day |

For long-term storage, convert recordings from WAV to MP3 after they have been reviewed:

```bash
#!/bin/bash
# /opt/vicistack/compress_recordings.sh
# Convert WAV recordings older than 7 days to MP3

RECORDING_DIR="/var/spool/asterisk/monitor"
find ${RECORDING_DIR} -name "*.wav" -mtime +7 -exec sh -c '
  lame --quiet -b 32 "$1" "${1%.wav}.mp3" && rm "$1"
' _ {} \;
```

Run this nightly via cron:

```
0 3 * * * /opt/vicistack/compress_recordings.sh >> /var/log/recording-compress.log 2>&1
```

### Recording File Location and Database Mapping

Recordings are stored in `/var/spool/asterisk/monitor/` (or a custom path you configure) and referenced in the `recording_log` table:

```sql
SELECT recording_id, channel, server_ip, filename, location,
       start_time, end_time, length_in_sec, lead_id, user
FROM recording_log
WHERE start_time > '2026-03-18'
ORDER BY start_time DESC
LIMIT 20;
```

The `location` field contains the full URL or file path where the recording can be accessed. If you run multiple servers, ensure recordings are accessible from a central location (NFS mount, S3 sync, or a centralized web server).

## QA Scorecard Design for Outbound Sales

A QA scorecard turns subjective "that was a good call" observations into measurable, repeatable scores. The key is designing a scorecard that captures the behaviors that actually drive conversions, not a 50-point checklist that takes 20 minutes to complete.

### Core Scorecard Structure

For outbound sales campaigns, we recommend a scorecard with 5-7 categories and a 1-5 scale:

| Category | Weight | What to Evaluate |
|----------|--------|------------------|
| **Opening** | 15% | Identified themselves, stated purpose, asked permission to continue |
| **Discovery** | 20% | Asked qualifying questions, identified pain points, listened actively |
| **Presentation** | 20% | Presented solution clearly, connected features to pain points, used customer's language |
| **Objection Handling** | 20% | Addressed objections without being defensive, used approved rebuttals, maintained composure |
| **Closing** | 15% | Asked for the commitment, handled final concerns, set clear next steps |
| **Compliance** | 10% | Followed script requirements, disclosed required information, did not make unauthorized promises |

### Scoring Scale

```
5 = Exceptional  -- Textbook execution, could be used as a training example
4 = Proficient   -- Met all requirements with minor gaps
3 = Acceptable   -- Met minimum requirements but missed opportunities
2 = Needs Work   -- Missed key elements, coaching required
1 = Unacceptable -- Failed to meet requirements, immediate coaching required
```

### Implementing Scorecards in VICIdial

VICIdial does not have a built-in QA scorecard module, but you can implement one using custom fields and the callback/note system, or (more commonly) by building a lightweight external scoring tool that reads from VICIdial's `recording_log` table.

#### Option 1: Custom MySQL Table

Create a dedicated QA scoring table:

```sql
CREATE TABLE vicistack_qa_scores (
  score_id INT AUTO_INCREMENT PRIMARY KEY,
  recording_id VARCHAR(50) NOT NULL,
  lead_id INT NOT NULL,
  agent_user VARCHAR(20) NOT NULL,
  scorer_user VARCHAR(20) NOT NULL,
  score_date DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  campaign_id VARCHAR(20),
  call_date DATETIME,

  -- Category scores (1-5)
  opening_score TINYINT CHECK (opening_score BETWEEN 1 AND 5),
  discovery_score TINYINT CHECK (discovery_score BETWEEN 1 AND 5),
  presentation_score TINYINT CHECK (presentation_score BETWEEN 1 AND 5),
  objection_score TINYINT CHECK (objection_score BETWEEN 1 AND 5),
  closing_score TINYINT CHECK (closing_score BETWEEN 1 AND 5),
  compliance_score TINYINT CHECK (compliance_score BETWEEN 1 AND 5),

  -- Weighted total (calculated on insert)
  weighted_total DECIMAL(5,2),

  -- Notes
  comments TEXT,
  coaching_notes TEXT,
  flagged_for_review TINYINT DEFAULT 0,

  INDEX idx_agent (agent_user, call_date),
  INDEX idx_campaign (campaign_id, call_date),
  INDEX idx_recording (recording_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Calculate the weighted score on insert:

```sql
-- Trigger to calculate weighted total
DELIMITER //
CREATE TRIGGER qa_score_calc BEFORE INSERT ON vicistack_qa_scores
FOR EACH ROW
BEGIN
  SET NEW.weighted_total = (
    (NEW.opening_score * 0.15) +
    (NEW.discovery_score * 0.20) +
    (NEW.presentation_score * 0.20) +
    (NEW.objection_score * 0.20) +
    (NEW.closing_score * 0.15) +
    (NEW.compliance_score * 0.10)
  );
END//
DELIMITER ;
```

#### Option 2: Spreadsheet-Based Scoring

For smaller teams, a Google Sheet or Excel workbook linked to your recording URLs works well as a starting point. Pull recording metadata daily:

```bash
#!/bin/bash
# Export today's recordings for QA review
mysql asterisk -e "
SELECT r.recording_id, r.filename, r.start_time, r.length_in_sec,
       r.user AS agent, r.lead_id, l.phone_number, l.first_name, l.last_name
FROM recording_log r
JOIN vicidial_list l ON r.lead_id = l.lead_id
WHERE r.start_time >= CURDATE()
AND r.length_in_sec > 30
ORDER BY r.start_time
" | tr '\t' ',' > /tmp/qa_review_$(date +%Y%m%d).csv
```

### How Many Calls to Score

For statistical significance with a 25-agent team:

- **Minimum**: 5 calls per agent per week (125 total)
- **Recommended**: 10 calls per agent per week (250 total)
- **New agents**: 5 calls per day for the first 2 weeks

Sample randomly, but weight toward:
- Calls with extreme durations (very short or very long)
- Calls from agents with declining conversion rates
- Calls flagged by automated systems (covered below)

## Recording Search and Playback Workflow

An efficient search workflow is critical. If QA reviewers spend 10 minutes finding each recording, you will review far fewer calls.

### VICIdial's Built-in Recording Search

VICIdial's admin interface provides recording search under **Reports > Recording Lookup**. You can search by:

- Date range
- Agent user ID
- Phone number
- Lead ID
- Campaign
- Call status (SALE, DNC, NI, etc.)

### Building a Faster Search Interface

For QA teams that review 50+ recordings per day, build a custom search page or script:

```sql
-- Find recordings to review: sales calls over 2 minutes from today
SELECT r.recording_id,
       r.user AS agent,
       r.start_time,
       r.length_in_sec,
       r.location AS recording_url,
       l.phone_number,
       l.first_name,
       l.last_name,
       v.status,
       v.term_reason
FROM recording_log r
JOIN vicidial_log v ON r.vicidial_id = v.uniqueid
JOIN vicidial_list l ON r.lead_id = l.lead_id
WHERE r.start_time >= CURDATE()
AND r.length_in_sec > 120
AND v.status = 'SALE'
ORDER BY r.start_time;
```

```sql
-- Find an agent's worst calls (short duration, non-sale status)
-- These often reveal coaching opportunities
SELECT r.recording_id,
       r.start_time,
       r.length_in_sec,
       r.location,
       v.status,
       l.phone_number
FROM recording_log r
JOIN vicidial_log v ON r.vicidial_id = v.uniqueid
JOIN vicidial_list l ON r.lead_id = l.lead_id
WHERE r.user = 'agent123'
AND r.start_time >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
AND r.length_in_sec BETWEEN 15 AND 60
AND v.status NOT IN ('SALE', 'CALLBK')
ORDER BY r.length_in_sec ASC
LIMIT 20;
```

### Playback Tips for QA Reviewers

- **Use playback speed controls**: Review at 1.25x or 1.5x speed. A 5-minute call takes 3.3 minutes at 1.5x, and you can still catch everything.
- **Skip dead air**: If the first 15 seconds are ringing, start at the 20-second mark.
- **Flag timestamps**: When you hear a notable moment (good objection handling, compliance issue, missed opportunity), note the timestamp for the coaching session.
- **Batch by agent**: Review all of one agent's calls in sequence -- patterns emerge faster than reviewing random agents.

## Automated QA Flagging

Manual QA review is essential, but automation can prioritize which recordings need human attention first.

### Duration-Based Flagging

Calls that are unusually short or unusually long often indicate problems:

```sql
-- Flag calls shorter than 30 seconds (probable immediate hangup or wrong number)
-- and longer than 20 minutes (probable compliance risk)
SELECT r.recording_id, r.user, r.start_time, r.length_in_sec,
       CASE
         WHEN r.length_in_sec < 30 THEN 'SHORT_CALL'
         WHEN r.length_in_sec > 1200 THEN 'LONG_CALL'
       END AS flag_reason
FROM recording_log r
WHERE r.start_time >= CURDATE()
AND (r.length_in_sec < 30 OR r.length_in_sec > 1200);
```

### Status-Based Flagging

Flag recordings where the call status raises questions:

```sql
-- Flag SALE calls that were under 2 minutes (possible fraudulent disposition)
SELECT r.recording_id, r.user, r.start_time, r.length_in_sec,
       v.status, l.phone_number
FROM recording_log r
JOIN vicidial_log v ON r.vicidial_id = v.uniqueid
JOIN vicidial_list l ON r.lead_id = l.lead_id
WHERE v.status = 'SALE'
AND r.length_in_sec < 120
AND r.start_time >= DATE_SUB(CURDATE(), INTERVAL 1 DAY);
```

### Silence Detection

High silence percentage on a call often indicates an agent who is disengaged or a customer who has lost interest. While full speech analytics requires specialized tools, you can do basic silence detection with SOX:

```bash
#!/bin/bash
# Detect recordings with high silence ratio
RECORDING_DIR="/var/spool/asterisk/monitor"

for wav in ${RECORDING_DIR}/*$(date +%Y%m%d)*.wav; do
  if [ -f "$wav" ]; then
    TOTAL_DURATION=$(soxi -D "$wav" 2>/dev/null)
    SILENCE_DURATION=$(sox "$wav" -n silence 1 0.5 0.1% \
      reverse silence 1 0.5 0.1% reverse stat 2>&1 | \
      grep "Length" | awk '{print $3}')

    if [ -n "$TOTAL_DURATION" ] && [ -n "$SILENCE_DURATION" ]; then
      SILENCE_PCT=$(echo "scale=0; ($SILENCE_DURATION / $TOTAL_DURATION) * 100" | bc)
      if [ "$SILENCE_PCT" -gt 40 ]; then
        echo "HIGH_SILENCE: $(basename $wav) - ${SILENCE_PCT}% silence"
      fi
    fi
  fi
done
```

### Keyword Spotting (Basic)

For basic keyword detection without investing in a full speech analytics platform, you can use open-source speech-to-text and grep for compliance keywords:

```bash
#!/bin/bash
# Basic keyword spotting using Whisper (or similar STT)
# Requires: whisper CLI installed

RECORDING=$1
KEYWORDS="guarantee|promise|definitely will|100 percent|no risk|free money"

# Transcribe the recording
whisper "$RECORDING" --model small --output_format txt --output_dir /tmp/

# Search transcript for flagged keywords
TRANSCRIPT="/tmp/$(basename ${RECORDING%.wav}).txt"
if grep -iE "$KEYWORDS" "$TRANSCRIPT"; then
  echo "FLAGGED: $(basename $RECORDING) contains compliance-risk keywords"
  grep -inE "$KEYWORDS" "$TRANSCRIPT"
fi
```

This is CPU-intensive, so run it on flagged recordings only (not every call), or use a dedicated processing server.

## Agent Coaching Based on QA Scores

QA scores are only valuable if they drive coaching conversations that improve performance. Here is a process that works.

### Weekly Coaching Reports

Generate a per-agent summary weekly:

```sql
-- Weekly agent QA summary
SELECT agent_user,
       COUNT(*) AS calls_scored,
       ROUND(AVG(weighted_total), 2) AS avg_score,
       ROUND(AVG(opening_score), 1) AS avg_opening,
       ROUND(AVG(discovery_score), 1) AS avg_discovery,
       ROUND(AVG(presentation_score), 1) AS avg_presentation,
       ROUND(AVG(objection_score), 1) AS avg_objection,
       ROUND(AVG(closing_score), 1) AS avg_closing,
       ROUND(AVG(compliance_score), 1) AS avg_compliance,
       MIN(weighted_total) AS worst_score,
       MAX(weighted_total) AS best_score
FROM vicistack_qa_scores
WHERE score_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY agent_user
ORDER BY avg_score ASC;
```

### The Coaching Conversation Structure

1. **Start positive**: Play the agent's highest-scored call from the week. Highlight what they did well.
2. **Identify one focus area**: Do not overwhelm them with 10 things to fix. Pick the category with the lowest average score.
3. **Play the example**: Play a specific recording where that category scored low. Ask the agent what they would do differently.
4. **Role play**: Practice the specific scenario with the corrected approach.
5. **Set a measurable goal**: "Next week, I want to see your discovery score average go from 2.3 to 3.0."

### Tracking Improvement Over Time

```sql
-- 4-week trend by agent
SELECT agent_user,
       YEARWEEK(score_date) AS week,
       ROUND(AVG(weighted_total), 2) AS avg_score,
       COUNT(*) AS calls_scored
FROM vicistack_qa_scores
WHERE score_date >= DATE_SUB(CURDATE(), INTERVAL 28 DAY)
GROUP BY agent_user, YEARWEEK(score_date)
ORDER BY agent_user, week;
```

### Calibration Sessions

QA scores are only meaningful if scorers are consistent. Run monthly calibration sessions:

1. Select 3 recordings (one good, one average, one poor)
2. Have all QA scorers independently score each recording
3. Compare scores. If any category differs by more than 1 point between scorers, discuss and align
4. Document the calibration decisions as scoring precedents

## Compliance Recording Requirements

Depending on your jurisdiction and industry, you may have legal obligations around call recording.

### One-Party vs. Two-Party Consent

In the United States, recording laws vary by state:

**One-Party Consent States** (most states): Only one party needs to know the call is being recorded. Since your agent knows, you are covered without informing the customer.

**Two-Party (All-Party) Consent States**: All parties must be informed. These states include California, Connecticut, Florida, Illinois, Maryland, Massachusetts, Michigan, Montana, New Hampshire, Oregon, Pennsylvania, and Washington.

If you dial into two-party consent states, you **must** play a recording notification or have agents verbally disclose at the start of every call:

```
"This call may be monitored or recorded for quality assurance purposes."
```

### Configuring VICIdial to Play Recording Disclosures

Set up an automated disclosure message in the campaign settings:

```
Campaign > Campaign Recording > Recording Disclosure:
  Recording Disclosure: Y
  Recording Disclosure Message: "recording_disclosure"
```

Upload the audio file (`recording_disclosure.wav`) to `/var/lib/asterisk/sounds/`:

```bash
# Convert your disclosure recording to the correct format
sox recording_disclosure_original.wav -r 8000 -c 1 \
  /var/lib/asterisk/sounds/recording_disclosure.wav
```

### Retention Requirements

Different regulations require different retention periods:

| Regulation | Retention Period | Notes |
|-----------|-----------------|-------|
| TCPA | No specific requirement | But maintain records for 4 years (statute of limitations) |
| PCI-DSS | Do NOT record credit card data | Pause recording during payment collection |
| HIPAA | 6 years minimum | If handling healthcare-related calls |
| State regulations | Varies | Check your state's requirements |

### PCI-DSS Compliance: Pausing Recordings

If agents collect credit card numbers over the phone, you must pause recording during the payment portion. VICIdial supports this with the agent interface's PAUSE RECORDING button, or you can automate it:

```
Campaign > Recording: ALLFORCE
Allow Recording Pause: Y
```

Train agents to:
1. Click "Pause Recording" before asking for the card number
2. Collect payment information
3. Click "Resume Recording" after the payment section

Audit this regularly -- agents forgetting to pause is a PCI violation.

## Building a Complete QA Workflow

Here is the end-to-end workflow for a 25-agent center:

### Daily

1. **Automated flagging** runs overnight, generating a priority review list
2. **QA reviewer** scores 50 calls (2 per agent) from the flagged + random pool
3. **Scores entered** into the QA scoring system
4. **Critical flags** (compliance issues, fraudulent dispositions) escalated to manager immediately

### Weekly

5. **Agent reports** generated showing 7-day averages by category
6. **Coaching sessions** conducted for agents below threshold (weighted score < 3.0)
7. **Top performers** recognized (weighted score > 4.5)

### Monthly

8. **Calibration session** with all QA scorers
9. **Scorecard review** -- adjust categories or weights based on what is driving conversions
10. **Compliance audit** -- verify all two-party consent disclosures are playing correctly
11. **Storage management** -- archive old recordings, verify retention compliance

## How ViciStack Helps

Building a QA workflow from scratch on VICIdial requires database customization, recording infrastructure management, storage planning, and ongoing process optimization. Most 25+ agent centers do not have a dedicated VICIdial DBA who can build the custom scoring tables, automated flagging queries, and reporting dashboards described in this guide.

ViciStack handles the QA infrastructure for VICIdial call centers:

- **Recording configuration** optimized for storage efficiency and audio quality
- **Custom QA scoring tables** integrated with your VICIdial database
- **Automated flagging** for short calls, disposition anomalies, and silence detection
- **Agent performance dashboards** showing QA trends alongside dial metrics
- **Storage management** with automated compression, archiving, and retention compliance
- **Compliance configuration** including recording disclosure setup and PCI pause automation

We have seen QA programs increase agent conversion rates by 15-30% within 60 days when combined with proper coaching. The recordings are already there -- ViciStack helps you turn them into revenue.

**Get a free analysis of your VICIdial recording and QA setup.** We will review your recording configuration, storage usage, and show you exactly where coaching opportunities are hiding in your data.

[Request your free ViciStack analysis](https://vicistack.com/proof/) -- response in under 5 minutes.

## Related Articles

- [VICIdial Database Partitioning for High-Volume Call Centers](/blog/vicidial-database-partitioning) -- manage recording_log table growth
- [VICIdial Remote Agent Setup](/blog/vicidial-remote-agent-setup) -- ensure recording works for remote agents
- [VICIdial Timezone-Aware Dialing and TCPA Compliance](/blog/vicidial-timezone-dialing-tcpa) -- compliance extends beyond recordings

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-qa-scoring).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
