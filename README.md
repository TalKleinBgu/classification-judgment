# Classification Judgment — Hebrew Legal Text Classifier

Automatic classification of paragraphs from Israeli court verdicts into semantic categories, using fine-tuned Hebrew BERT models and GPT-4-based few-shot approaches.

---

## Overview

Israeli court judgments follow a predictable structure but vary significantly in formatting and language. This project builds and evaluates a pipeline that:

1. Extracts paragraphs from Hebrew DOCX verdict files
2. Identifies section boundaries using document formatting cues (bold, Miriam font)
3. Classifies each paragraph into one of six legal categories
4. Compares the performance of fine-tuned BERT models against GPT-4 few-shot classification strategies

Two case domains are covered: **drug offenses** and **weapons offenses**.

---

## Classification Categories

| Label | Hebrew | Description |
|-------|--------|-------------|
| 0 | עובדות המקרה | Case Facts — background and factual descriptions |
| 1 | ראיות לעונש | Sentencing Evidence — evidence relevant to sentencing |
| 2 | טיעוני הצדדים | Parties' Arguments — claims by prosecution and defense |
| 3 | תסקיר שירות מבחן | Probation Service Report |
| 4 | דיון והכרעה | Discussion & Ruling — judicial analysis and decision |
| 5 | מידע אחר | Other Information — miscellaneous content |

---

## Project Structure

```
classification-judgment/
├── config/
│   └── config.yaml                  # Paths, case types, API keys
├── src/
│   ├── split_docx_paragraph.py      # DOCX → CSV paragraph extractor
│   ├── fine_tuning.ipynb            # Fine-tune AlephBERT & DictaBERT
│   ├── with_titles.ipynb            # GPT classification with section titles
│   ├── no_titles.ipynb              # GPT classification without titles
│   ├── whole_verdict.ipynb          # GPT classification of full verdicts
│   └── cheeck_presentage.ipynb      # Header detection & section analysis
├── cases_docx/
│   ├── drugs/                       # Raw DOCX verdict files (drug cases)
│   ├── weapon/                      # Raw DOCX verdict files (weapon cases)
│   └── no_titles/                   # Verdicts without section title headers
├── cases_csv/
│   ├── drugs/                       # Preprocessed paragraph CSVs (drugs)
│   ├── weapon/                      # Preprocessed paragraph CSVs (weapons)
│   └── no_titles/                   # Preprocessed paragraph CSVs (no titles)
├── gt/
│   ├── drugs_combined.csv           # Labeled data — drugs (1,845 paragraphs)
│   ├── weapon_combined.csv          # Labeled data — weapons (1,489 paragraphs)
│   ├── no_titles_combined.csv       # Labeled data — no-titles (937 paragraphs)
│   └── all_combined.csv             # Combined labeled dataset (4,271 paragraphs)
├── results/
│   ├── fine_tuning/
│   │   ├── finetuned_alephbert/best/   # Saved AlephBERT checkpoint
│   │   ├── finetuned_dicta/best/       # Saved DictaBERT checkpoint
│   │   ├── test_metrics_summary.csv    # Evaluation metrics on test set
│   │   └── all_predictions.csv         # Per-paragraph model predictions
│   ├── drugs/                          # GPT results for drug cases
│   ├── weapon/                         # GPT results for weapon cases
│   └── no_titles/                      # GPT results for no-titles cases
└── CSV_OUTPUT/                         # Intermediate preprocessing outputs
```

---

## Installation

```bash
pip install pandas numpy transformers torch scikit-learn
pip install python-docx pyyaml openai datasets
pip install matplotlib seaborn arabic-reshaper python-bidi
pip install pywin32   # Windows only — for .doc → .docx conversion
```

> Requires Python 3.8+ and a CUDA-capable GPU for fine-tuning (recommended).

---

## Configuration

Edit [`config/config.yaml`](config/config.yaml) before running:

```yaml
TYPES: ['drugs', 'weapon']          # Case domains to process
DOCX_PATH: cases_docx/{type}        # Input DOCX directory
CSV_PATH: cases_csv/{type}          # Output CSV directory
OPENAI_API_KEY: <YOUR_API_KEY>      # OpenAI API key for GPT experiments
RESULTS_PATH: results               # Results output directory
GT_PATH: gt                         # Ground truth directory
```

---

## Pipeline

### Step 1 — Extract Paragraphs from DOCX

```bash
python src/split_docx_paragraph.py
```

Reads verdict files from `cases_docx/{type}/`, detects section boundaries using bold/Miriam-font formatting, strips metadata headers, and writes one CSV per verdict to `cases_csv/{type}/`.

Each output row contains:

| Column | Description |
|--------|-------------|
| `verdict` | Source file name |
| `text` | Paragraph text |
| `part` | Detected section title |

**Notes:**
- Handles Hebrew complex scripts via XML-level `w:bCs` inspection
- Automatically converts `.doc` to `.docx` on Windows (requires pywin32)
- Removes duplicate paragraphs and skips quoted-only content

### Step 2 — Ground Truth Preparation

Manually labeled CSVs must be placed in `gt/`. Pre-labeled files are included for drugs, weapons, and no-titles cases. The combined dataset totals **4,271 labeled paragraphs**.

### Step 3 — Fine-tune BERT Models

Open and run [`src/fine_tuning.ipynb`](src/fine_tuning.ipynb).

- **Models:** [AlephBERT](https://huggingface.co/onlplab/alephbert-base) (general Hebrew) and [DictaBERT](https://huggingface.co/dicta-il/dictabert) (legal Hebrew)
- **Split:** 70% train / 15% validation / 15% test
- **Architecture:** `BertForSequenceClassification` with 6 output classes
- **Output:** Best checkpoints saved to `results/fine_tuning/finetuned_*/best/`

Both models achieve approximately **~91% accuracy** on the held-out test set.

### Step 4 — GPT-based Classification

Three few-shot classification strategies using GPT-4 mini:

| Notebook | Strategy | Input |
|----------|----------|-------|
| [`with_titles.ipynb`](src/with_titles.ipynb) | Title + Text | Section header and paragraph text |
| [`no_titles.ipynb`](src/no_titles.ipynb) | Text Only | Paragraph text alone |
| [`whole_verdict.ipynb`](src/whole_verdict.ipynb) | Full Verdict | Entire document at once |

Results are saved to `results/{type}/` as CSVs with predictions from all strategies.

### Step 5 — Header Detection & Analysis

Run [`src/cheeck_presentage.ipynb`](src/cheeck_presentage.ipynb) to detect the presence of the three main verdict sections (facts / arguments / ruling) across the corpus and generate distribution statistics.

---

## Models

### Fine-tuned BERT

| Model | Base | Domain |
|-------|------|--------|
| AlephBERT | `onlplab/alephbert-base` | General Hebrew |
| DictaBERT | `dicta-il/dictabert` | Hebrew Legal Text |

Both are fine-tuned with `BertForSequenceClassification` (12 layers, 768 hidden size, 12 attention heads).

### GPT-4 Mini (Few-Shot)

Hebrew system prompts guide the model to assign one of the six categories per paragraph. The `with_titles` strategy provides the detected section header as additional context.

---

## Evaluation Metrics

- Accuracy, Precision, Recall, F1-score (macro & micro)
- Per-class classification report
- Confusion matrices

---


## Data

All verdict documents are real Israeli court judgments in Hebrew. Ground-truth labels were assigned manually. The dataset covers:

| Domain | Paragraphs |
|--------|-----------|
| Drugs | 1,845 |
| Weapons | 1,489 |
| No Titles | 937 |
| **Total** | **4,271** |

