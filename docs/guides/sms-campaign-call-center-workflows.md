# SMS Campaign Integration for Call Centers: 10DLC Registration, Dialer Workflows, and Cadences That Convert

**Last updated: March 2026 | Reading time: ~26 minutes**

Your agents dial 400 numbers a day and talk to maybe 60 of the people attached to them. The other 340 calls go to voicemail, get screened, or ring out. Meanwhile, 90% of text messages get read within 3 minutes of delivery.

That gap -- between phone calls that don't connect and text messages that get read almost instantly -- is where most call centers are leaving money on the table. The existing guide on [SMS campaigns for call centers](/blog/sms-campaign-call-center/) covers the compliance fundamentals and cadence theory. This guide goes into the technical integration: how to wire SMS into your VICIdial dialer workflow so texts fire automatically based on call outcomes, and how to build multi-channel cadences that actually improve your [contact rate](/blog/contact-rate-optimization-guide/).

But first, the compliance piece, because if you get that wrong, nothing else matters.

## 10DLC Registration: The Non-Optional First Step

Since February 1, 2025, US carriers block 100% of unregistered Application-to-Person (A2P) traffic sent over 10-digit long code (10DLC) numbers. There are no exceptions, no grace period, and no workaround. If your messages are not registered, they are not delivered.

### What 10DLC Is

10DLC is a system that lets businesses send text messages using standard 10-digit phone numbers (the same numbers you use for calling) instead of short codes (5-6 digit numbers). The registration happens in two parts through The Campaign Registry (TCR):

**Brand Registration** -- register your business entity. TCR verifies your EIN, legal name, and contact information. This is a one-time step per business.

**Campaign Registration** -- register each SMS use case. A sales follow-up campaign is separate from an appointment reminder campaign. Each campaign gets its own throughput limits and content rules.

### Registration Process and Timeline

| Step | Timeline | Cost |
|---|:---:|:---:|
| Brand registration | 1-5 business days | $4 one-time (standard) |
| Brand vetting (higher throughput) | 3-10 business days | $40 one-time |
| Campaign registration | 1-7 business days | $10/month per campaign |
| Total from start to sending | 1-3 weeks | ~$54 to start |

The timeline varies by provider. Twilio, Telnyx, and Bandwidth all handle TCR registration through their dashboards. If you are already using one of these for SIP trunking, use the same provider for SMS to simplify the setup.

### Throughput Limits by Trust Score

After registration, your brand gets a trust score that determines how many messages you can send:

| Trust Score | Message Segments per Second | Daily Cap |
|:---:|:---:|:---:|
| Low (unvetted) | 0.2 | ~17,000 |
| Medium | 1.0 | ~86,000 |
| High (vetted) | 3.0 | ~260,000 |

For a call center running 50 agents at 400 dials per day, you generate roughly 200-300 SMS messages per day (assuming you text every no-answer and voicemail). Even the lowest trust tier handles that volume with room to spare. But if you plan to run marketing blasts or appointment reminders in addition to disposition-triggered texts, get the brand vetting done upfront.

## TCPA Compliance for Text Messages

The TCPA applies to text messages exactly like it applies to phone calls. The fines are the same: $500 per violation for unintentional, $1,500 per violation for willful. On a list of 10,000 contacts, that is $5 million to $15 million in theoretical exposure.

### Consent Requirements

Since late 2023, the FCC requires **one-to-one consent**. A lead who gave permission to Company A does not automatically consent to messages from Company B, even if both companies bought the lead from the same source.

Your consent record must include:

```
Required consent documentation per subscriber:
- Phone number
- Timestamp of consent (UTC)
- IP address (for web opt-ins)
- Exact consent language shown to the subscriber
- Source (web form URL, paper form scan, verbal recording)
- Campaign(s) consented to
- Expected message frequency disclosed
```

Store this in your CRM or a dedicated consent database. You need it to defend against TCPA claims, and "we had consent but can not prove it" is the same as "we did not have consent" in court.

### Opt-Out Handling

As of April 2025, businesses must honor opt-out requests within 10 business days and accept revocation through any reasonable method. In practice, this means:

- Respond to STOP, UNSUBSCRIBE, CANCEL, END, and QUIT keywords automatically
- Process opt-outs from email requests, phone calls, and [web forms](/blog/vicidial-agent-screen-customization/)
- Remove the number from all SMS campaigns (not just the one they replied to)
- Send a confirmation message after processing the opt-out

