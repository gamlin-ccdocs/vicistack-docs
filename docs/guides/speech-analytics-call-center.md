# Speech Analytics for Call Centers: From Call Recordings to Automated QA Without a Six-Figure Platform

**Last updated: March 2026 | Reading time: ~26 minutes**

Here is the dirty secret of call center QA: most operations review 1-2% of their calls. A QA analyst listens to maybe 5-10 recordings per agent per month, fills out a scorecard, and hopes that sample is representative.

It is not. Two percent coverage means 98% of your calls -- including compliance violations, missed upsells, and the call where your best agent snapped at a customer -- go completely unreviewed.

Speech analytics changes that equation. Automated transcription and analysis can process 100% of your calls, flag the ones that matter, and hand your QA team a prioritized list instead of a random sample. McKinsey data shows contact centers using speech analytics see a 10% improvement in customer satisfaction scores. Sprinklr reports 20-30% cost savings and a 40% productivity boost when speech analytics is implemented properly.

The problem is price. Enterprise speech analytics platforms from CallMiner, Verint, and NICE run $50K-$200K per year for a mid-sized operation. That prices out most call centers under 100 agents.

But the underlying technology -- transcription via Whisper, sentiment analysis via open-source NLP models, keyword detection via pattern matching -- is available for the cost of a decent GPU server and some engineering time. If you already run [VICIdial call recording](/blog/vicidial-call-recording/), you are sitting on a goldmine of unanalyzed audio data.

This guide covers how to build a working speech analytics pipeline: from call recordings to transcripts to automated [QA scoring](/blog/vicidial-qa-scoring/), using tools you can actually afford.

## What Speech Analytics Actually Does

Strip away the vendor marketing and speech analytics breaks down into four concrete capabilities:

### 1. Transcription (Speech-to-Text)

Convert audio recordings into searchable text. This is the foundation. Without accurate transcripts, nothing else works.

Modern transcription accuracy with Whisper-class models runs 92-97% word error rate on clean call center audio. That is good enough for keyword detection and sentiment analysis, though you will still want human review on compliance-critical calls.

### 2. Keyword and Phrase Detection

Search transcripts for specific words and phrases. The two main use cases:

**Compliance monitoring** -- detect when agents skip required disclosures, make unauthorized promises, or use prohibited language.

**Sales intelligence** -- identify competitor mentions, objection patterns, buying signals, and pricing discussions.

### 3. Sentiment Analysis

Score the emotional tone of the conversation. Most implementations track:

- **Agent sentiment** -- are they frustrated, bored, engaged, professional?
- **Customer sentiment** -- are they angry, confused, interested, ready to buy?
- **Sentiment trajectory** -- did the call start negative and end positive (good recovery) or start positive and end negative (lost sale)?

### 4. Automated QA Scoring

Combine transcription, keywords, and sentiment into an automated quality score for each call. Instead of manually scoring 5 calls per agent per month, the system scores every call and surfaces the outliers for human review.

Opus Research found that 68% of companies using speech analytics saw it as a cost-saving tool, and 52% saw direct revenue improvement. The ROI comes from doing more with less -- not replacing QA analysts, but focusing their time on the 5% of calls that actually need human attention.

## Setting Up the Recording Pipeline

Speech analytics starts with audio files. If your recordings are bad, your transcripts will be bad, and your analytics will be useless.

### VICIdial Recording Configuration

Set recording at the campaign level for full coverage:

```
Campaign > Detail > Recording: ALLFORCE
Campaign > Detail > Recording Method: STEREO
Campaign > Detail > Recording Filename: FULLDATE_AGENT_CUSTPHONE
```

The key settings:

- **ALLFORCE** records every call regardless of agent action. ALLCALLS lets agents control recording (they will forget or skip it).
- **STEREO** records agent and customer on separate channels. This is critical for speech analytics because it lets you run sentiment analysis on each speaker independently. MONO mixes both channels, making speaker separation impossible.
- **Filename format** with date, agent, and customer phone makes it easy to correlate recordings with CDR data later.

### Recording Storage and Format

