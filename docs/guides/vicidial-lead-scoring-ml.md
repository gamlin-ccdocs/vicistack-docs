# Machine Learning Lead Scoring for VICIdial: Prioritize Your Best Leads

**A practical guide to building a lead scoring model that plugs into VICIdial. No PhD required. Extract your call history, train a model that predicts which leads are worth dialing, score your lists, and feed the results back into VICIdial's hopper. Real Python code, real SQL, real conversion lifts.**

---

## The Problem With Dialing Everything

Here's something that's true at every [outbound call center](/blog/ai-outbound-call-center-2026/) I've worked with: 60-80% of the leads in your list will never convert. They're disconnected numbers, wrong numbers, people who will never buy, numbers that have been called twelve times with no answer. Your agents dial them anyway, because the dialer doesn't know the difference between a lead that has a 35% chance of converting and one that has a 0.2% chance.

VICIdial's predictive dialer is good at keeping agents busy. It's not good at deciding *which* leads to dial. It works through the hopper in order — list position, or maybe randomized, or maybe weighted by list priority. But it doesn't know that leads from area code 469 in your dataset convert at 8.2% while leads from 313 convert at 1.1%. It doesn't know that leads who answered on the first attempt but said "call back later" are 4x more likely to close than leads who went to voicemail three times. It doesn't know that calling between 10 AM and 2 PM in the lead's local timezone yields double the contact rate of calling at 5 PM.

You know all of this intuitively. Your veteran agents know it. But nobody has quantified it, turned it into a score, and used that score to prioritize the hopper.

That's what we're building.

---

## What You Need

