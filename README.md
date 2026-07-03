# Student Risk & Attendance Insight Assistant

An AI-powered decision support platform that helps teachers identify at-risk students early by combining structured analytics (BigQuery) with a conversational AI assistant (NVIDIA NIM / Llama 3.1 Nemotron).

Built for **Gen AI Academy APAC Edition Hackathon — Cohort 2**

🔗 **Live Demo:** https://nidhi199.github.io/student-risk-assistant/
📊 **Dashboard:** https://datastudio.google.com/reporting/edbfb2e1-055f-4011-8d68-0d775c12cfa8 
☁️ **Backend:** https://student-risk-assistant-739262828516.us-central1.run.app

---

## What I Built

A teacher-facing web application that:
- Ingests synthetic student data (attendance, test scores, assignment submission rates)
- Computes a **weighted risk score** per student using BigQuery SQL
- Displays a **visual dashboard** (Looker Studio) with class-level and student-level trends
- Provides a **natural language chat assistant** so a teacher can ask questions like *"Which students in 8B need attention this week?"* and receive grounded, explainable answers citing real numbers
- Filters results by class section for focused review

---

## What I Actually Did — Step by Step

### 1. Created the GCP Project
- Created a new Google Cloud project: `student-risk-assistant`
- Enabled required APIs: BigQuery, Cloud Run, Cloud Build, Vertex AI, Agent Platform

### 2. Generated Synthetic Dataset
- Created a Python script to generate 180 realistic student records across 10 class sections (6A–10B)
- Fields: `student_id`, `student_name`, `class_section`, `attendance_pct`, `test1_score`, `test2_score`, `test3_score`, `assignment_submission_rate`, `behavioral_notes`
- Students are distributed across three realistic profiles: strong (30%), average (45%), at-risk (25%)
- Saved as `students_sample.csv` — clearly labeled as synthetic data

### 3. Loaded Data into BigQuery
- Created dataset: `school_data`
- Loaded CSV into table: `students` using BigQuery's auto-detect schema
- Verified 180 rows loaded correctly

### 4. Built SQL Analytics Layer
Ran four key queries in BigQuery:

**Query 1 — Average attendance by class:**
```sql
SELECT class_section,
       ROUND(AVG(attendance_pct), 1) AS avg_attendance,
       COUNT(*) AS student_count
FROM `school_data.students`
GROUP BY class_section
ORDER BY avg_attendance ASC;
```

**Query 2 — Students below attendance threshold:**
```sql
SELECT student_id, student_name, class_section, attendance_pct
FROM `school_data.students`
WHERE attendance_pct < 65
ORDER BY attendance_pct ASC;
```

**Query 3 — Score trend per student:**
```sql
SELECT student_id, student_name,
       test1_score, test2_score, test3_score,
       ROUND(test3_score - test1_score, 1) AS score_change,
       CASE
         WHEN test3_score - test1_score >= 10 THEN 'Improving'
         WHEN test3_score - test1_score <= -10 THEN 'Declining'
         ELSE 'Stable'
       END AS trend
FROM `school_data.students`
ORDER BY score_change ASC;
```

**Query 4 — Weighted risk score view (the core analytics output):**
```sql
CREATE OR REPLACE VIEW `school_data.student_risk_scores` AS
SELECT
  student_id, student_name, class_section,
  attendance_pct,
  ROUND((test1_score + test2_score + test3_score) / 3, 1) AS avg_score,
  assignment_submission_rate,
  behavioral_notes,
  ROUND(
    100 - (
      attendance_pct * 0.40 +
      ((test1_score + test2_score + test3_score) / 3) * 0.35 +
      assignment_submission_rate * 0.25
    ), 1
  ) AS risk_score,
  CASE
    WHEN 100 - (attendance_pct * 0.40 +
         ((test1_score + test2_score + test3_score) / 3) * 0.35 +
         assignment_submission_rate * 0.25) >= 45 THEN 'High Risk'
    WHEN 100 - (attendance_pct * 0.40 +
         ((test1_score + test2_score + test3_score) / 3) * 0.35 +
         assignment_submission_rate * 0.25) >= 25 THEN 'Medium Risk'
    ELSE 'Low Risk'
  END AS risk_category
FROM `school_data.students`;
```

**Risk formula:**
```
risk_score = 100 − (attendance × 0.40 + avg_score × 0.35 + submission_rate × 0.25)
```
Higher score = higher risk. Weights chosen to reflect attendance as the strongest early-warning signal.

### 5. Built and Deployed Cloud Run Backend
- Built a Python Flask API (`main.py`) with two endpoints: `/health` and `/ask`
- Added CORS support (`flask-cors`) to allow browser requests from GitHub Pages
- Containerized using Docker
- Deployed to Cloud Run via `gcloud run deploy --source .`

**Troubleshooting faced and resolved:**
- Empty zip file error → fixed by ensuring all files were in the same folder before deploying
- Gunicorn worker boot error → fixed by syncing main.py and requirements.txt
- CORS error → fixed by adding `flask-cors` and `CORS(app)` to Flask app
- Vertex AI model access denied → all Gemini models returned 404 on the hackathon sandbox account (known restriction on Gen AI Academy sandbox projects)
- Switched to NVIDIA NIM API → resolved model access issue completely

