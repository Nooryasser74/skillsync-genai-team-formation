# Development Guide

This guide explains how to set up, run, and extend the SkillSync project.

SkillSync is a standalone Generative AI team-formation system that combines LLM-based information extraction, semantic embeddings, similarity scoring, deterministic team construction, evaluation utilities, and a Streamlit interface.

---

## 1. Prerequisites

Before running the project, make sure you have:

- Python 3.8+
- Git
- pip
- An OpenRouter API key or OpenAI-compatible API key for LLM-based extraction and explanation

---

## 2. Installation

Clone the repository:

```bash
git clone https://github.com/Nooryasser74/skillsync-genai-team-formation.git
cd skillsync-genai-team-formation
```

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it.

On Windows:

```bash
.venv\Scripts\activate
```

On macOS/Linux:

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

---

## 3. Environment Variables

The LLM extraction and explanation modules require an API key.

Create a `.env` file in the project root:

```bash
OPENROUTER_API_KEY=your_api_key_here
```

Alternatively, the API key can be entered directly in the Streamlit sidebar.

Important: never commit `.env` to GitHub.

The `.gitignore` file should include:

```gitignore
.env
.venv/
venv/
__pycache__/
*.pyc
data/artifacts/
*.npy
*.jsonl
```

---

## 4. Running the Application

Start the Streamlit interface:

```bash
streamlit run src/modules/ui/mini_app.py
```

The application provides the following tabs:

- `Candidate Pool`
- `Run Pipeline`
- `Evaluation`
- `Explanation Report`
- `Raw JSON`

---

## 5. End-to-End Workflow

### Step 1: Upload Candidate Data

Go to the `Candidate Pool` tab and upload a candidate CSV file.

The candidate data should contain profile information such as:

- candidate ID,
- developer type,
- skills,
- tools,
- experience,
- availability,
- collaboration signals,
- personality or behavioral text.

---

### Step 2: Run Candidate Extraction

The system converts each CSV row into a structured JSON profile using an LLM.

Generated files:

```text
data/artifacts/extraction/candidate_profiles_with_evidence.json
data/artifacts/extraction/candidate_profiles.json
data/artifacts/extraction/candidate_inputs.json
```

The version with evidence is used for inspection and evaluation.  
The clean version is used for embeddings and team construction.

---

### Step 3: Generate Candidate Embeddings

After candidate extraction, the system automatically creates candidate embeddings using:

```text
SentenceTransformer: all-MiniLM-L6-v2
```

Generated files:

```text
data/artifacts/candidate_embeddings.npy
data/artifacts/candidate_ids.json
data/artifacts/candidate_dissimilarity.npy
```

The dissimilarity matrix is used to support diversity-aware team formation.

---

### Step 4: Extract Project Requirements

Go to the `Run Pipeline` tab and enter a natural-language project description.

The system extracts:

- required skills,
- required tools,
- required roles,
- team size,
- project working hours,
- required hours per person.

Generated file:

```text
data/artifacts/project/project_requirements.json
```

---

### Step 5: Generate Project Embeddings and Similarity

The extracted project requirements are embedded into the same vector space as candidate profiles.

The system then computes candidate-project similarity.

Generated files:

```text
data/artifacts/project_embeddings.npy
data/artifacts/project_ids.json
data/artifacts/candidate_project_similarity.npy
data/artifacts/candidate_project_similarity.json
```

---

### Step 6: Build Teams

The team builder selects candidates using a multi-objective heuristic.

The scoring function combines:

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
- `role_gain` rewards candidates who help satisfy required roles,
- `overlap_redundancy` penalizes duplicated skills/tools.

Generated files:

```text
data/artifacts/teams_ui.json
data/artifacts/teams_summary.json
data/artifacts/teams_overview.json
```

---

### Step 7: Evaluate the Pipeline

Go to the `Evaluation` tab.

The evaluation layer checks:

#### Candidate Extraction

- schema validity,
- invalid categorical labels,
- evidence consistency,
- grounded skills/tools,
- missing or unsupported evidence.

#### Project Requirement Extraction

- schema completeness,
- list type correctness,
- duplicate items,
- whether extracted skills/tools are supported by the original project description.

#### Team Quality

- coverage percentage,
- missing skills/tools,
- missing roles,
- internal similarity,
- contribution distribution,
- availability feasibility.

Generated files:

```text
data/artifacts/evaluation/candidate_extraction_eval.json
data/artifacts/evaluation/project_extraction_eval.json
data/artifacts/evaluation/teams_evaluation_summary.json
data/artifacts/evaluation/latency_log.jsonl
```

---

### Step 8: Generate Explanation Report

Go to the `Explanation Report` tab.

The LLM generates a grounded report based only on the extracted artifacts and generated teams.

The report explains:

- why a team fits the project,
- which requirements are covered,
- which risks or gaps remain,
- how team members complement each other.

Generated file:

```text
data/artifacts/explanation/team_report.json
```

---

## 6. Project Structure