- VICIdial with at least 3 months of call history (more is better — 6-12 months is ideal)
- Python 3.8+ on any machine that can reach your VICIdial database
- Basic SQL knowledge (you're running VICIdial, so you have this)
- About 4 hours for the initial setup

You don't need a GPU. You don't need a data science team. You don't need to understand gradient descent. You need a training dataset, a Python script, and the willingness to let numbers override gut feelings.

---

## Step 1: Extract Your Training Data

The training dataset comes from your historical call outcomes. We want every lead that has a definitive outcome — either they converted (SALE, APPSET, QUALIFY, whatever your positive dispositions are) or they didn't (DNC, DEAD, DISSATISFIED, or enough failed attempts that we consider them dead).

```sql
-- training_data_extract.sql
-- Pull leads with definitive outcomes for model training

SELECT
    vl.lead_id,
    vl.phone_number,
    vl.phone_code,
    LEFT(vl.phone_number, 3) AS area_code,
    vl.state,
    vl.city,
    vl.postal_code,
    LEFT(vl.postal_code, 3) AS zip3,
    vl.source_id,
    vl.vendor_lead_code,
    vl.list_id,
    vl.gmt_offset_now,
    vl.called_count,
    DATEDIFF(NOW(), vl.entry_date) AS lead_age_days,
    DATEDIFF(NOW(), vl.last_local_call_time) AS days_since_last_call,
    vl.status AS current_status,

    -- Call history features
    call_stats.total_attempts,
    call_stats.total_talk_seconds,
    call_stats.avg_talk_seconds,
    call_stats.max_talk_seconds,
    call_stats.human_answers,
    call_stats.voicemails,
    call_stats.no_answers,
    call_stats.busys,
    call_stats.attempts_morning,
    call_stats.attempts_afternoon,
    call_stats.attempts_evening,
    call_stats.first_call_hour,
    call_stats.best_call_hour,

    -- Target variable: did this lead result in a positive outcome?
    CASE
        WHEN vl.status IN ('SALE','XFER','APPSET','QUALIFY','DM_SALE') THEN 1
        ELSE 0
    END AS converted

FROM vicidial_list vl
LEFT JOIN (
    SELECT
        lead_id,
        COUNT(*) AS total_attempts,
        SUM(length_in_sec) AS total_talk_seconds,
        AVG(CASE WHEN length_in_sec > 0 THEN length_in_sec END) AS avg_talk_seconds,
        MAX(length_in_sec) AS max_talk_seconds,
        SUM(CASE WHEN status IN ('SALE','XFER','APPSET','CALLBK','A','B')
            THEN 1 ELSE 0 END) AS human_answers,
        SUM(CASE WHEN status IN ('AA','AM','AL') THEN 1 ELSE 0 END) AS voicemails,
        SUM(CASE WHEN status IN ('NA','N') THEN 1 ELSE 0 END) AS no_answers,
        SUM(CASE WHEN status = 'B' THEN 1 ELSE 0 END) AS busys,
        SUM(CASE WHEN HOUR(call_date) BETWEEN 8 AND 11 THEN 1 ELSE 0 END) AS attempts_morning,
        SUM(CASE WHEN HOUR(call_date) BETWEEN 12 AND 16 THEN 1 ELSE 0 END) AS attempts_afternoon,
        SUM(CASE WHEN HOUR(call_date) BETWEEN 17 AND 20 THEN 1 ELSE 0 END) AS attempts_evening,
        MIN(HOUR(call_date)) AS first_call_hour,
        HOUR(MAX(CASE WHEN length_in_sec > 30 THEN call_date END)) AS best_call_hour
    FROM vicidial_log
    WHERE call_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
    GROUP BY lead_id
) call_stats ON vl.lead_id = call_stats.lead_id

WHERE vl.status IN (
    -- Positive outcomes
    'SALE','XFER','APPSET','QUALIFY','DM_SALE',
    -- Negative/terminal outcomes
    'DNC','DNCL','DEAD','NI','NP','DISC','WRONG',
    -- Enough attempts to consider dead
    'NA','B','AA','AM'
)
AND vl.entry_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
AND call_stats.total_attempts >= 1;
```

Export to CSV:

```bash
mysql -u reporting -pReportPass123 asterisk -e "
$(cat training_data_extract.sql)
" | tr '\t' ',' > /tmp/training_data.csv

# Check row count
wc -l /tmp/training_data.csv
# You want at least 10,000 rows. 50,000+ is better.
```

### How Much Data Do You Need?

Rule of thumb:
- **10,000 leads** with outcomes — minimum viable model. Will overfit on rare features.
- **50,000 leads** — solid model. Can capture area code and time-of-day patterns.
- **200,000+ leads** — excellent model. Can capture granular zip-code-level patterns.

If you have fewer than 10,000 completed leads, keep collecting data for another month before training. A model trained on thin data will give you garbage scores and you'll lose trust in the whole approach.

---

## Step 2: Feature Engineering

Raw data isn't useful to a model. You need to transform it into features that capture patterns. Here's the Python script that loads the CSV and engineers features:

```python
#!/usr/bin/env python3
"""
lead_scoring.py — Train and deploy a lead scoring model for VICIdial
"""

import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import (
    classification_report, roc_auc_score, precision_recall_curve
)
from sklearn.preprocessing import LabelEncoder
import joblib
import warnings
warnings.filterwarnings('ignore')


def load_and_prepare_data(csv_path):
    """Load training data and engineer features."""
    df = pd.read_csv(csv_path)
    print(f"Loaded {len(df)} records")
    print(f"Conversion rate: {df['converted'].mean():.2%}")

    # Fill NaN values
    df['total_attempts'] = df['total_attempts'].fillna(0)
    df['total_talk_seconds'] = df['total_talk_seconds'].fillna(0)
    df['avg_talk_seconds'] = df['avg_talk_seconds'].fillna(0)
    df['max_talk_seconds'] = df['max_talk_seconds'].fillna(0)
    df['human_answers'] = df['human_answers'].fillna(0)
    df['voicemails'] = df['voicemails'].fillna(0)
    df['no_answers'] = df['no_answers'].fillna(0)
    df['busys'] = df['busys'].fillna(0)
    df['attempts_morning'] = df['attempts_morning'].fillna(0)
    df['attempts_afternoon'] = df['attempts_afternoon'].fillna(0)
    df['attempts_evening'] = df['attempts_evening'].fillna(0)
    df['lead_age_days'] = df['lead_age_days'].fillna(999)
    df['days_since_last_call'] = df['days_since_last_call'].fillna(999)

    # ---- Engineered Features ----

    # Contact rate: what fraction of attempts reached a human?
    df['contact_rate'] = np.where(
        df['total_attempts'] > 0,
        df['human_answers'] / df['total_attempts'],
        0
    )

    # Voicemail rate: high VM rate suggests the number is valid but person avoids calls
    df['vm_rate'] = np.where(
        df['total_attempts'] > 0,
        df['voicemails'] / df['total_attempts'],
        0
    )

    # Engagement score: longer average talk time = more engaged lead
    df['engagement_score'] = np.log1p(df['avg_talk_seconds'])

    # Time-of-day preference (when does this lead answer?)
    df['pref_morning'] = np.where(
        df['total_attempts'] > 0,
        df['attempts_morning'] / df['total_attempts'],
        0.33
    )
    df['pref_afternoon'] = np.where(
        df['total_attempts'] > 0,
        df['attempts_afternoon'] / df['total_attempts'],
        0.33
    )
    df['pref_evening'] = np.where(
        df['total_attempts'] > 0,
        df['attempts_evening'] / df['total_attempts'],
        0.33
    )

    # Lead freshness: newer leads convert better
    df['lead_freshness'] = np.exp(-df['lead_age_days'] / 30)

    # Recency: how recently was this lead contacted?
    df['recency_score'] = np.exp(-df['days_since_last_call'] / 7)

    # Attempt saturation: diminishing returns after N attempts
    df['attempt_saturation'] = 1 - np.exp(-df['total_attempts'] / 5)

    # Area code encoding (top N area codes get their own feature)
    df['area_code'] = df['area_code'].astype(str)
    area_code_counts = df['area_code'].value_counts()
    top_area_codes = area_code_counts[area_code_counts >= 50].index.tolist()
    df['area_code_group'] = df['area_code'].apply(
        lambda x: x if x in top_area_codes else 'OTHER'
    )

    # Encode area code groups
    le_area = LabelEncoder()
    df['area_code_encoded'] = le_area.fit_transform(df['area_code_group'])

    # State encoding
    le_state = LabelEncoder()
    df['state'] = df['state'].fillna('UNK')
    df['state_encoded'] = le_state.fit_transform(df['state'])

    # Source encoding
    le_source = LabelEncoder()
    df['source_id'] = df['source_id'].fillna('UNK').astype(str)
    df['source_encoded'] = le_source.fit_transform(df['source_id'])

    # GMT offset (proxy for timezone behavior)
    df['gmt_offset_now'] = df['gmt_offset_now'].fillna(-5)

    return df, le_area, le_state, le_source


# Feature columns for the model
FEATURE_COLS = [
    'total_attempts', 'total_talk_seconds', 'avg_talk_seconds',
    'max_talk_seconds', 'human_answers', 'voicemails', 'no_answers',
    'busys', 'lead_age_days', 'days_since_last_call',
    'contact_rate', 'vm_rate', 'engagement_score',
    'pref_morning', 'pref_afternoon', 'pref_evening',
    'lead_freshness', 'recency_score', 'attempt_saturation',
    'area_code_encoded', 'state_encoded', 'source_encoded',
    'gmt_offset_now',
]


def train_model(df):
    """Train a gradient boosting classifier."""
    X = df[FEATURE_COLS]
    y = df['converted']

    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42, stratify=y
    )

    print(f"Training set: {len(X_train)} ({y_train.mean():.2%} positive)")
    print(f"Test set:     {len(X_test)} ({y_test.mean():.2%} positive)")

    # Gradient Boosting — good out of the box, handles mixed feature types
    model = GradientBoostingClassifier(
        n_estimators=300,
        max_depth=5,
        learning_rate=0.05,
        subsample=0.8,
        min_samples_leaf=20,
        max_features='sqrt',
        random_state=42,
    )

    model.fit(X_train, y_train)

    # Evaluate
    y_pred = model.predict(X_test)
    y_prob = model.predict_proba(X_test)[:, 1]

    print("\n--- Test Set Performance ---")
    print(classification_report(y_test, y_pred))
    print(f"ROC AUC: {roc_auc_score(y_test, y_prob):.4f}")

    # Cross-validation
    cv_scores = cross_val_score(model, X, y, cv=5, scoring='roc_auc')
    print(f"Cross-val AUC: {cv_scores.mean():.4f} (+/- {cv_scores.std():.4f})")

    # Feature importance
    print("\n--- Top 10 Features ---")
    importance = pd.DataFrame({
        'feature': FEATURE_COLS,
        'importance': model.feature_importances_
    }).sort_values('importance', ascending=False)
    for _, row in importance.head(10).iterrows():
        print(f"  {row['feature']:<25} {row['importance']:.4f}")

    return model


def score_leads(model, df):
    """Score all leads and return sorted results."""
    X = df[FEATURE_COLS]
    df['score'] = model.predict_proba(X)[:, 1]

    # Bucket into priority tiers
    df['priority'] = pd.cut(
        df['score'],
        bins=[0, 0.05, 0.15, 0.30, 1.0],
        labels=['LOW', 'MEDIUM', 'HIGH', 'HOT']
    )

    print("\n--- Score Distribution ---")
    print(df['priority'].value_counts())
    print(f"\nMean score: {df['score'].mean():.4f}")
    print(f"Median score: {df['score'].median():.4f}")

    return df


if __name__ == '__main__':
    import sys
    csv_path = sys.argv[1] if len(sys.argv) > 1 else '/tmp/training_data.csv'

    df, le_area, le_state, le_source = load_and_prepare_data(csv_path)
    model = train_model(df)

    # Save model and encoders
    joblib.dump(model, '/opt/vicidial/lead_scoring/model.pkl')
    joblib.dump(le_area, '/opt/vicidial/lead_scoring/le_area.pkl')
    joblib.dump(le_state, '/opt/vicidial/lead_scoring/le_state.pkl')
    joblib.dump(le_source, '/opt/vicidial/lead_scoring/le_source.pkl')
    print("\nModel saved to /opt/vicidial/lead_scoring/")
```

### What the Model Actually Learns

When you run this on a typical VICIdial dataset, the top features are usually:

1. **contact_rate** — Leads who have answered before are way more likely to answer again
2. **engagement_score** (log talk time) — Leads who talked for 2+ minutes in prior calls have real interest
3. **lead_freshness** — Newer leads convert at 2-5x the rate of 90-day-old leads
4. **attempt_saturation** — After 6-8 attempts with no human contact, the probability of ever reaching the lead drops below 3%
5. **area_code_encoded** — Some area codes just perform better for your specific product. It's demographics, income levels, regulatory environment, cultural factors.
6. **source_encoded** — The lead vendor matters enormously. Some sources are 4x better than others.

That list isn't surprising to anyone who's run an outbound floor. What's different is that the model quantifies it and turns it into a score between 0 and 1 for every single lead in your list. Not a gut feeling. A number.

---

## Step 3: Score Your Active Lists

Now take the trained model and score every lead that's still dialable:

```sql
-- active_leads_extract.sql
-- Pull active/dialable leads for scoring

SELECT
    vl.lead_id,
    LEFT(vl.phone_number, 3) AS area_code,
    vl.state,
    vl.source_id,
    vl.list_id,
    vl.gmt_offset_now,
    vl.called_count,
    DATEDIFF(NOW(), vl.entry_date) AS lead_age_days,
    DATEDIFF(NOW(), COALESCE(vl.last_local_call_time, vl.entry_date)) AS days_since_last_call,
    COALESCE(cs.total_attempts, 0) AS total_attempts,
    COALESCE(cs.total_talk_seconds, 0) AS total_talk_seconds,
    COALESCE(cs.avg_talk_seconds, 0) AS avg_talk_seconds,
    COALESCE(cs.max_talk_seconds, 0) AS max_talk_seconds,
    COALESCE(cs.human_answers, 0) AS human_answers,
    COALESCE(cs.voicemails, 0) AS voicemails,
    COALESCE(cs.no_answers, 0) AS no_answers,
    COALESCE(cs.busys, 0) AS busys,
    COALESCE(cs.attempts_morning, 0) AS attempts_morning,
    COALESCE(cs.attempts_afternoon, 0) AS attempts_afternoon,
    COALESCE(cs.attempts_evening, 0) AS attempts_evening,
    cs.first_call_hour,
    cs.best_call_hour
FROM vicidial_list vl
LEFT JOIN (
    SELECT
        lead_id,
        COUNT(*) AS total_attempts,
        SUM(length_in_sec) AS total_talk_seconds,
        AVG(CASE WHEN length_in_sec > 0 THEN length_in_sec END) AS avg_talk_seconds,
        MAX(length_in_sec) AS max_talk_seconds,
        SUM(CASE WHEN status IN ('SALE','XFER','APPSET','CALLBK','A','B')
            THEN 1 ELSE 0 END) AS human_answers,
        SUM(CASE WHEN status IN ('AA','AM','AL') THEN 1 ELSE 0 END) AS voicemails,
        SUM(CASE WHEN status IN ('NA','N') THEN 1 ELSE 0 END) AS no_answers,
        SUM(CASE WHEN status = 'B' THEN 1 ELSE 0 END) AS busys,
        SUM(CASE WHEN HOUR(call_date) BETWEEN 8 AND 11 THEN 1 ELSE 0 END) AS attempts_morning,
        SUM(CASE WHEN HOUR(call_date) BETWEEN 12 AND 16 THEN 1 ELSE 0 END) AS attempts_afternoon,
        SUM(CASE WHEN HOUR(call_date) BETWEEN 17 AND 20 THEN 1 ELSE 0 END) AS attempts_evening,
        MIN(HOUR(call_date)) AS first_call_hour,
        HOUR(MAX(CASE WHEN length_in_sec > 30 THEN call_date END)) AS best_call_hour
    FROM vicidial_log
    GROUP BY lead_id
) cs ON vl.lead_id = cs.lead_id
WHERE vl.status IN ('NEW','CALLBK','A','B','PDROP','DROP','AA','AM','AL','NA','N')
AND vl.list_id IN (
    SELECT list_id FROM vicidial_lists WHERE active = 'Y'
);
```

Score them:

```python
#!/usr/bin/env python3
"""
score_active_leads.py — Score active VICIdial leads and output results
"""

import pandas as pd
import numpy as np
import joblib
import pymysql
from lead_scoring import FEATURE_COLS, load_and_prepare_data

# Load model and encoders
model = joblib.load('/opt/vicidial/lead_scoring/model.pkl')
le_area = joblib.load('/opt/vicidial/lead_scoring/le_area.pkl')
le_state = joblib.load('/opt/vicidial/lead_scoring/le_state.pkl')
le_source = joblib.load('/opt/vicidial/lead_scoring/le_source.pkl')

# Connect and pull active leads
conn = pymysql.connect(
    host='127.0.0.1', user='reporting', password='ReportPass123',
    db='asterisk', charset='utf8mb4'
)

query = open('/opt/vicidial/lead_scoring/active_leads_extract.sql').read()
df = pd.read_sql(query, conn)
print(f"Active leads to score: {len(df)}")

# Apply same feature engineering as training
df['area_code'] = df['area_code'].astype(str)
df['area_code_group'] = df['area_code'].apply(
    lambda x: x if x in le_area.classes_ else 'OTHER'
)
df['area_code_encoded'] = le_area.transform(df['area_code_group'])

df['state'] = df['state'].fillna('UNK')
df['state'] = df['state'].apply(lambda x: x if x in le_state.classes_ else 'UNK')
df['state_encoded'] = le_state.transform(df['state'])

df['source_id'] = df['source_id'].fillna('UNK').astype(str)
df['source_id'] = df['source_id'].apply(lambda x: x if x in le_source.classes_ else 'UNK')
df['source_encoded'] = le_source.transform(df['source_id'])

# Engineered features
df['total_attempts'] = df['total_attempts'].fillna(0)
df['total_talk_seconds'] = df['total_talk_seconds'].fillna(0)
df['avg_talk_seconds'] = df['avg_talk_seconds'].fillna(0)
df['max_talk_seconds'] = df['max_talk_seconds'].fillna(0)
df['human_answers'] = df['human_answers'].fillna(0)
df['voicemails'] = df['voicemails'].fillna(0)
df['no_answers'] = df['no_answers'].fillna(0)
df['busys'] = df['busys'].fillna(0)
df['attempts_morning'] = df['attempts_morning'].fillna(0)
df['attempts_afternoon'] = df['attempts_afternoon'].fillna(0)
df['attempts_evening'] = df['attempts_evening'].fillna(0)
df['lead_age_days'] = df['lead_age_days'].fillna(999)
df['days_since_last_call'] = df['days_since_last_call'].fillna(999)
df['gmt_offset_now'] = df['gmt_offset_now'].fillna(-5)

df['contact_rate'] = np.where(df['total_attempts'] > 0, df['human_answers'] / df['total_attempts'], 0)
df['vm_rate'] = np.where(df['total_attempts'] > 0, df['voicemails'] / df['total_attempts'], 0)
df['engagement_score'] = np.log1p(df['avg_talk_seconds'])
df['pref_morning'] = np.where(df['total_attempts'] > 0, df['attempts_morning'] / df['total_attempts'], 0.33)
df['pref_afternoon'] = np.where(df['total_attempts'] > 0, df['attempts_afternoon'] / df['total_attempts'], 0.33)
df['pref_evening'] = np.where(df['total_attempts'] > 0, df['attempts_evening'] / df['total_attempts'], 0.33)
df['lead_freshness'] = np.exp(-df['lead_age_days'] / 30)
df['recency_score'] = np.exp(-df['days_since_last_call'] / 7)
df['attempt_saturation'] = 1 - np.exp(-df['total_attempts'] / 5)

# Score
X = df[FEATURE_COLS]
df['score'] = model.predict_proba(X)[:, 1]

# Output scored leads
output = df[['lead_id', 'list_id', 'score']].sort_values('score', ascending=False)
output.to_csv('/tmp/scored_leads.csv', index=False)

# Summary
print(f"\nScoring complete:")
print(f"  HOT  (>0.30): {(df['score'] > 0.30).sum()} leads")
print(f"  HIGH (>0.15): {((df['score'] > 0.15) & (df['score'] <= 0.30)).sum()} leads")
print(f"  MED  (>0.05): {((df['score'] > 0.05) & (df['score'] <= 0.15)).sum()} leads")
print(f"  LOW  (<=0.05): {(df['score'] <= 0.05).sum()} leads")

conn.close()
```

---

## Step 4: Feed Scores Into VICIdial

This is the critical integration step. You have scores. Now you need VICIdial to dial high-scoring leads first.

VICIdial's hopper (the queue of leads waiting to be dialed) is populated by the `VDhopper` process. It pulls leads based on list priority, status order, and the campaign's `hopper_level` setting. There are a few ways to use your ML scores to influence hopper priority.

### Option A: Use the rank Field (Simplest)

VICIdial's `vicidial_list` table has a `rank` field (integer, -99 to 9999). When the campaign's `lead_order` is set to include rank-based sorting, the hopper will prioritize higher-ranked leads.

In the VICIdial admin panel, go to **Campaigns > [Your Campaign] > Detail** and set **Lead Order** to one of the rank-based options like `DOWN COUNT 2nd NEW` or whichever includes rank priority for your workflow.

Then update the rank based on scores via the VICIdial API:

```python
#!/usr/bin/env python3
"""
update_lead_ranks.py — Push ML scores into VICIdial rank field via API
"""

import pandas as pd
import requests
import time

VICIDIAL_API = "https://your-vicidial-server/vicidial/non_agent_api.php"
API_USER = "apiuser"
API_PASS = "apipass123"

scored = pd.read_csv('/tmp/scored_leads.csv')

# Convert score (0-1) to rank (0-999)
scored['rank'] = (scored['score'] * 999).astype(int).clip(0, 999)

# Sort by score descending — update highest-priority leads first
scored = scored.sort_values('score', ascending=False)

updated = 0
errors = 0

for _, row in scored.iterrows():
    params = {
        'source': 'lead_scoring',
        'user': API_USER,
        'pass': API_PASS,
        'function': 'update_lead',
        'lead_id': int(row['lead_id']),
        'rank': int(row['rank']),
    }

    try:
        resp = requests.get(VICIDIAL_API, params=params, timeout=10)
        if 'SUCCESS' in resp.text:
            updated += 1
        else:
            errors += 1
            if errors < 10:
                print(f"Error updating {row['lead_id']}: {resp.text}")
    except Exception as e:
        errors += 1

    # Rate limit: VICIdial API can handle ~50 req/sec
    if updated % 50 == 0:
        time.sleep(1)

    if updated % 1000 == 0:
        print(f"Updated {updated} leads, {errors} errors")

print(f"\nDone. Updated: {updated}, Errors: {errors}")
```

### Option B: Use List Priority (Bulk Approach)

If updating individual ranks is too slow for large lists (100K+ leads), split your leads into priority lists based on score buckets:

1. Create 4 lists in VICIdial admin with different priorities:
   - List 9001 — HOT leads (score > 0.30), Campaign Priority: 9
   - List 9002 — HIGH leads (0.15-0.30), Campaign Priority: 7
   - List 9003 — MEDIUM leads (0.05-0.15), Campaign Priority: 5
   - List 9004 — LOW leads (< 0.05), Campaign Priority: 3

2. Move leads between lists via the API based on their scores:

```python
# Move leads to scored lists via API
for _, row in scored.iterrows():
    if row['score'] > 0.30:
        target_list = 9001
    elif row['score'] > 0.15:
        target_list = 9002
    elif row['score'] > 0.05:
        target_list = 9003
    else:
        target_list = 9004

    params = {
        'source': 'lead_scoring',
        'user': API_USER,
        'pass': API_PASS,
        'function': 'update_lead',
        'lead_id': int(row['lead_id']),
        'list_id': target_list,
    }
    requests.get(VICIDIAL_API, params=params, timeout=10)
```

Then attach all four lists to your campaign. VICIdial will pull from the highest-priority list first, working through HOT leads before touching LOW leads.

### Option C: Custom Hopper Loading (Advanced)

For operations that want maximum control, you can write a custom hopper loader that replaces VICIdial's built-in `VDhopper` process. This is the nuclear option — more control, more complexity, more things that can break.

The approach: disable VICIdial's native hopper loading for your campaign and run your own cron script that populates the `vicidial_hopper` table in score order. This uses the VICIdial API to load leads into the hopper:

```bash
#!/bin/bash
# /opt/vicidial/lead_scoring/load_hopper.sh
# Run every 30 seconds via cron

CAMPAIGN="OUTBOUND_B2B"
HOPPER_LEVEL=200  # How many leads to keep in the hopper

# Get current hopper count
CURRENT=$(mysql -u cron -pCronPass123 asterisk -N -e "
    SELECT COUNT(*) FROM vicidial_hopper WHERE campaign_id='${CAMPAIGN}' AND status='READY';
")

NEEDED=$((HOPPER_LEVEL - CURRENT))
if [ "$NEEDED" -le 0 ]; then
    exit 0
fi

# Pull top N undialed leads by score (from our scored table)
mysql -u cron -pCronPass123 asterisk -N -e "
    SELECT lead_id
    FROM vicidial_list vl
    WHERE vl.status IN ('NEW','CALLBK','A','B')
    AND vl.list_id IN (9001,9002,9003,9004)
    AND vl.lead_id NOT IN (
        SELECT lead_id FROM vicidial_hopper WHERE campaign_id='${CAMPAIGN}'
    )
    ORDER BY vl.rank DESC
    LIMIT ${NEEDED};
" | while read LEAD_ID; do
    # Use API to add to hopper
    curl -s "https://your-vicidial-server/vicidial/non_agent_api.php?\
source=hopper_load&user=apiuser&pass=apipass123&\
function=add_lead_to_hopper&lead_id=${LEAD_ID}&campaign_id=${CAMPAIGN}" > /dev/null
done
```

---

## Step 5: Automate the Workflow

Set up a daily cron job that retrains the model weekly and rescores leads daily:

```bash
# /etc/cron.d/lead-scoring

# Rescore active leads every morning at 6 AM before the dialing shift starts
0 6 * * 1-6 root /usr/bin/python3 /opt/vicidial/lead_scoring/score_active_leads.py >> /var/log/lead-scoring.log 2>&1

# Update VICIdial ranks after scoring
30 6 * * 1-6 root /usr/bin/python3 /opt/vicidial/lead_scoring/update_lead_ranks.py >> /var/log/lead-scoring.log 2>&1

# Retrain model every Sunday night with latest data
0 22 * * 0 root /opt/vicidial/lead_scoring/retrain.sh >> /var/log/lead-scoring.log 2>&1
```

The retrain script:

```bash
#!/bin/bash
# /opt/vicidial/lead_scoring/retrain.sh

echo "[$(date)] Starting model retrain..."

# Extract fresh training data
mysql -u reporting -pReportPass123 asterisk -e "
$(cat /opt/vicidial/lead_scoring/training_data_extract.sql)
" | tr '\t' ',' > /tmp/training_data.csv

ROW_COUNT=$(wc -l < /tmp/training_data.csv)
echo "Training data rows: $ROW_COUNT"

if [ "$ROW_COUNT" -lt 5000 ]; then
    echo "ERROR: Not enough training data. Skipping retrain."
    exit 1
fi

# Backup current model
cp /opt/vicidial/lead_scoring/model.pkl \
   /opt/vicidial/lead_scoring/model.pkl.$(date +%Y%m%d)

# Train new model
python3 /opt/vicidial/lead_scoring/lead_scoring.py /tmp/training_data.csv

echo "[$(date)] Retrain complete."
```

---

## Step 6: Measuring the Impact

The whole point of this exercise is to see conversion rates go up and cost-per-acquisition go down. Here's how to measure it.

### A/B Test: Scored vs Unscored

The cleanest test: run two identical campaigns, same agents, same hours, same scripts. One campaign uses ML-scored lead ordering. The other uses VICIdial's default ordering. Run for two weeks and compare.

```sql
-- Compare scored campaign vs control campaign
SELECT
    campaign_id,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN length_in_sec > 0 THEN 1 ELSE 0 END) AS contacts,
    ROUND(SUM(CASE WHEN length_in_sec > 0 THEN 1 ELSE 0 END) /
          COUNT(*) * 100, 2) AS contact_rate,
    SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) AS sales,
    ROUND(SUM(CASE WHEN status = 'SALE' THEN 1 ELSE 0 END) /
          NULLIF(SUM(CASE WHEN length_in_sec > 0 THEN 1 ELSE 0 END), 0) * 100, 2) AS close_rate,
    ROUND(AVG(CASE WHEN length_in_sec > 0 THEN length_in_sec END), 0) AS avg_talk_sec
FROM vicidial_log
WHERE call_date >= '2026-04-01'
    AND call_date < '2026-04-15'
    AND campaign_id IN ('SCORED_B2B', 'CONTROL_B2B')
GROUP BY campaign_id;
```

Expected results from real deployments:

```
+-------------+-------+----------+---------+-------+--------+----------+
| campaign_id | calls | contacts | cont_rt | sales | cls_rt | avg_talk |
+-------------+-------+----------+---------+-------+--------+----------+
| SCORED_B2B  | 15420 |     4102 |   26.60 |   287 |   6.99 |      194 |
| CONTROL_B2B | 15180 |     3412 |   22.48 |   178 |   5.22 |      186 |
+-------------+-------+----------+---------+-------+--------+----------+
```

That's a 34% improvement in close rate and an 18% improvement in contact rate. The scored campaign reaches the right people more often and closes them at a higher rate because the leads it dials first are genuinely better.

### Lift Chart

Generate a lift chart to visualize how much better the model is than random dialing:

```python
from sklearn.metrics import roc_auc_score
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

# Score test set
y_test_probs = model.predict_proba(X_test)[:, 1]

# Sort by predicted probability
sorted_idx = np.argsort(-y_test_probs)
y_sorted = y_test.values[sorted_idx]

# Cumulative conversion at each decile
n = len(y_sorted)
deciles = np.arange(0.1, 1.1, 0.1)
random_conversions = y_sorted.mean()

print("Decile | % Leads | Cumulative Conversion | Lift")
print("-" * 55)
for d in deciles:
    idx = int(d * n)
    cum_conv = y_sorted[:idx].mean()
    lift = cum_conv / random_conversions
    print(f"  {d:.0%}   |  {d:.0%}    |    {cum_conv:.2%}              | {lift:.1f}x")
```

Typical output:

```
Decile | % Leads | Cumulative Conversion | Lift
-------------------------------------------------------
  10%  |  10%    |    18.42%              | 3.4x
  20%  |  20%    |    14.87%              | 2.7x
  30%  |  30%    |    11.93%              | 2.2x
  40%  |  40%    |    9.78%               | 1.8x
  50%  |  50%    |    8.32%               | 1.5x
  60%  |  60%    |    7.21%               | 1.3x
  70%  |  70%    |    6.44%               | 1.2x
  80%  |  80%    |    5.89%               | 1.1x
  90%  |  90%    |    5.56%               | 1.0x
  100% |  100%   |    5.41%               | 1.0x
```

That top decile delivers 3.4x the conversion rate of random dialing. In practical terms: if you only have time to dial 30% of your list today, the model tells you which 30% to dial to capture 66% of the available conversions.

---

## Common Pitfalls

**Training on biased data.** If your historical data only includes leads that were dialed during business hours, the model learns that business hours are great — but not because of the time, because that's the only data it has. Make sure your training data has variety in call times, lead sources, and attempt counts.

**Overfitting to area codes.** Area codes with fewer than 50 leads in your training set produce unreliable conversion estimates. The model might learn that area code 808 has a 50% conversion rate because you called 4 leads from Hawaii and 2 of them bought. That's noise, not signal. The `OTHER` bucketing in the feature engineering handles this, but watch for it.

**Ignoring model drift.** Your model was trained on March data. By June, your lead sources changed, your product pricing changed, your competitor launched, and your best closer quit. The model doesn't know any of this. Retrain monthly. Compare each retrained model's AUC against the previous one — if AUC drops by more than 0.05, something changed in your business and you need to investigate.

**Scoring leads that have no history.** Brand-new leads with zero call attempts have no behavioral features — no contact rate, no talk time, no voicemail history. The model can only use their demographic features (area code, state, source). The score for these leads will cluster around the overall conversion rate for their area code and source. That's fine — it's still better than no scoring. As the leads accumulate call history, rescore them and their scores will sharpen.

**Not excluding low-scoring leads.** If the model says a lead has a 0.3% chance of converting, and your cost per dial attempt is $0.15 (agent time + phone cost), then dialing that lead costs you $50 per conversion just in dial costs. Some leads aren't worth dialing at all. Set a minimum score threshold below which leads don't enter the hopper. For most operations, that threshold is somewhere between 0.02 and 0.05.

---

## The Math Behind the Lift

A typical outbound B2B campaign running VICIdial has these economics:

- Average deal value: $2,000
- Conversion rate (unscored): 4.5%
- Cost per dial attempt: $0.12 (agent time at $15/hr, 8 seconds average per attempt)
- Dials to contact: 4.5 attempts per human contact
- Cost per contact: $0.54
- Cost per sale: $12.00

With ML scoring (top 50% of leads only):

- Conversion rate: 7.2% (1.6x lift)
- Same cost per contact: $0.54
- Cost per sale: $7.50
- Sales per hour per agent: up from 1.8 to 2.9

The conversion lift is nice. The agent efficiency gain is what pays for everything. Your agents are spending their talk time on leads that are more likely to close, which means more sales per hour, which means higher commissions, lower turnover, and lower cost per acquisition. The model pays for itself in the first week.

If you want to take the scoring further, combine it with [VICIdial's lead recycling features](https://vicistack.com/blog/vicidial-lead-recycling/) to automatically rescore leads after each attempt and adjust their priority. And our guide on [VICIdial list management](https://vicistack.com/blog/vicidial-list-management/) covers the list structure and upload workflows that make multi-list scoring practical.

---

*Want help building a lead scoring model tuned to your specific VICIdial operation? [Contact ViciStack](https://vicistack.com/contact/) — we've deployed ML scoring across operations from 10 to 500 agents.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-lead-scoring-ml).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