Every outbound SMS must include opt-out instructions. The standard footer:

```
Reply STOP to unsubscribe. Msg & data rates may apply.
```

### Timing Restrictions

Same as calling: no messages before 8 AM or after 9 PM in the recipient's local time zone. Your messaging gateway needs timezone awareness, just like your dialer.

## Wiring SMS Into VICIdial Workflows

VICIdial does not have native SMS sending. You need an external messaging gateway connected via API triggers. The integration pattern is straightforward: VICIdial dispositions a call, a script detects the disposition, and fires an SMS through your provider's API.

### Architecture Overview

```
VICIdial Agent dispositions call → vicidial_log updated
          ↓
Polling script reads new dispositions every 30-60 seconds
          ↓
Disposition-to-SMS mapping determines which template to send
          ↓
API call to Telnyx/Twilio/SignalWire sends the message
          ↓
SMS delivery status logged to sms_log table
          ↓
Inbound replies routed back to agent screen or queue
```

### The Disposition Polling Script

This script runs as a cron job, checking for new call dispositions and firing SMS messages based on the outcome:

```python
#!/usr/bin/env python3
"""sms_trigger.py - Send SMS based on VICIdial call dispositions"""

import os
import json
import time
import requests
import mysql.connector
from datetime import datetime, timedelta

# Configuration
DB_CONFIG = {
    "host": "localhost",
    "user": "cron",
    "password": os.environ.get("VICI_DB_PASS", ""),
    "database": "vicidial"
}

# Telnyx API (swap for Twilio/SignalWire as needed)
TELNYX_API_KEY = os.environ.get("TELNYX_API_KEY", "")
TELNYX_FROM_NUMBER = "+15551234567"
TELNYX_MESSAGING_PROFILE = os.environ.get("TELNYX_MSG_PROFILE", "")

# Disposition-to-SMS mapping
DISPOSITION_SMS_MAP = {
    "NA": {
        "template": "Hi {first_name}, we tried reaching you about {campaign_topic}. "
                    "Text back a good time to talk. Reply STOP to opt out.",
        "delay_seconds": 60,
        "max_sends": 2,
        "cooldown_hours": 24
    },
    "AM": {
        "template": "Hi {first_name}, we left you a voicemail about {campaign_topic}. "
                    "Have a quick question? Text us back. Reply STOP to opt out.",
        "delay_seconds": 120,
        "max_sends": 1,
        "cooldown_hours": 48
    },
    "CALLBK": {
        "template": "Hi {first_name}, confirming your callback for {callback_date}. "
                    "Text YES to confirm or suggest a new time. Reply STOP to opt out.",
        "delay_seconds": 30,
        "max_sends": 1,
        "cooldown_hours": 0
    },
    "SALE": {
        "template": "Thanks {first_name}! Your enrollment is confirmed. "
                    "Your rep {agent_name} is your point of contact. "
                    "Reply STOP to opt out.",
        "delay_seconds": 300,
        "max_sends": 1,
        "cooldown_hours": 0
    }
}

def get_new_dispositions(since_minutes=2):
    """Pull recent call dispositions from VICIdial."""
    conn = mysql.connector.connect(**DB_CONFIG)
    cursor = conn.cursor(dictionary=True)
    cursor.execute("""
        SELECT
            v.uniqueid, v.lead_id, v.user AS agent_user,
            v.status AS disposition, v.phone_number,
            v.call_date, v.campaign_id,
            l.first_name, l.last_name
        FROM vicidial_log v
        JOIN vicidial_list l ON v.lead_id = l.lead_id
        WHERE v.call_date >= NOW() - INTERVAL %s MINUTE
            AND v.status IN ('NA', 'AM', 'CALLBK', 'SALE')
            AND v.phone_number NOT IN (
                SELECT phone_number FROM sms_dnc_list
            )
            AND v.phone_number NOT IN (
                SELECT phone_number FROM sms_log
                WHERE sent_at >= NOW() - INTERVAL 24 HOUR
                AND disposition = v.status
            )
        ORDER BY v.call_date DESC
    """, (since_minutes,))
    rows = cursor.fetchall()
    cursor.close()
    conn.close()
    return rows

def send_sms(to_number, message):
    """Send SMS via Telnyx API."""
    resp = requests.post(
        "https://api.telnyx.com/v2/messages",
        headers={
            "Authorization": f"Bearer {TELNYX_API_KEY}",
            "Content-Type": "application/json"
        },
        json={
            "from": TELNYX_FROM_NUMBER,
            "to": f"+1{to_number}",
            "text": message,
            "messaging_profile_id": TELNYX_MESSAGING_PROFILE
        }
    )
    return resp.status_code == 200, resp.json()

def log_sms(lead_id, phone_number, disposition, message, status):
    """Log SMS send to database for tracking and deduplication."""
    conn = mysql.connector.connect(**DB_CONFIG)
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO sms_log
            (lead_id, phone_number, disposition, message, status, sent_at)
        VALUES (%s, %s, %s, %s, %s, NOW())
    """, (lead_id, phone_number, disposition, message, status))
    conn.commit()
    cursor.close()
    conn.close()

def process_dispositions():
    """Main processing loop."""
    dispositions = get_new_dispositions(since_minutes=2)

    for dispo in dispositions:
        sms_config = DISPOSITION_SMS_MAP.get(dispo["disposition"])
        if not sms_config:
            continue

        message = sms_config["template"].format(
            first_name=dispo.get("first_name", "there"),
            campaign_topic="your recent inquiry",
            callback_date="your scheduled time",
            agent_name=dispo.get("agent_user", "your representative")
        )

        # Check timing restriction (8 AM - 9 PM local)
        hour = datetime.now().hour
        if hour < 8 or hour >= 21:
            log_sms(dispo["lead_id"], dispo["phone_number"],
                    dispo["disposition"], message, "deferred_time")
            continue

        success, resp = send_sms(dispo["phone_number"], message)
        log_sms(dispo["lead_id"], dispo["phone_number"],
                dispo["disposition"], message,
                "sent" if success else "failed")

if __name__ == "__main__":
    process_dispositions()
```

