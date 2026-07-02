# SkillSync: Generative AI System for Explainable Team Formation

SkillSync is an end-to-end Generative AI prototype for forming balanced technical teams from candidate profiles and natural-language project descriptions.

The system combines LLM-based information extraction, semantic embeddings, candidate-project similarity scoring, diversity-aware team construction, automatic evaluation, and grounded explanation reports. It is designed as a decision-support tool: the system recommends teams and explains the reasoning, while the final decision remains under human control.

---

## Overview

Manual team formation is often slow, subjective, and based on incomplete information. In university projects, hackathons, research groups, and engineering organizations, teams are frequently assembled without a clear view of each member’s skills, tools, experience, availability, and collaboration style.

This can lead to:

- missing technical competencies,
- duplicated skill sets,
- unbalanced workload distribution,
- weak role coverage,
- hidden collaboration risks,
- and limited transparency in why a team was formed.

SkillSync addresses this problem by converting noisy candidate data and free-text project descriptions into structured, machine-readable profiles. These profiles are embedded into a shared semantic space and used by a deterministic team-building algorithm that balances project fit, skill coverage, role coverage, diversity, and redundancy.

---

## Key Features

- LLM-based candidate profile extraction from CSV data
- Strict JSON schema generation with anti-hallucination constraints
- Evidence-grounded extraction for behavioral and collaboration attributes
- Project requirement extraction from natural-language descriptions
- Semantic embeddings using `all-MiniLM-L6-v2`
- Candidate-project cosine similarity scoring
- Candidate-candidate dissimilarity for diversity-aware team formation
- Multi-objective team-building heuristic
- Streamlit interface for running the full pipeline
- Evaluation dashboard for extraction quality, team quality, and latency
- LLM-generated explanation reports grounded in system artifacts
- Raw JSON inspection for transparency and debugging

---

## System Architecture

```text
Candidate CSV
    ↓
LLM Candidate Profile Extraction
    ↓
candidate_profiles_with_evidence.json
candidate_profiles.json
    ↓
Candidate Embeddings
    ↓
candidate_embeddings.npy
candidate_dissimilarity.npy

Project Description
    ↓
LLM Project Requirement Extraction
    ↓
project_requirements.json
    ↓
Project Embeddings
    ↓
candidate_project_similarity.json

Candidate Profiles + Similarity Scores
    ↓
Multi-Objective Team Builder
    ↓
Generated Teams
    ↓
Evaluation Dashboard
    ↓
LLM Explanation Report
```

---

## Technical Pipeline

### 1. Candidate Profile Extraction

SkillSync accepts candidate data in CSV format and converts each row into a structured JSON profile using an LLM.

Each extracted candidate profile includes:

- technical skills,
- tools,
- developer type,
- work experience,
- industry,
- weekly availability,
- Belbin-style team role,
- communication style,
- conflict style,
- leadership preference,
- deadline discipline,
- learning orientation,
- and knowledge-sharing behavior.

The extraction prompt uses strict rules:

- return valid JSON only,
- do not invent unsupported information,
- copy metadata fields exactly where required,
- extract technical skills and tools only when grounded in the input,
- use `Unknown` for missing or ambiguous information,
- attach short evidence spans for behavioral labels.

Main outputs:

```text
data/artifacts/extraction/candidate_profiles_with_evidence.json
data/artifacts/extraction/candidate_profiles.json
data/artifacts/extraction/candidate_inputs.json
```

---

### 2. Candidate Embeddings and Diversity Matrix

After extraction, each structured candidate profile is converted into a compact text representation containing role, skills, tools, Belbin role, and communication style.

The text is embedded using:

```text
SentenceTransformer: all-MiniLM-L6-v2
```

The system saves:

```text
candidate_embeddings.npy
candidate_ids.json
candidate_dissimilarity.npy
```

The candidate dissimilarity matrix is used later to avoid forming teams where all members are too similar.

---

### 3. Project Requirement Extraction

The user enters a free-text project description through the Streamlit interface.

The project extractor converts this description into a structured JSON object containing:

- required skills,
- required tools,
- required roles,
- team size,
- project working hours,
- required hours per person.

The extractor uses strict role whitelisting and technology normalization. For example:

```text
AWS → Amazon Web Services (AWS)
GCP → Google Cloud
PyTorch / Torch → Torch/PyTorch
Bash / Shell scripting → Bash/Shell (all shells)
```

Main output:

```text
data/artifacts/project/project_requirements.json
```

---

### 4. Candidate-Project Similarity

Project requirements are embedded into the same vector space as candidate profiles.

The system computes cosine similarity between:

```text
candidate embeddings × project embedding
```

This produces a candidate-project fit matrix used by the team builder.

Main outputs:

```text
candidate_project_similarity.json
candidate_project_similarity.npy
```

---

### 5. Multi-Objective Team Builder

The team builder selects candidates using a deterministic scoring function.

Each candidate is scored using:

```text
score =
    α × project_fit
  + γ × coverage_gain
  + β × diversity
  + ρ × role_gain
  - δ × overlap_redundancy
```

Where:

- `project_fit` measures candidate-project semantic similarity,
- `coverage_gain` rewards candidates who add missing required skills/tools,
- `diversity` rewards candidates who are less redundant with the current team,
- `role_gain` rewards candidates who help satisfy missing required roles,
- `overlap_redundancy` penalizes duplicated skills/tools.

The system supports:

- greedy construction for a single high-fit team,
- round-robin allocation for multiple balanced teams,
- optional LLM-assisted reranking among top candidates,
- UI-ready team summaries with missing skills, missing roles, coverage, fit, and diversity metrics.

---

### 6. Evaluation Layer

SkillSync includes evaluation modules for both extraction and team quality.

Candidate extraction evaluation checks:

- schema validity,
- invalid categorical labels,
- evidence consistency,
- grounded skills and tools,
- missing or unsupported evidence.

Project extraction evaluation checks:

- schema completeness,
- list type correctness,
- duplicate outputs,
- whether extracted skills/tools are supported by the project description,
- alias-aware grounding for normalized technologies.

Team evaluation checks:

- skill/tool coverage percentage,
- missing requirements,
- missing roles,
- internal similarity,
- contribution distribution across members,
- availability feasibility.

---

## Evaluation Results

The system was evaluated on a subset of 200 candidate profiles derived from the Stack Overflow Annual Developer Survey 2024.

| Metric | Result |
|---|---:|
| Candidate profiles evaluated | 200 |
| Original survey variables | 120+ |
| Selected features | 28 |
| Candidate schema validity rate | 1.00 |
| Invalid categorical labels | 0 |
| Average skills verbatim-in-text rate | 1.00 |
| Average tools verbatim-in-text rate | 1.00 |
| Evidence present: personality role | 1.00 |
| Evidence present: communication style | 1.00 |
| Evidence present: conflict style | 1.00 |
| Evidence present: leadership preference | 0.93 |
| Evidence present: knowledge sharing | 0.61 |

Project requirement extraction was evaluated on multiple project descriptions using precision, recall, F1-score, and accuracy.

| Project | Precision | Recall | F1-score | Accuracy |
|---|---:|---:|---:|---:|
| Customer Loyalty Platform | 0.750 | 0.500 | 0.600 | 0.429 |
| Smart Recommendation Marketplace | 1.000 | 0.833 | 0.909 | 0.833 |
| Virtual Events Platform | 0.857 | 1.000 | 0.923 | 0.857 |
| Employee Performance Dashboard | 1.000 | 0.667 | 0.800 | 0.667 |
| Recommendation-Based News App | 1.000 | 1.000 | 1.000 | 1.000 |

Latency observations:

| Stage | Latency |
|---|---:|
| Candidate extraction | 10.4–15.9s per candidate |
| Project extraction | 0.02–4.9s |
| Explanation generation | 17.2–46.7s |

---

## Streamlit Application

The project includes an interactive Streamlit application with the following tabs.

### Candidate Pool

- Upload candidate CSV data
- Select number of candidates to extract
- Run LLM-based candidate extraction
- Inspect extracted profiles
- Automatically generate candidate embeddings

### Run Pipeline

- Enter a project description
- Configure team size and working hours
- Extract project requirements
- Compute project embeddings
- Compute candidate-project similarity
- Build teams

### Evaluation

- Inspect candidate extraction quality
- Inspect project extraction quality
- Evaluate generated teams
- View latency logs

### Explanation Report

- Generate an LLM-based team explanation
- Summarize team strengths
- Identify risks and missing requirements
- Explain why specific candidates were selected

### Raw JSON

- Inspect all intermediate artifacts for transparency and debugging

---

## Tech Stack

| Layer | Tools |
|---|---|
| Language | Python |
| UI | Streamlit |
| LLM API | OpenRouter / OpenAI-compatible API |
| LLM model | GPT-4o-mini |
| Embeddings | SentenceTransformers |
| Embedding model | all-MiniLM-L6-v2 |
| Similarity | Cosine similarity |
| Data processing | pandas, NumPy |
| Evaluation | Custom schema, grounding, and team-quality evaluators |
| Storage | JSON, JSONL, NumPy `.npy` artifacts |