VICIdial stores recordings in `/var/spool/asterisk/monitor/` by default, organized by date. The default format is WAV (uncompressed).

For speech analytics processing, you want WAV files -- not MP3. Transcription models work better with uncompressed audio. If storage is a concern, compress after transcription:

```bash
# Check recording storage usage
du -sh /var/spool/asterisk/monitor/
du -sh /var/spool/asterisk/monitor/$(date +%Y/%m/%d)/

# Count recordings per day
find /var/spool/asterisk/monitor/$(date +%Y/%m/%d)/ -name "*.wav" | wc -l
```

A typical 50-agent operation generates 3-5 GB of WAV recordings per day. That is about 1.5 TB per year. Budget your storage accordingly.

### Centralizing Recordings for Processing

If you are running a multi-server VICIdial cluster, recordings live on whichever server handled the call. You need to centralize them before processing.

Set up an rsync job to pull recordings to your analytics server:

```bash
#!/bin/bash
# sync-recordings.sh - Pull recordings from VICIdial servers to analytics box
ANALYTICS_DIR="/data/recordings"
VICIDIAL_SERVERS=("vici-web1" "vici-tel1" "vici-tel2")
TODAY=$(date +%Y/%m/%d)

for server in "${VICIDIAL_SERVERS[@]}"; do
    rsync -avz --include="*.wav" --exclude="*" \
        "${server}:/var/spool/asterisk/monitor/${TODAY}/" \
        "${ANALYTICS_DIR}/${TODAY}/"
done
```