### Database Tables for SMS Tracking

Create the tracking tables in your VICIdial database:

```sql
CREATE TABLE IF NOT EXISTS sms_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    lead_id INT NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    disposition VARCHAR(10),
    message TEXT,
    status ENUM('sent','failed','delivered','deferred_time','opted_out') DEFAULT 'sent',
    sent_at DATETIME NOT NULL,
    delivered_at DATETIME,
    response_text TEXT,
    response_at DATETIME,
    INDEX idx_phone_date (phone_number, sent_at),
    INDEX idx_lead (lead_id),
    INDEX idx_status (status, sent_at)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS sms_dnc_list (
    phone_number VARCHAR(20) PRIMARY KEY,
    opted_out_at DATETIME NOT NULL,
    opt_out_keyword VARCHAR(20),
    source VARCHAR(50) DEFAULT 'sms_reply'
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS sms_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    template_name VARCHAR(100) NOT NULL,
    disposition VARCHAR(10),
    campaign_id VARCHAR(20),
    message_text TEXT NOT NULL,
    delay_seconds INT DEFAULT 60,
    max_sends INT DEFAULT 1,
    cooldown_hours INT DEFAULT 24,
    active TINYINT DEFAULT 1,
    created_at DATETIME DEFAULT NOW(),
    UNIQUE KEY idx_dispo_campaign (disposition, campaign_id)
) ENGINE=InnoDB;
```

### Cron Schedule

```bash
# Run disposition-triggered SMS every 2 minutes during operating hours
*/2 8-20 * * 1-6 python3 /opt/sms-integration/sms_trigger.py >> /var/log/sms/trigger.log 2>&1

# Process inbound SMS replies every minute
* 8-21 * * * python3 /opt/sms-integration/sms_inbound.py >> /var/log/sms/inbound.log 2>&1

# Daily SMS report
0 7 * * * python3 /opt/sms-integration/sms_report.py >> /var/log/sms/daily_report.log 2>&1
```

## Multi-Touch Cadence Design

Sending a single text after a missed call is table stakes. The real conversion improvement comes from building multi-touch cadences that alternate voice and text across multiple days.

### The 7-Day High-Intent Cadence

For warm leads (web form submissions, inbound inquiries):

