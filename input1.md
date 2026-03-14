# Pipeline: Job Application Screener

## Scenario
A recruiter uploads a job description and a candidate's resume. The system runs three
independent specialist evaluations in parallel (skills match, experience quality, culture fit),
then aggregates them into a composite score, and finally generates a structured feedback
report for both the recruiter and the candidate.

---

## Project
- **Name**: job-screener
- **Output folder**: backend2/
- **Description**: Multi-specialist parallel evaluation pipeline for job application screening.

---

## Initial State
```
job_description: str   # The full job description text
resume_text: str       # The candidate's resume as plain text
```

---

## Agents

### 1. IngestAgent
- **Type**: BaseAgent wrapping one LlmAgent
- **LlmAgent field name**: `ingest_llm`
- **Reads**: `job_description`, `resume_text`
- **Writes**: `parsed_data`
- **Output schema**:
  - `candidate_name: str` — Candidate's full name extracted from resume
  - `role_title: str` — Job title being applied for, from the job description
  - `required_skills: list[str]` — Skills explicitly required in the job description
  - `candidate_skills: list[str]` — Skills listed or demonstrated in the resume
  - `years_of_experience: float` — Total years of relevant experience from resume
  - `education_level: Literal["high_school", "bachelors", "masters", "phd", "other"]` — Highest qualification
  - `seniority_required: Literal["junior", "mid", "senior", "lead", "any"]` — Seniority level from job description
- **Instruction summary**: You are a resume parser. Extract structured information from the job description and resume. Be precise and factual — only extract what is explicitly stated. Use `{job_description}` and `{resume_text}`.

---

### 2. EvaluationCouncilAgent
- **Type**: BaseAgent wrapping a ParallelAgent containing three LlmAgents
- **Reads**: `parsed_data`
- **Parallel sub-agents**:

#### 2a. SkillsMatcherAgent (inside parallel group)
  - **LlmAgent field name**: `skills_matcher_llm`
  - **Reads**: `parsed_data`
  - **Writes**: `skills_match`
  - **Output schema**:
    - `matched_skills: list[str]` — Skills present in both job description and resume
    - `missing_skills: list[str]` — Required skills absent from resume
    - `bonus_skills: list[str]` — Candidate skills not required but valuable
    - `match_score: float` — Score from 0.0 to 10.0 based on skills overlap
    - `verdict: Literal["strong_match", "partial_match", "weak_match"]` — Overall skills verdict
  - **Instruction summary**: You are a technical skills evaluator. Compare required skills vs candidate skills from `{parsed_data}`. Be objective and strict — missing a critical skill matters more than having bonus skills.

#### 2b. ExperienceEvaluatorAgent (inside parallel group)
  - **LlmAgent field name**: `experience_evaluator_llm`
  - **Reads**: `parsed_data`
  - **Writes**: `experience_eval`
  - **Output schema**:
    - `experience_score: float` — Score from 0.0 to 10.0
    - `meets_seniority: bool` — Whether candidate meets the required seniority level
    - `experience_gaps: list[str]` — Specific experience areas that are lacking
    - `experience_highlights: list[str]` — Strongest relevant experience points
    - `verdict: Literal["overqualified", "well_matched", "underqualified"]` — Experience fit
  - **Instruction summary**: You are an experience evaluator. Assess whether the candidate's years and quality of experience matches the seniority requirement in `{parsed_data}`. Consider domain relevance, not just total years.

#### 2c. CultureFitAgent (inside parallel group)
  - **LlmAgent field name**: `culture_fit_llm`
  - **Reads**: `parsed_data`
  - **Writes**: `culture_fit`
  - **Output schema**:
    - `fit_score: float` — Score from 0.0 to 10.0
    - `positive_signals: list[str]` — Resume elements suggesting good culture fit
    - `concern_signals: list[str]` — Elements that may indicate a mismatch
    - `communication_quality: Literal["excellent", "good", "average", "poor"]` — Assessed from how the resume is written
    - `verdict: Literal["likely_fit", "neutral", "likely_mismatch"]` — Culture fit verdict
  - **Instruction summary**: You are a culture fit assessor. From the resume writing style, career trajectory, and stated achievements in `{parsed_data}`, infer soft skills and culture fit indicators. Do not stereotype — base assessments on concrete signals only.

