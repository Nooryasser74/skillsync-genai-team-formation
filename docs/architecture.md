# SkillSync Architecture

SkillSync is a standalone Generative AI system for explainable team formation.  
It transforms candidate data and project descriptions into structured profiles, semantic embeddings, team recommendations, evaluation outputs, and explanation reports.

The system combines:

- LLM-based information extraction,
- semantic embeddings,
- cosine similarity,
- diversity-aware team construction,
- deterministic evaluation,
- and Streamlit-based interaction.

---

## 1. System Overview

SkillSync is designed as an AI-assisted decision-support system for forming balanced technical teams.

The system receives two main inputs:

1. **Candidate data**  
   Candidate profiles uploaded as CSV rows. These may contain skills, tools, experience, availability, developer type, collaboration signals, and personality-style text.

2. **Project description**  
   A natural-language description of the project, including expected technologies, required roles, team size, and working-hour constraints.

The output is a set of recommended teams with:

- selected candidate IDs,
- technical coverage,
- missing skills/tools,
- missing roles,
- candidate-project fit scores,
- internal diversity metrics,
- availability information,
- and an LLM-generated explanation report.

---

## 2. High-Level Architecture

```text
Candidate CSV
    │
    ▼
LLM Candidate Profile Extraction
    │
    ├── candidate_profiles_with_evidence.json
    ├── candidate_profiles.json
    └── candidate_inputs.json
    │
    ▼
Candidate Embedding Layer
    │
    ├── candidate_embeddings.npy
    ├── candidate_ids.json
    └── candidate_dissimilarity.npy
    │
    ▼
──────────────────────────────────────────────

Project Description
    │
    ▼
LLM Project Requirement Extraction
    │
    └── project_requirements.json
    │
    ▼
Project Embedding Layer
    │
    ├── project_embeddings.npy
    ├── project_ids.json
    └── candidate_project_similarity.json
    │
    ▼
──────────────────────────────────────────────

Candidate Profiles
+ Candidate Similarity
+ Project Similarity
+ Project Requirements
    │
    ▼
Multi-Objective Team Builder
    │
    ├── teams_ui.json
    ├── teams_summary.json
    └── teams_overview.json
    │
    ▼
Evaluation Dashboard
    │
    ├── candidate_extraction_eval.json
    ├── project_extraction_eval.json
    ├── teams_evaluation_summary.json
    └── latency_log.jsonl
    │
    ▼
LLM Explanation Report
    │
    └── team_report.json
```

---

## 3. Application Flow

The complete pipeline follows these stages:

1. Upload candidate CSV data.
2. Extract structured candidate profiles using an LLM.
3. Generate candidate embeddings using `all-MiniLM-L6-v2`.
4. Compute candidate-candidate dissimilarity.
5. Enter a natural-language project description.
6. Extract project requirements using an LLM.
7. Generate a project embedding.
8. Compute candidate-project similarity.
9. Build teams using a multi-objective heuristic.
10. Evaluate extraction quality and team quality.
11. Generate a grounded LLM explanation report.

---

## 4. Main Components

### 4.1 Streamlit UI

The Streamlit interface is the main entry point for the system.

Main file:

```text
src/modules/ui/mini_app.py
```

The UI provides five main tabs:

- `Candidate Pool`
- `Run Pipeline`
- `Evaluation`
- `Explanation Report`
- `Raw JSON`

Responsibilities:

- upload candidate CSV files,
- configure API settings,
- run candidate extraction,
- run project extraction,
- trigger embedding generation,
- build teams,
- display team summaries,
- run evaluation modules,
- inspect raw JSON artifacts,
- generate explanation reports.

---

### 4.2 JSON Extraction Layer

The JSON extraction layer converts noisy candidate data and free-text project descriptions into structured JSON.

Main files:

```text
src/modules/json_extraction/extractor.py
src/modules/json_extraction/proj_extractor.py
```

#### Candidate Extraction

Candidate extraction takes one CSV row and builds a normalized profile text.  
The LLM then extracts a strict JSON schema containing:

- candidate ID,
- weekly availability,
- developer type,
- work experience,
- industry,
- technical skills,
- tools,
- Belbin-style team role,
- communication style,
- conflict style,
- leadership preference,
- deadline discipline,
- learning orientation,
- knowledge-sharing behavior,
- evidence spans.

The prompt uses strict anti-hallucination constraints:

- return valid JSON only,
- do not invent unsupported facts,
- use `Unknown` for missing values,
- copy metadata fields exactly where required,
- extract skills/tools only when grounded in the input,
- provide short evidence excerpts for behavioral labels.

Outputs:

```text
data/artifacts/extraction/candidate_profiles_with_evidence.json
data/artifacts/extraction/candidate_profiles.json
data/artifacts/extraction/candidate_inputs.json
```

#### Project Requirement Extraction

Project extraction converts a natural-language project description into:

- required skills,
- required tools,
- required roles.

The project extractor uses role whitelisting and technology normalization so that downstream matching remains consistent.

Output:

```text
data/artifacts/project/project_requirements.json
```

---

### 4.3 Candidate Embedding Layer

The candidate embedding layer converts structured candidate profiles into semantic vectors.

Main files:

```text
src/modules/candidate_embeddings/text_encoder.py
src/modules/candidate_embeddings/embedding_service.py
src/modules/candidate_embeddings/similarity.py
src/modules/candidate_embeddings/repository.py
```

Process:

1. Load extracted candidate profiles.
2. Convert each profile into a compact text representation.
3. Encode the text using `all-MiniLM-L6-v2`.
4. Save candidate embeddings.
5. Compute candidate-candidate cosine dissimilarity.

The candidate text representation includes:

- developer type,
- skills,
- tools,
- Belbin role,
- communication style.

Outputs:

```text
data/artifacts/candidate_embeddings.npy
data/artifacts/candidate_ids.json
data/artifacts/candidate_dissimilarity.npy
```

The dissimilarity matrix is used by the team builder to avoid selecting overly similar candidates.

---

### 4.4 Project Embedding Layer

The project embedding layer converts extracted project requirements into semantic vectors.

Main files:

```text
src/modules/project_embeddings/embeddings.py
src/modules/project_embeddings/generate_embeddings.py
src/modules/project_embeddings/compute_similarity.py
src/modules/project_embeddings/store_embeddings.py
src/modules/project_embeddings/store_similarity.py
```

Process:

1. Load `project_requirements.json`.
2. Convert required roles, skills, and tools into text.
3. Encode the project text using `all-MiniLM-L6-v2`.
4. Compute cosine similarity between candidate embeddings and the project embedding.
5. Save the candidate-project similarity matrix.

Outputs:

```text
data/artifacts/project_embeddings.npy
data/artifacts/project_ids.json
data/artifacts/candidate_project_similarity.npy
data/artifacts/candidate_project_similarity.json
```

The candidate-project similarity score is used as the project-fit component in team construction.

---

### 4.5 Team Builder

The team builder constructs teams using a deterministic multi-objective heuristic.

Main file:

```text
src/modules/team_builder/builder.py
```

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

- `project_fit` measures semantic similarity between candidate and project,
- `coverage_gain` rewards candidates who add missing required skills/tools,
- `diversity` rewards candidates who are less similar to current team members,
- `role_gain` rewards candidates who help satisfy missing project roles,
- `overlap_redundancy` penalizes duplicated skills/tools.

Supported modes:

- greedy team construction for one high-fit team,
- round-robin construction for multiple balanced teams,
- optional LLM reranking among top candidates,
- UI-ready team summaries.

Outputs:

```text
data/artifacts/teams_ui.json
data/artifacts/teams_summary.json
data/artifacts/teams_overview.json
```

---

### 4.6 Evaluation Layer

The evaluation layer checks the quality of extracted data and generated teams.

Main files:

```text
src/modules/evaluators/profile_ex_eval.py
src/modules/evaluators/project_ex_eval.py
src/modules/evaluators/team_eval.py
src/modules/evaluators/latency_logger.py
```

#### Candidate Extraction Evaluation

Checks:

- schema validity,
- invalid categorical labels,
- evidence consistency,
- grounded skills/tools,
- missing evidence,
- unsupported evidence.

Output:

```text
data/artifacts/evaluation/candidate_extraction_eval.json
```

#### Project Requirement Evaluation

Checks:

- schema completeness,
- list type correctness,
- duplicate outputs,
- whether extracted skills/tools are supported by the original project description,
- alias-aware grounding for normalized technologies.

Output:

```text
data/artifacts/evaluation/project_extraction_eval.json
```

#### Team Evaluation

Checks:

- coverage percentage,
- covered skills/tools,
- missing skills/tools,
- missing roles,
- average internal similarity,
- member contribution distribution,
- availability feasibility.

Output:

```text
data/artifacts/evaluation/teams_evaluation_summary.json
```