---

## Project Structure

```text
SkillSync/
│
├── data/
│   ├── sample/
│   └── artifacts/
│       ├── extraction/
│       ├── project/
│       ├── evaluation/
│       └── explanation/
│
├── src/
│   ├── modules/
│   │   ├── json_extraction/
│   │   ├── candidate_embeddings/
│   │   ├── project_embeddings/
│   │   ├── team_builder/
│   │   ├── evaluators/
│   │   ├── EXPLANATION/
│   │   └── ui/
│   │
│   └── shared/
│
├── requirements.txt
├── README.md
└── .gitignore
```

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/your-username/skillsync-generative-ai-team-formation.git
cd skillsync-generative-ai-team-formation
```

### 2. Create and activate a virtual environment

```bash
python -m venv .venv
```

On Windows:

```bash
.venv\Scripts\activate
```

On macOS/Linux:

```bash
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Set API key

SkillSync uses an OpenAI-compatible API through OpenRouter for LLM extraction and explanation.

Create a `.env` file or export the variable manually:

```bash
OPENROUTER_API_KEY=your_api_key_here
```

You can also paste the API key directly into the Streamlit sidebar.

### 5. Run the application

```bash
streamlit run src/modules/ui/mini_app.py
```

---

## Example Workflow

1. Open the Streamlit UI.
2. Go to `Candidate Pool`.
3. Upload candidate CSV data.
4. Run candidate extraction.
5. Candidate embeddings are generated automatically.
6. Go to `Run Pipeline`.
7. Enter a project description.
8. Extract project requirements.
9. Generate project embeddings and candidate-project similarity.
10. Build teams.
11. Inspect team coverage, missing skills, missing roles, fit, and diversity.
12. Go to `Evaluation` to validate the pipeline.
13. Go to `Explanation Report` to generate a grounded LLM report.

---

## Example Project Description

```text
We want to build a recommendation-based web application for personalized news.
The system should include a Python backend, a React frontend, a PostgreSQL database,
and machine learning models for ranking articles. We need developers who can work
on backend APIs, frontend UI, data pipelines, and ML-based recommendation logic.
```

The system extracts requirements such as:

```json
{
  "technical_requirements": {
    "skills": ["Python", "React", "PostgreSQL", "Machine Learning"],
    "tools": [],
    "roles": [
      "Developer, back-end",
      "Developer, front-end",
      "Data engineer",
      "Data scientist or machine learning specialist"
    ]
  }
}
```

---

## Design Philosophy

SkillSync is not designed to replace human decision-making.

Instead, it acts as an AI-assisted decision-support system:

- the LLM extracts and explains,
- embeddings provide semantic comparison,
- deterministic algorithms build teams,
- evaluation modules check quality,
- humans remain responsible for the final decision.

This separation reduces the risk of opaque automated decisions while still using Generative AI where it is useful: structuring information, normalizing ambiguous text, and generating understandable explanations.

---

## Limitations

- The current scoring function uses fixed weights rather than learned or adaptive weights.
- Role matching is partly semantic and may require additional validation in real-world settings.
- Candidate profiles are based on survey-style data, which may contain self-reporting bias.
- Real-world validation with actual team performance outcomes was not performed.
- LLM extraction quality depends on the clarity of the input data and project description.
- Candidate extraction can be slow for large datasets because profiles are processed sequentially.

---

## Future Improvements

- Add Pydantic or JSON Schema validation for all LLM outputs.
- Add asynchronous or batched LLM extraction.
- Add adaptive weighting based on user feedback.
- Improve role matching using hybrid explicit-role and embedding-based matching.
- Add real-world user studies to compare team recommendations against actual team outcomes.
- Add support for multiple simultaneous projects and constrained allocation.
- Improve privacy handling for production use.
- Add unit tests for extraction, embeddings, similarity, and team-building modules.

---

## My Contribution

This repository contains my standalone implementation and documentation version of SkillSync, developed in the context of the TU Wien Generative AI course.

My work focused on building and integrating the applied AI pipeline, including:

- LLM-based JSON extraction for candidate and project profiles,
- prompt design with grounding and anti-hallucination constraints,
- semantic embedding and similarity-based matching,
- team-building logic using fit, diversity, coverage, and role constraints,
- evaluation utilities for extraction and team quality,
- Streamlit integration for running the full workflow,
- documentation and portfolio-ready repository cleanup.

---

## Author

**Nooreldin Lasheen**  
MSc Data Science, TU Wien  
GitHub: `your-github-link`  
LinkedIn: `your-linkedin-link`
