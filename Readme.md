# 🎮 Player Progression Analytics Dashboard  
### Avatar: Frontiers of Pandora

---

## 📌 Project Overview

Game teams often need to understand how players move through key content:

- Where do players stop playing?  
- How long does content take to complete?  
- Which player segments behave differently?  

To address this, I built an **end-to-end analytics solution** using:

- **SQL** for telemetry transformation and modeling  
- **Power BI** for interactive visualization  

This resulted in a **self-serve progression analytics dashboard** enabling designers, product managers, and analysts to explore player behavior in near real time.

---

## 🎯 Business Questions 

The dashboard was designed to answer:

- What percentage of players finish the core campaign?
- At which chapters or objectives do player drop-offs occur most?
- How long does it take players to complete major content?
- How do progression and completion vary by platform, region, or cohort?
- What do progression timelines look like as a function of playtime?

> All references to specific content names, mission titles, brands, or proprietary terminology have been replaced with generic placeholders.

---

## 🗂 Dataset Description

**Data Source:**  
Synthetic but structurally realistic telemetry modeled after internal game engine event data.  
All identifiers are anonymized and aggregated.

### Tables Used

#### 🔹 FactEvents
Raw player telemetry events such as:
- `ContentStarted`
- `ContentCompleted`
- `ContentAbandoned`
- `SessionStart`
- `SessionEnd`

---

#### 🔹 DimContent
Lookup table describing campaign structure (nomenclature is changed to generic terms):
- Chapter 1  
- Chapter 2  
- Objective 1  
- Objective 2  

---

#### 🔹 DimPlayer
- Player IDs  
- Platform  
- Region  
- Cohort Labels  

---

#### 🔹 DimDate
Standard calendar date dimension.

---

## 🧩 Data Modeling (Star Schema)

```
                   ┌───────────────────┐
                   │     DimDate       │───┐
                   └───────────────────┘   │
                   ┌───────────────────┐   │
                   │    DimPlayer      │───┼────┐
                   └───────────────────┘   │    │
                   ┌───────────────────┐   │    │
                   │   DimPlatform     │───┼────┤
                   └───────────────────┘   │    │
                   ┌───────────────────┐   │    │
                   │   DimContent      │───┼────┤
                   └───────────────────┘   │    │
                                           ▼    ▼
                                      ┌─────────────────┐
                                      │  FactProgress   │
                                      │ (content events)│
                                      └─────────────────┘
```

The model follows a clean **star schema design** to optimize Power BI performance and enable scalable reporting.

---

## 🛠 SQL Transformations 

### 1️⃣ Normalize Raw Telemetry into Fact Table

```sql
WITH raw_clean AS (
    SELECT
        player_id,
        event_time,
        event_name,
        content_id,
        SUM(playtime_seconds) OVER (
            PARTITION BY player_id 
            ORDER BY event_time
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cumulative_playtime
    FROM raw_events
    WHERE event_name IN (
        'ContentStarted',
        'ContentCompleted',
        'ContentAbandoned'
    )
)

SELECT
    player_id,
    content_id,
    CASE event_name
        WHEN 'ContentStarted' THEN 'Started'
        WHEN 'ContentCompleted' THEN 'Completed'
        WHEN 'ContentAbandoned' THEN 'Abandoned'
    END AS event_type,
    event_time,
    cumulative_playtime
FROM raw_clean;
```

---

### 2️⃣ Compute First Start/Completion per Player

```sql
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY player_id, content_id, event_type
            ORDER BY event_time
        ) AS rn
    FROM FactProgress
)

SELECT *
FROM ranked
WHERE rn = 1;
```

---

### 3️⃣ Compute Funnel Metrics

```sql
SELECT
    c.chapter_index,
    starters,
    completers,
    ROUND(100.0 * completers / NULLIF(starters, 0), 2) AS completion_rate
FROM funnel_aggregates;
```

---

## 📊 Power BI Dashboard 

### 🔹 Completion KPIs
- Overall completion rate  
- Average time to complete core content  
- Longest completion durations  
- Highest drop-off locations
  
<img width="1513" height="966" alt="PowerBI" src="https://github.com/user-attachments/assets/fff154d5-cb8b-4a8c-935f-983a40687291" />

---

### 🔻 Progression Funnel
Visualizes:
- % of players who start each chapter  
- % who complete each chapter  
- Drop-off between objectives  

---

### 📈 Progression Timeline
Completion curves plotted against:
- **Cumulative playtime (hours)**  
rather than calendar time.

---

### 🎛 Filters
- Region (Region A, B, C…)  
- Platform (Platform 1 / 2 / 3)  
- Ownership Type (Expansion Access)  
- Session length bucket  
- Content ID / Content Tier  


---

## 📐 DAX Measures 
```DAX
Total Starters :=
CALCULATE(
    DISTINCTCOUNT(FactProgress[PlayerId]),
    FactProgress[EventType] = "Started"
)

Total Completers :=
CALCULATE(
    DISTINCTCOUNT(FactProgress[PlayerId]),
    FactProgress[EventType] = "Completed"
)

Completion Rate % :=
DIVIDE(
    [Total Completers],
    [Total Starters],
    0
)

Avg Hours to Complete :=
AVERAGEX(
    FactContentDurations,
    FactContentDurations[HoursToComplete]
)
```

---

## 🔍 Key Insights

- Completion rates vary significantly by platform and region.
- A mid-campaign objective shows the highest player drop-off, suggesting pacing or difficulty imbalance.
- Median completion time aligns with expected targets, with a long-tail distribution among high-engagement players.
- Early-game progression strongly predicts full campaign completion likelihood.

---

## 🚀 Business Impact

- Enabled design teams to identify friction points quickly.
- Reduced repetitive ad-hoc SQL requests through self-serve reporting.
- Standardized progression evaluation across builds and updates.
- Enabled experiment teams to compare gameplay changes version-to-version.

---

## 🧠 Skills Demonstrated

- Advanced SQL (CTEs, Window Functions, Funnel Logic)
- Star Schema Data Modeling
- Power BI (DAX, KPI Design, Interactive Filtering)
- Funnel & Cohort Analysis
- Telemetry Normalization
- Analytical Storytelling

---