#### Latency Logging

Tracks runtime for LLM-related stages:

- candidate extraction,
- project extraction,
- explanation generation.

Output:

```text
data/artifacts/evaluation/latency_log.jsonl
```

---

### 4.7 Explanation Report Layer

The explanation layer generates a human-readable report for selected teams.

Main module:

```text
src/modules/EXPLANATION/
```

The report is generated by an LLM but grounded in existing artifacts:

- candidate profiles,
- candidate inputs,
- project requirements,
- team summaries,
- evaluation outputs.

The LLM is used for explanation, not for uncontrolled decision-making.

The report explains:

- why the team fits the project,
- what requirements are covered,
- what gaps or risks remain,
- how team members complement each other,
- and why the recommendation is reasonable.

Output:

```text
data/artifacts/explanation/team_report.json
```

---

## 5. Data and Artifact Storage

SkillSync stores intermediate outputs in:

```text
data/artifacts/
```

Typical artifact structure:

```text
data/artifacts/
│
├── extraction/
│   ├── candidate_profiles.json
│   ├── candidate_profiles_with_evidence.json
│   └── candidate_inputs.json
│
├── project/
│   ├── project_description.txt
│   └── project_requirements.json
│
├── evaluation/
│   ├── candidate_extraction_eval.json
│   ├── project_extraction_eval.json
│   ├── teams_evaluation_summary.json
│   └── latency_log.jsonl
│
├── explanation/
│   └── team_report.json
│
├── candidate_embeddings.npy
├── candidate_ids.json
├── candidate_dissimilarity.npy
├── project_embeddings.npy
├── project_ids.json
├── candidate_project_similarity.npy
└── candidate_project_similarity.json
```

Generated artifacts should not be committed to GitHub.

---

## 6. Interface Objects

Shared objects are defined in:

```text
src/shared/interfaces.py
```

The two main interface objects are:

- `CandidateProfile`
- `ProjectDescription`

These objects allow the UI, embedding modules, team builder, and evaluators to exchange data consistently.

---

## 7. Technology Stack

| Layer | Technology |
|---|---|
| UI | Streamlit |
| Language | Python |
| LLM API | OpenRouter / OpenAI-compatible API |
| LLM Model | GPT-4o-mini |
| Embeddings | SentenceTransformers |
| Embedding Model | all-MiniLM-L6-v2 |
| Similarity | Cosine similarity |
| Data Processing | pandas, NumPy |
| Evaluation | Custom schema, grounding, and team-quality evaluators |
| Storage | JSON, JSONL, NumPy `.npy` files |

---

## 8. Design Principles

SkillSync follows these design principles:

### 1. Human-in-the-loop decision support

The system recommends teams but does not replace human judgment.

### 2. Grounded LLM usage

LLMs are used for extraction and explanation, with strict prompts and artifact grounding.

### 3. Deterministic team construction

The core team-building logic is algorithmic and inspectable.

### 4. Transparency

Intermediate JSON files, similarity matrices, team summaries, and evaluation outputs are saved for inspection.

### 5. Modularity

Each major pipeline step is separated into its own module.

### 6. Reproducibility

Intermediate artifacts are stored so that individual stages can be inspected, rerun, or evaluated.

---

## 9. Running the System

From the project root:

```bash
pip install -r requirements.txt
streamlit run src/modules/ui/mini_app.py
```

The main workflow is:

1. Upload candidate CSV data in `Candidate Pool`.
2. Run candidate extraction.
3. Generate candidate embeddings.
4. Enter a project description in `Run Pipeline`.
5. Extract project requirements.
6. Generate project embeddings and candidate-project similarity.
7. Build teams.
8. Evaluate outputs in `Evaluation`.
9. Generate a report in `Explanation Report`.

---

## 10. Limitations

Current limitations:

- candidate extraction is sequential and can be slow for large datasets,
- team scoring uses fixed weights,
- role matching may require stronger validation in production,
- real-world validation with actual team outcomes was not performed,
- LLM extraction quality depends on the quality of input text,
- generated artifacts are local file-based rather than database-backed.

---

## 11. Future Extensions

Possible future improvements:

- asynchronous or batched LLM extraction,
- Pydantic or JSON Schema validation for all LLM outputs,
- adaptive team-scoring weights based on feedback,
- stronger privacy handling for real candidate data,
- database-backed artifact storage,
- multi-project candidate allocation,
- user feedback loop for improving recommendations,
- real-world evaluation against actual team performance outcomes.