```text
skillsync-genai-team-formation/
│
├── data/
│   ├── sample/
│   └── artifacts/
│
├── docs/
│   ├── architecture.md
│   ├── development_guide.md
│   ├── SkillSync_Final_Report_Nooreldin_Lasheen.pdf
│   └── SkillSync_Project_Plan_Nooreldin_Lasheen.pdf
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
├── main.py
├── config.yaml
├── requirements.txt
├── README.md
└── .gitignore
```

---

## 7. Main Modules

### `json_extraction`

Handles LLM-based structured extraction.

Main responsibilities:

- candidate profile extraction,
- project requirement extraction,
- strict JSON schema generation,
- evidence-grounded behavioral extraction,
- anti-hallucination prompting.

Important files:

```text
src/modules/json_extraction/extractor.py
src/modules/json_extraction/proj_extractor.py
```

---

### `candidate_embeddings`

Converts extracted candidate profiles into semantic embeddings.

Main responsibilities:

- profile-to-text conversion,
- skill/tool cleaning,
- candidate embedding generation,
- candidate-candidate dissimilarity calculation.

Important files:

```text
src/modules/candidate_embeddings/text_encoder.py
src/modules/candidate_embeddings/embedding_service.py
src/modules/candidate_embeddings/similarity.py
```

---

### `project_embeddings`

Converts extracted project requirements into semantic embeddings.

Main responsibilities:

- project-to-text conversion,
- project embedding generation,
- candidate-project similarity computation,
- storage of similarity matrices.

Important files:

```text
src/modules/project_embeddings/embeddings.py
src/modules/project_embeddings/generate_embeddings.py
src/modules/project_embeddings/compute_similarity.py
```

---

### `team_builder`

Constructs teams using deterministic scoring.

Main responsibilities:

- project-fit scoring,
- diversity scoring,
- coverage gain calculation,
- role gain calculation,
- redundancy penalty,
- greedy and round-robin team construction,
- UI-ready team summaries.

Important file:

```text
src/modules/team_builder/builder.py
```

---

### `evaluators`

Contains local evaluation utilities.

Main responsibilities:

- candidate extraction evaluation,
- project extraction evaluation,
- team quality evaluation,
- latency logging.

Important files:

```text
src/modules/evaluators/profile_ex_eval.py
src/modules/evaluators/project_ex_eval.py
src/modules/evaluators/team_eval.py
src/modules/evaluators/latency_logger.py
```

---

### `ui`

Contains the Streamlit application.

Important file:

```text
src/modules/ui/mini_app.py
```

---

## 8. Development Notes

### Generated Artifacts

Most pipeline outputs are written to:

```text
data/artifacts/
```

This folder should not be committed to GitHub because it contains generated outputs, embeddings, similarity matrices, evaluation logs, and possibly intermediate data.

---

### API Keys

Never hardcode API keys in source code.

Use:

```text
OPENROUTER_API_KEY
```

or enter the key through the Streamlit sidebar.

---

### Data Privacy

Candidate data may contain personal or sensitive information. For public GitHub use, only include sample or anonymized data.

Avoid committing:

- private candidate data,
- API keys,
- generated artifacts,
- `.env` files,
- large binary files.

---

## 9. Code Quality Guidelines

Recommended practices:

- Use clear function names.
- Keep modules separated by responsibility.
- Use type hints where possible.
- Add docstrings for important functions.
- Keep generated files out of Git.
- Prefer logging over print statements for production-style code.
- Validate LLM outputs before downstream use.
- Keep prompts deterministic for extraction tasks.

---

## 10. Common Issues

### Streamlit cannot import modules

Run the app from the project root:

```bash
streamlit run src/modules/ui/mini_app.py
```

Do not run it from inside `src/`.

---

### Missing API key

Set:

```bash
OPENROUTER_API_KEY=your_api_key_here
```

or paste the key into the Streamlit sidebar.

---

### Candidate-project similarity not generated

Make sure both candidate embeddings and project requirements exist:

```text
data/artifacts/candidate_embeddings.npy
data/artifacts/candidate_ids.json
data/artifacts/project/project_requirements.json
```

Then rerun project extraction or project embedding generation.

---

### Team builder returns no teams

Check:

- enough candidates were extracted,
- team size is not too large,
- candidate-project similarity exists,
- candidate IDs match between profiles and embeddings,
- required number of teams is feasible.

---

## 11. Future Development

Planned improvements:

- Add Pydantic or JSON Schema validation for all LLM outputs.
- Add asynchronous or batched LLM extraction.
- Add adaptive scoring weights based on user feedback.
- Improve role matching using hybrid explicit-role and embedding-based matching.
- Add unit tests for extraction, embeddings, similarity, and team-building modules.
- Add support for multiple simultaneous projects.
- Add stronger privacy handling for production use.
- Add real-world evaluation with actual team outcomes.

---

## 12. Maintainer

**Nooreldin Lasheen**  
MSc Data Science, TU Wien  
GitHub: `https://github.com/Nooryasser74`