---

### 3. ScoringAgent
- **Type**: BaseAgent wrapping one LlmAgent
- **LlmAgent field name**: `scorer_llm`
- **Reads**: `skills_match`, `experience_eval`, `culture_fit`, `parsed_data`
- **Writes**: `score_result`
- **Output schema**:
  - `composite_score: float` — Weighted final score from 0.0 to 10.0
  - `skills_weight: float` — Weight applied to skills score (decided by agent based on role)
  - `experience_weight: float` — Weight applied to experience score
  - `culture_weight: float` — Weight applied to culture score
  - `recommendation: Literal["strong_hire", "hire", "maybe", "no_hire"]` — Final recommendation
  - `decision_rationale: str` — 2-3 sentence explanation of the recommendation
  - `shortlist: bool` — Whether to shortlist this candidate for interview
- **Instruction summary**: You are a hiring decision engine. Synthesize the three specialist evaluations from `{skills_match}`, `{experience_eval}`, and `{culture_fit}`. Weight scores intelligently based on the role seniority from `{parsed_data}` — senior roles should weight experience more, technical roles should weight skills more.

---

### 4. FeedbackAgent
- **Type**: BaseAgent wrapping one LlmAgent
- **LlmAgent field name**: `feedback_llm`
- **Reads**: `score_result`, `skills_match`, `experience_eval`, `parsed_data`
- **Writes**: `feedback`
- **Output schema**:
  - `recruiter_summary: str` — 3-4 sentence briefing for the recruiter
  - `candidate_feedback: str` — Constructive feedback paragraph for the candidate (professional, encouraging tone)
  - `interview_focus_areas: list[str]` — Topics the interviewer should probe if shortlisted
  - `development_suggestions: list[str]` — Skills or experience gaps candidate should address
  - `next_action: Literal["schedule_interview", "send_rejection", "hold_for_review"]` — Concrete next step
- **Instruction summary**: You are a feedback composer. Using `{score_result}`, `{skills_match}`, `{experience_eval}`, and `{parsed_data}`, write a recruiter briefing and candidate-facing feedback. Candidate feedback must be professional and constructive regardless of the decision.

---

## Pipeline

```
Sequential:
  IngestAgent
    → EvaluationCouncilAgent  (parallel: SkillsMatcherAgent + ExperienceEvaluatorAgent + CultureFitAgent)
      → ScoringAgent
        → FeedbackAgent
```

State flow:
```
job_description + resume_text  →  IngestAgent         →  parsed_data
parsed_data                    →  SkillsMatcherAgent   →  skills_match      ─┐
parsed_data                    →  ExperienceEvaluator  →  experience_eval   ─┤  (parallel)
parsed_data                    →  CultureFitAgent      →  culture_fit       ─┘
skills_match + experience_eval
  + culture_fit + parsed_data  →  ScoringAgent         →  score_result
score_result + skills_match
  + experience_eval
  + parsed_data                →  FeedbackAgent        →  feedback
```

---

## Test Input (for main.py)

```python
initial_state = {
    "job_description": (
        "We are looking for a Senior Python Backend Engineer to join our platform team. "
        "The ideal candidate has 5+ years of experience building scalable APIs with FastAPI or Django, "
        "strong knowledge of PostgreSQL and Redis, experience with Docker and Kubernetes, "
        "and familiarity with cloud platforms (GCP preferred). "
        "You will lead technical design discussions and mentor junior engineers. "
        "Strong communication skills and a collaborative mindset are essential."
    ),
    "resume_text": (
        "Jane Smith | jane@email.com\n"
        "Summary: Backend engineer with 4 years of experience in Python web development.\n"
        "Skills: Python, FastAPI, PostgreSQL, Redis, Docker, AWS, REST APIs, Git\n"
        "Experience:\n"
        "  - Backend Engineer @ TechCorp (2021-2024): Built microservices with FastAPI, "
        "maintained PostgreSQL databases, deployed on AWS ECS.\n"
        "  - Junior Developer @ StartupXYZ (2020-2021): Django REST framework, MySQL.\n"
        "Education: B.Tech Computer Science, 2020.\n"
        "Projects: Open source contributor to FastAPI ecosystem. Built a Redis-based "
        "rate limiter library with 200+ GitHub stars."
    ),
}
```