### 6. Integrated NVIDIA NIM for AI Inference
- The hackathon sandbox project (`student-risk-assistant`) had Vertex AI publisher model access restrictions — all Gemini model variants (1.0-pro, 1.5-flash, 2.0-flash, 2.5-pro) returned 404 NOT_FOUND
- Signed up at **build.nvidia.com** and generated a free NVIDIA NIM API key
- Integrated `nvidia/llama-3.1-nemotron-nano-8b-v1` model via direct HTTP POST (OpenAI-compatible endpoint)
- No SDK needed — used Python `requests` library for clean, dependency-light integration
- Stored API key securely as a Cloud Run environment variable

**AI flow:**
```
Teacher types question
        ↓
Cloud Run pulls top 10 at-risk students from BigQuery
        ↓
Student data + question injected into prompt
        ↓
NVIDIA NIM (Llama 3.1 Nemotron) generates grounded answer
        ↓
Answer returned to teacher's browser
```

### 7. Built Looker Studio Dashboard
Connected Looker Studio directly to BigQuery `student_risk_scores` view and built:
- Bar chart: average risk score by class section
- Color-coded table: students sorted by risk score descending (red = High Risk, orange = Medium, green = Low)
- Bar chart: average score by class
- Scorecard: total student count
- Dropdown filter: filter by class section
- Conditional formatting: 3 rules applied for risk category colors

### 8. Built Front-End Chat UI
- Single-page HTML/JS chat interface (`index.html`)
- Dark theme, responsive layout
- Class section dropdown filter
- 4 quick-action suggestion chips
- Connects to Cloud Run `/ask` endpoint via `fetch` API
- Shows "Based on N student records" metadata with each response

### 9. Hosted on GitHub Pages
- Pushed `index.html` to GitHub repository
- Enabled GitHub Pages (Settings → Pages → main branch)
- Got a free public URL: `https://USERNAME.github.io/REPO/`
- No hosting cost, no server needed for the frontend

---

## Final Architecture

```
students_sample.csv (synthetic, 180 rows)
        │
        ▼
Cloud Storage → BigQuery (students table)
                      │
                      ▼
            SQL View: student_risk_scores
            (weighted risk formula)
                 │              │
                 ▼              ▼
      Looker Studio        Cloud Run
      Dashboard            Flask API
      (visual layer)            │
                                ▼
                         NVIDIA NIM API
                    (Llama 3.1 Nemotron)
                                │
                                ▼
                    GitHub Pages (index.html)
                      Teacher Chat UI
```

---

## Google Cloud Services Used

| Service | Role |
|---|---|
| **BigQuery** | Structured student data storage + SQL risk score view |
| **Cloud Run** | Serverless backend hosting Flask API |
| **Cloud Build** | Automated Docker image build during deployment |
| **Looker Studio** | Visual dashboard connected to BigQuery |
| **Agent Platform API** | Enabled on project (Vertex AI endpoint) |

---

## AI / External Services Used

| Service | Role |
|---|---|
| **NVIDIA NIM** | AI inference — Llama 3.1 Nemotron model |
| **GitHub Pages** | Free public frontend hosting |

---

## Demonstrated Acceleration

- Manual review of 180 student records to identify at-risk cases: **~2 hours** per class cycle
- This tool surfaces the same prioritized list with plain-language explanations: **under 10 seconds**
- BigQuery SQL view recalculates risk scores for all 180 students instantly on data refresh
- Scales to any number of students with zero additional manual effort

---

## Responsible AI Notes

- All data is **synthetic** — no real student PII was used at any stage
- AI assistant is explicitly constrained to answer **only from provided BigQuery data** — hallucination is prevented by design
- Risk scores are a **decision-support signal, not a verdict** — final judgment stays with the teacher
- Risk formula weights (40/35/25) are fully visible and explainable — not a black box model
- In a production deployment, real data would require anonymization, consent, and data protection safeguards

---

## Challenges Faced

| Challenge | How I Resolved It |
|---|---|
| Cloud Run deploy gave empty zip error | Ensured all 3 files were in same folder before deploying |
| Gemini model returned 404 on all variants | Switched to NVIDIA NIM API — worked immediately |
| CORS error blocking browser requests | Added flask-cors library and CORS(app) to Flask |
| Gunicorn worker boot failure | Synced imports between main.py and requirements.txt |
| API key quota exhausted (free tier limit 0) | Used NVIDIA NIM instead which has generous free tier |
| Wrong project linked to AI Studio key | Created fresh key linked to correct project |

---

## Files in This Submission

| File | Purpose |
|---|---|
| `students_sample.csv` | Synthetic dataset (180 students, 10 class sections) |
| `main.py` | Cloud Run Flask backend (BigQuery + NVIDIA NIM) |
| `requirements.txt` | Python dependencies |
| `Dockerfile` | Container config for Cloud Run |
| `index.html` | Teacher-facing chat UI (hosted on GitHub Pages) |

---

## Future Scope

- Replace synthetic data with real anonymized school information system data
- Add predictive forecasting: likelihood of dropout next term using historical trends
- Multi-language support (Hindi/regional languages) for Indian government schools
- Parent-facing notifications for High Risk flags, with teacher approval workflow
- Scheduled Cloud Run jobs for weekly automated risk re-evaluation alerts
- Upgrade AI model to a larger NVIDIA NIM model (e.g., Llama 3.3 70B) for richer explanations

---

## Team

**Nidhi** — AI & Computer Instructor, Freelance Web Developer
B.Tech Computer Science Engineering
Gen AI Academy APAC Cohort 2 Participant