| Day | Time | Channel | Action |
|:---:|:---:|:---:|---|
| Day 1 | +0 min | Call | Immediate call attempt |
| Day 1 | +1 min | SMS | "We just tried calling about your inquiry..." |
| Day 1 | +4 hours | Call | Second attempt, different time slot |
| Day 1 | +4.5 hours | SMS | "Still hoping to connect..." (only if no answer) |
| Day 2 | AM | Call | Morning attempt |
| Day 2 | PM | SMS | Value-add content (not just "call us back") |
| Day 3 | PM | Call | Afternoon attempt |
| Day 4 | AM | SMS | Social proof or [case study](/blog/vicidial-roi-case-study/) link |
| Day 5 | PM | Call | Last high-effort attempt |
| Day 7 | AM | SMS | Final text with direct calendar link |

### The 14-Day Cold Outbound Cadence

For purchased leads or cold lists:

| Day | Channel | Template Focus |
|:---:|:---:|---|
| Day 1 | Call | Initial contact attempt |
| Day 1 | SMS | Brief introduction + value prop |
| Day 3 | Call | Second attempt, different time |
| Day 5 | SMS | Industry stat or pain point |
| Day 7 | Call | Third attempt |
| Day 7 | SMS | Case study or testimonial |
| Day 10 | Call | Fourth attempt |
| Day 12 | SMS | Direct offer or incentive |
| Day 14 | Call + SMS | Final attempt + breakup message |

The breakup message on Day 14 is surprisingly effective. Something like: "Last attempt to reach you about [topic]. If the timing isn't right, no hard feelings. Text LATER if you want us to try again next month."

### Cadence Performance Benchmarks

Track these metrics per cadence:

| Metric | Good | Average | Poor |
|---|:---:|:---:|:---:|
| SMS delivery rate | 97%+ | 93-96% | Below 93% |
| SMS response rate | 8-15% | 4-7% | Below 4% |
| SMS opt-out rate | Below 2% | 2-5% | Above 5% |
| Call + SMS [contact rate](/blog/contact-rate-optimization/) | 35-45% | 25-34% | Below 25% |
| Cadence-to-conversion | 8-12% | 4-7% | Below 4% |

The key metric is **Call + SMS contact rate** compared to call-only contact rate. If you are running 18% contact rate on calls alone and 35% with the SMS cadence layered in, that is a 94% improvement in conversations [per lead](/blog/call-center-cost-per-lead-benchmarks/) -- on the same list.

## Handling Inbound SMS Replies

When a lead texts back, that reply needs to reach an agent fast. The response window for inbound texts is measured in minutes, not hours. A lead who texts "what time works?" at 2 PM and gets a reply at 5 PM has already moved on.

### Routing Replies to Agents

Build an inbound SMS processor that checks for opt-out keywords first, then routes real replies to the agent who handled the original call:

```python
#!/usr/bin/env python3
"""sms_inbound.py - Process inbound SMS replies"""

import os
import mysql.connector
import requests

DB_CONFIG = {
    "host": "localhost",
    "user": "cron",
    "password": os.environ.get("VICI_DB_PASS", ""),
    "database": "vicidial"
}

OPT_OUT_KEYWORDS = {"stop", "unsubscribe", "cancel", "end", "quit"}

def process_inbound_messages():
    """Fetch new inbound messages from provider and process them."""
    # Pull unprocessed inbound messages from webhook table
    conn = mysql.connector.connect(**DB_CONFIG)
    cursor = conn.cursor(dictionary=True)
    cursor.execute("""
        SELECT id, from_number, message_text, received_at
        FROM sms_inbound_queue WHERE processed = 0
        ORDER BY received_at ASC
    """)
    messages = cursor.fetchall()

    for msg in messages:
        phone = msg["from_number"].replace("+1", "").strip()
        text_lower = msg["message_text"].strip().lower()

        # Check for opt-out
        if text_lower in OPT_OUT_KEYWORDS:
            cursor.execute("""
                INSERT IGNORE INTO sms_dnc_list
                (phone_number, opted_out_at, opt_out_keyword) VALUES (%s, NOW(), %s)
            """, (phone, text_lower))
            # Send confirmation
            send_sms(phone, "You have been unsubscribed. No further messages will be sent.")
        else:
            # Find the original agent and create a callback
            cursor.execute("""
                SELECT v.user, v.lead_id, v.campaign_id
                FROM vicidial_log v
                JOIN sms_log s ON v.lead_id = s.lead_id
                WHERE s.phone_number = %s
                ORDER BY v.call_date DESC LIMIT 1
            """, (phone,))
            original = cursor.fetchone()

            if original:
                # Insert callback for the original agent
                cursor.execute("""
                    INSERT INTO vicidial_callbacks
                    (lead_id, list_id, campaign_id, status, user,
                     recipient, callback_time, comments)
                    SELECT lead_id, list_id, %s, 'LIVE', %s,
                           'USERONLY', NOW(), %s
                    FROM vicidial_list WHERE lead_id = %s
                """, (original["campaign_id"], original["user"],
                      f"SMS reply: {msg['message_text'][:200]}", original["lead_id"]))

        cursor.execute("UPDATE sms_inbound_queue SET processed = 1 WHERE id = %s", (msg["id"],))

    conn.commit()
    cursor.close()
    conn.close()
```