Run this via cron every hour during operating hours. Set it up on the analytics server (not the VICIdial servers -- you don't want rsync load on your telephony boxes during peak dialing).

## Deploying the Transcription Engine

This is where the magic (and the compute cost) lives. You need a speech-to-text model that can process hundreds of recordings per day with acceptable accuracy.

### Option 1: OpenAI Whisper (Open Source, Self-Hosted)

Whisper is OpenAI's open-source speech recognition model. The `large-v3` model delivers near-human accuracy on English call center audio. It runs on any NVIDIA GPU with 10+ GB VRAM.

Install Whisper on your GPU server:

```bash
pip install openai-whisper

# Or for faster inference, use faster-whisper (CTranslate2 backend)
pip install faster-whisper
```

faster-whisper is the practical choice for production. It runs 4x faster than vanilla Whisper with the same accuracy, and uses half the VRAM.

### Batch Transcription Script

```python
#!/usr/bin/env python3
"""batch_transcribe.py - Transcribe call recordings using faster-whisper"""

import os
import sys
import json
import glob
from datetime import datetime
from faster_whisper import WhisperModel

MODEL_SIZE = "large-v3"
DEVICE = "cuda"        # use "cpu" if no GPU (much slower)
COMPUTE_TYPE = "float16"  # use "int8" on older GPUs for speed
RECORDINGS_DIR = "/data/recordings"
TRANSCRIPTS_DIR = "/data/transcripts"
BATCH_SIZE = 100

model = WhisperModel(MODEL_SIZE, device=DEVICE, compute_type=COMPUTE_TYPE)

def transcribe_file(wav_path):
    """Transcribe a single WAV file and return segments with timestamps."""
    segments, info = model.transcribe(wav_path, beam_size=5, language="en")
    result = {
        "file": os.path.basename(wav_path),
        "language": info.language,
        "language_probability": round(info.language_probability, 3),
        "duration_seconds": round(info.duration, 1),
        "segments": []
    }
    full_text = []
    for segment in segments:
        result["segments"].append({
            "start": round(segment.start, 2),
            "end": round(segment.end, 2),
            "text": segment.text.strip()
        })
        full_text.append(segment.text.strip())
    result["full_text"] = " ".join(full_text)
    return result

def process_day(date_str):
    """Process all recordings for a given date."""
    day_dir = os.path.join(RECORDINGS_DIR, date_str.replace("-", "/"))
    if not os.path.isdir(day_dir):
        print(f"No recordings directory for {date_str}")
        return

    out_dir = os.path.join(TRANSCRIPTS_DIR, date_str.replace("-", "/"))
    os.makedirs(out_dir, exist_ok=True)

    wav_files = glob.glob(os.path.join(day_dir, "*.wav"))
    already_done = set(
        f.replace(".json", ".wav")
        for f in os.listdir(out_dir) if f.endswith(".json")
    )
    pending = [f for f in wav_files if os.path.basename(f) not in already_done]
    print(f"{date_str}: {len(pending)} new recordings to transcribe "
          f"({len(already_done)} already done)")

    for i, wav_path in enumerate(pending[:BATCH_SIZE]):
        try:
            result = transcribe_file(wav_path)
            out_file = os.path.join(
                out_dir,
                os.path.basename(wav_path).replace(".wav", ".json")
            )
            with open(out_file, "w") as f:
                json.dump(result, f, indent=2)
            if (i + 1) % 10 == 0:
                print(f"  Transcribed {i+1}/{len(pending[:BATCH_SIZE])}")
        except Exception as e:
            print(f"  ERROR transcribing {wav_path}: {e}")

if __name__ == "__main__":
    target_date = sys.argv[1] if len(sys.argv) > 1 else datetime.now().strftime("%Y-%m-%d")
    process_day(target_date)
```

On an NVIDIA RTX 3090 with faster-whisper large-v3, expect about 30 seconds of processing per minute of audio. A 3-minute call takes ~90 seconds to transcribe. A 50-agent operation generating 500 calls per day (average 4 minutes each) needs about 17 hours of GPU time -- just barely doable in 24 hours on a single GPU.

For larger operations, run multiple GPUs or use the `medium` model (2x faster, ~3% lower accuracy). The accuracy tradeoff is worth it above 1,000 calls per day.

### Option 2: Cloud Transcription APIs

If you do not want to manage GPU infrastructure:

| Provider | Cost per Minute | Accuracy | Latency |
|---|:---:|:---:|:---:|
| Deepgram | $0.0043 | 95%+ | Real-time |
| AssemblyAI | $0.0065 | 94%+ | Near real-time |
| Google Speech-to-Text | $0.009 | 93%+ | Near real-time |
| AWS Transcribe | $0.024 | 92%+ | Batch |

At $0.0043/minute (Deepgram), a 50-agent operation with 500 calls averaging 4 minutes costs about $86/day or ~$2,580/month. Cheaper than enterprise speech analytics platforms, but it adds up.

Self-hosted Whisper costs you only the GPU hardware (~$1,500 one-time for a used RTX 3090 or $300/month for a cloud GPU instance), making it the better choice if you have the engineering capacity to maintain it.

## Building Keyword and Compliance Detection

With transcripts in hand, the next layer is keyword detection. This is the simplest and highest-ROI part of the pipeline.

### Compliance Keyword Lists

Define keyword groups based on what your QA team needs to catch:

```json
{
  "compliance_required": {
    "description": "Phrases agents MUST say on every call",
    "phrases": [
      "this call may be recorded",
      "this call is being recorded",
      "for quality and training purposes",
      "my name is"
    ],
    "alert_on": "missing"
  },
  "compliance_prohibited": {
    "description": "Phrases agents must NEVER say",
    "phrases": [
      "i guarantee",
      "guaranteed results",
      "no risk",
      "100 percent",
      "i promise"
    ],
    "alert_on": "present"
  },
  "competitor_mentions": {
    "description": "Competitor names for competitive intelligence",
    "phrases": [
      "five9", "convoso", "genesys", "talkdesk",
      "ringcentral", "nice incontact", "dialpad"
    ],
    "alert_on": "present"
  },
  "buying_signals": {
    "description": "Positive buying indicators",
    "phrases": [
      "how much does it cost",
      "what are the terms",
      "when can we start",
      "send me the contract",
      "sounds good"
    ],
    "alert_on": "present"
  },
  "objection_patterns": {
    "description": "Common customer objections",
    "phrases": [
      "too expensive",
      "not interested",
      "already have",
      "call me back",
      "take me off your list"
    ],
    "alert_on": "present"
  }
}
```

### Keyword Detection Script

```python
#!/usr/bin/env python3
"""keyword_scan.py - Scan transcripts for compliance and intelligence keywords"""

import json
import os
import glob
from datetime import datetime

KEYWORDS_FILE = "/data/analytics/keyword_config.json"
TRANSCRIPTS_DIR = "/data/transcripts"

def load_keywords():
    with open(KEYWORDS_FILE) as f:
        return json.load(f)

def scan_transcript(transcript, keywords):
    """Scan a single transcript against all keyword groups."""
    text_lower = transcript["full_text"].lower()
    results = {"file": transcript["file"], "flags": [], "score": 100}

    for group_name, group in keywords.items():
        matched = [p for p in group["phrases"] if p.lower() in text_lower]

        if group["alert_on"] == "missing":
            missing = [p for p in group["phrases"] if p.lower() not in text_lower]
            if missing:
                results["flags"].append({
                    "group": group_name,
                    "type": "missing_required",
                    "missing_phrases": missing,
                    "severity": "high"
                })
                results["score"] -= 15 * len(missing)

        elif group["alert_on"] == "present" and matched:
            severity = "high" if "prohibited" in group_name else "info"
            results["flags"].append({
                "group": group_name,
                "type": "detected",
                "matched_phrases": matched,
                "severity": severity
            })
            if "prohibited" in group_name:
                results["score"] -= 25 * len(matched)

    results["score"] = max(0, results["score"])
    return results

def scan_day(date_str):
    """Scan all transcripts for a given date."""
    keywords = load_keywords()
    day_dir = os.path.join(TRANSCRIPTS_DIR, date_str.replace("-", "/"))
    if not os.path.isdir(day_dir):
        return []

    results = []
    for json_file in glob.glob(os.path.join(day_dir, "*.json")):
        with open(json_file) as f:
            transcript = json.load(f)
        result = scan_transcript(transcript, keywords)
        results.append(result)

    flagged = [r for r in results if r["flags"]]
    print(f"{date_str}: {len(results)} calls scanned, "
          f"{len(flagged)} flagged ({len(flagged)/max(len(results),1)*100:.1f}%)")

    high_severity = [r for r in results if any(
        fl["severity"] == "high" for fl in r["flags"]
    )]
    if high_severity:
        print(f"  HIGH SEVERITY FLAGS: {len(high_severity)} calls need immediate review")
        for r in high_severity[:5]:
            print(f"    {r['file']}: score={r['score']}, "
                  f"flags={[fl['group'] for fl in r['flags']]}")

    return results
```

This runs in seconds even on thousands of transcripts because it is just string matching. No GPU needed. Schedule it to run right after transcription completes.

## Adding Sentiment Analysis

Sentiment scoring tells you how calls *feel*, not just what was said. An agent who hits every compliance checkbox but sounds dead inside is still going to lose sales.

### Lightweight Sentiment Scoring

You do not need a massive NLP model for call center sentiment. A fine-tuned transformer running on CPU handles it fine. The `cardiffnlp/twitter-roberta-base-sentiment-latest` model from Hugging Face is a solid starting point -- it is small, fast, and accurate enough for conversation segments.

```python
#!/usr/bin/env python3
"""sentiment_score.py - Score transcript segments for sentiment"""

from transformers import pipeline
import json
import os
import sys

sentiment_model = pipeline(
    "sentiment-analysis",
    model="cardiffnlp/twitter-roberta-base-sentiment-latest",
    device=-1  # CPU; use 0 for GPU
)

LABEL_MAP = {"positive": 1.0, "neutral": 0.0, "negative": -1.0}

def score_transcript(transcript_path):
    """Score each segment and compute call-level sentiment metrics."""
    with open(transcript_path) as f:
        data = json.load(f)

    segments = data.get("segments", [])
    if not segments:
        return None

    scored_segments = []
    for seg in segments:
        text = seg["text"].strip()
        if len(text) < 10:
            continue
        try:
            result = sentiment_model(text[:512])[0]
            scored_segments.append({
                "start": seg["start"],
                "end": seg["end"],
                "text": text,
                "sentiment": result["label"],
                "confidence": round(result["score"], 3),
                "sentiment_value": LABEL_MAP.get(result["label"], 0.0)
            })
        except Exception:
            pass

    if not scored_segments:
        return None

    values = [s["sentiment_value"] for s in scored_segments]
    n = len(values)
    first_third = values[:n//3] if n >= 3 else values
    last_third = values[-(n//3):] if n >= 3 else values

    return {
        "file": data["file"],
        "total_segments": len(scored_segments),
        "average_sentiment": round(sum(values) / len(values), 3),
        "opening_sentiment": round(sum(first_third) / max(len(first_third), 1), 3),
        "closing_sentiment": round(sum(last_third) / max(len(last_third), 1), 3),
        "sentiment_trajectory": round(
            (sum(last_third) / max(len(last_third), 1)) -
            (sum(first_third) / max(len(first_third), 1)), 3
        ),
        "negative_segment_count": sum(1 for s in scored_segments if s["sentiment"] == "negative"),
        "positive_segment_count": sum(1 for s in scored_segments if s["sentiment"] == "positive"),
        "segments": scored_segments
    }
```

### What the Sentiment Numbers Mean

| Metric | Good Range | Warning | Action Required |
|---|:---:|:---:|---|
| Average Sentiment | 0.1 to 0.5 | -0.1 to 0.1 | Below -0.1 |
| Opening Sentiment | 0.2+ | 0.0 to 0.2 | Below 0.0 |
| Closing Sentiment | 0.3+ | 0.0 to 0.3 | Below 0.0 |
| Sentiment Trajectory | Positive (going up) | Flat | Negative (going down) |
| Negative Segment % | Under 15% | 15-30% | Over 30% |

The most important metric is **sentiment trajectory**. A call that starts negative and ends positive means the agent recovered the situation. A call that starts positive and ends negative means the agent lost the customer during the conversation -- and that is where your coaching should focus.

## Connecting Analytics to Your QA Workflow

The analytics pipeline produces three outputs: transcripts, keyword flags, and sentiment scores. The last step is turning those into actionable QA workflow.

### Priority-Based Review Queue

Instead of random sampling, build a review queue that prioritizes calls based on risk:

```python
def build_review_queue(keyword_results, sentiment_results):
    """Merge keyword and sentiment data into a prioritized review queue."""
    queue = []
    sentiment_lookup = {s["file"]: s for s in sentiment_results if s}

    for kr in keyword_results:
        sent = sentiment_lookup.get(kr["file"], {})
        priority = 0

        # High-severity keyword flags
        high_flags = [f for f in kr["flags"] if f["severity"] == "high"]
        priority += len(high_flags) * 30

        # Low keyword QA score
        if kr["score"] < 60:
            priority += 20

        # Negative sentiment trajectory
        trajectory = sent.get("sentiment_trajectory", 0)
        if trajectory < -0.3:
            priority += 25

        # High negative segment count
        neg_pct = sent.get("negative_segment_count", 0) / max(
            sent.get("total_segments", 1), 1)
        if neg_pct > 0.3:
            priority += 15

        if priority > 0:
            queue.append({
                "file": kr["file"],
                "priority": priority,
                "keyword_score": kr["score"],
                "flags": kr["flags"],
                "sentiment_avg": sent.get("average_sentiment"),
                "sentiment_trajectory": trajectory,
                "review_reasons": []
            })
            if high_flags:
                queue[-1]["review_reasons"].append("compliance_flag")
            if trajectory < -0.3:
                queue[-1]["review_reasons"].append("negative_trajectory")
            if kr["score"] < 60:
                queue[-1]["review_reasons"].append("low_qa_score")

    queue.sort(key=lambda x: -x["priority"])
    return queue
```

This gives your QA team a sorted list where the worst calls float to the top. A compliance violation with negative sentiment trajectory gets reviewed first. A clean call with neutral sentiment gets skipped entirely.

### Agent-Level Reporting

Aggregate the per-call data to agent-level metrics for coaching:

```sql
SELECT
    agent_id,
    COUNT(*) AS total_calls,
    AVG(keyword_score) AS avg_qa_score,
    AVG(avg_sentiment) AS avg_sentiment,
    AVG(sentiment_trajectory) AS avg_trajectory,
    SUM(CASE WHEN keyword_score < 60 THEN 1 ELSE 0 END) AS low_score_calls,
    SUM(CASE WHEN has_compliance_flag = 1 THEN 1 ELSE 0 END) AS compliance_flags
FROM call_analytics
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY agent_id
ORDER BY avg_qa_score ASC;
```

Agents with consistently low QA scores or high compliance flag counts need targeted coaching. Agents with negative sentiment trajectories might be burning out -- route them to the conversation about workload and [break scheduling](/blog/call-center-agent-burnout/).

### The Full Pipeline Schedule

Put it all together with a cron schedule on your analytics server:

```bash
# crontab for speech analytics pipeline
# Sync recordings every hour during business hours
0 8-20 * * 1-5 /data/scripts/sync-recordings.sh >> /var/log/analytics/sync.log 2>&1

# Transcribe previous day's recordings overnight
0 22 * * * python3 /data/scripts/batch_transcribe.py $(date -d yesterday +%Y-%m-%d) >> /var/log/analytics/transcribe.log 2>&1

# Run keyword scan after transcription
0 4 * * * python3 /data/scripts/keyword_scan.py $(date -d yesterday +%Y-%m-%d) >> /var/log/analytics/keywords.log 2>&1

# Run sentiment scoring after keywords
0 5 * * * python3 /data/scripts/sentiment_score.py $(date -d yesterday +%Y-%m-%d) >> /var/log/analytics/sentiment.log 2>&1

# Generate review queue and agent reports
0 6 * * * python3 /data/scripts/build_reports.py $(date -d yesterday +%Y-%m-%d) >> /var/log/analytics/reports.log 2>&1
```

By 6 AM, yesterday's calls are fully transcribed, scanned, scored, and prioritized. Your QA team starts their day with a ready-to-go review queue instead of randomly picking recordings.

## Cost Comparison: Build vs. Buy

Here is the honest math on building this yourself versus buying an enterprise platform:

| Component | Self-Hosted Cost | Enterprise Platform |
|---|:---:|:---:|
| Transcription (500 calls/day) | $300/mo (GPU) or $2,500/mo (API) | Included |
| Sentiment Analysis | $0 (open-source model, CPU) | Included |
| Keyword Detection | $0 (pattern matching) | Included |
| QA Dashboard | Engineering time (40-80 hours) | Included |
| Real-Time Alerts | Engineering time (20-40 hours) | Included |
| Agent Coaching Workflow | Manual process | Automated |
| Total Year 1 | $4K-$30K + 60-120 hrs engineering | $50K-$200K |
| Total Year 2+ | $4K-$30K/year | $50K-$200K/year |

The break-even point depends on your engineering capacity. If you have a developer who can build and maintain the pipeline, self-hosting saves $20K-$170K per year. If you don't, the engineering cost might push you toward a mid-tier vendor like Observe.AI or Level AI that costs $30K-$60K.

For VICIdial operations specifically, the self-hosted route makes more sense because you already have the infrastructure -- the recordings, the database, the server environment. The teams at [ViciStack](https://vicistack.com/) have deployed this pipeline across multiple call centers, and the typical implementation takes 2-3 weeks from recording configuration to working QA dashboard.

## What to Do First

Start with transcription. Just get your recordings turned into text. Even without keyword detection or sentiment analysis, searchable transcripts change how your QA team works.

The next step is the compliance keyword scan -- it is the highest-ROI piece because a single compliance violation can cost more than the entire analytics pipeline.

Sentiment scoring comes last. It is valuable for coaching and agent development, but it does not prevent fires the way compliance monitoring does.

If you want this built and running in two weeks instead of two months, [talk to us](https://vicistack.com/contact/). We have done this integration enough times that the architecture decisions are already made -- it is just configuration and deployment at this point.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/speech-analytics-call-center).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