This creates a VICIdial callback assigned to the original agent when a lead replies by text. The agent sees the callback in their queue with the SMS reply in the comments field, giving them context before they dial back.

### SMS Reply Timing

Track your reply-to-callback time. The lead texted you -- they are engaged right now. Every minute of delay reduces the probability of a conversion:

| Reply Time | Conversion Impact |
|:---:|---|
| Under 5 min | Peak engagement, highest close rate |
| 5-15 min | Good, slight drop-off |
| 15-60 min | Noticeable decline in engagement |
| 1-4 hours | Lead has moved on, reconnection harder |
| 4+ hours | Basically a cold re-contact |

If your agents are too busy to handle callbacks quickly, create a dedicated "SMS response" campaign in VICIdial with higher priority routing.

## Measuring SMS ROI

Track SMS performance separately from voice to understand the incremental value:

```sql
SELECT
    t.disposition AS trigger_dispo,
    COUNT(DISTINCT s.lead_id) AS leads_texted,
    SUM(CASE WHEN s.status = 'delivered' THEN 1 ELSE 0 END) AS delivered,
    SUM(CASE WHEN s.response_text IS NOT NULL THEN 1 ELSE 0 END) AS responses,
    ROUND(SUM(CASE WHEN s.response_text IS NOT NULL THEN 1 ELSE 0 END) /
        GREATEST(COUNT(*), 1) * 100, 1) AS response_rate_pct,
    SUM(CASE WHEN v2.status IN ('SALE','XFER') THEN 1 ELSE 0 END) AS conversions_after_sms,
    ROUND(SUM(CASE WHEN v2.status IN ('SALE','XFER') THEN 1 ELSE 0 END) /
        GREATEST(COUNT(DISTINCT s.lead_id), 1) * 100, 1) AS sms_assisted_conv_pct
FROM sms_log s
LEFT JOIN vicidial_log v2 ON s.lead_id = v2.lead_id
    AND v2.call_date > s.sent_at
    AND v2.call_date < s.sent_at + INTERVAL 7 DAY
    AND v2.status IN ('SALE','XFER')
JOIN sms_templates t ON s.disposition = t.disposition
WHERE s.sent_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY t.disposition
ORDER BY sms_assisted_conv_pct DESC;
```

This query shows which disposition triggers produce the most SMS-assisted conversions. If your "NA" (no answer) texts lead to 6% conversions within 7 days but your "AM" (answering machine) texts produce only 1%, allocate more of your messaging budget to the NA workflow and rethink the AM template.

## What to Build First

If you are starting from zero SMS integration, here is the priority order:

1. **10DLC registration** -- start today, it takes 1-3 weeks. Do not wait.
2. **Consent audit** -- verify you have documented one-to-one consent for every contact you plan to text. If you don't, you can not text them.
3. **Single disposition trigger** -- start with "NA" (no answer) only. Send one text within 60 seconds of a missed call. This is the highest-ROI single addition.
4. **Opt-out handling** -- make sure STOP works before you send a single message.
5. **Multi-touch cadence** -- once the basic trigger works, expand to the 7-day cadence.
6. **Reporting and optimization** -- measure delivery, response, and conversion rates. A/B test templates.

The teams at [ViciStack](https://vicistack.com/) wire SMS into dialer workflows as part of every [contact rate optimization](/blog/contact-rate-optimization-guide/) engagement because the combination of voice and text consistently outperforms either channel alone by 40-80%. If you want disposition-triggered SMS running on your VICIdial instance without spending months on the integration, [we build that](https://vicistack.com/contact/).

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/sms-campaign-call-center-workflows).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
