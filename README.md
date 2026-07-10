# Where Experts Disagree, Models Fail: Detecting Implicit Legal Citations in French Court Decisions

Data and code for the benchmark and experiments in the paper. We release **1,015 (chunk, article) pairs** drawn from French first-instance civil decisions (Judilibre). A JuriBERT bi-encoder is trained on *explicit* French Civil Code citations and used to retrieve *implicit*-citation candidates; candidates are filtered adversarially with OpenAI o3, and three legal annotators (A1, A2, A3) label the survivors. The benchmark, every model prediction reported in the paper, the reference legal data, and a bounding-risk study are all shipped in `DATA/`. `CODE/` holds the full data pipeline (`pipeline/`), the per-experiment notebooks behind each paper section and appendix (`analysis/`), and a self-contained notebook that reproduces every table from the shipped data (`reproduce/`).

## Repository layout

```
implicit-legal-citations/
├── README.md · DATA.md · LICENSE · requirements.txt · .gitignore
├── CODE/
│   ├── pipeline/    01 → 07, the full pipeline that regenerates held-back artifacts
│   ├── analysis/    per-experiment notebooks behind each paper section/appendix
│   └── reproduce/   08_reproduce_results.ipynb, reproduces the paper's tables
├── DATA/
│   ├── inputs/      reference legal data
│   └── outputs/     benchmark + all predictions + bounding-risk study (see DATA.md)
└── docs/            annotation_guide_bilingual.pdf
```

## Code

Each stage is a self-contained Jupyter notebook. Run notebooks **from the repository root** so that the relative paths to `DATA/` (and to the held-back `artifacts/` folder) resolve.

| Notebook | Stage | What it does |
|---|---|---|
| `CODE/pipeline/01_data_collection.ipynb` | Collection | Pulls first-instance civil decisions from the Judilibre API into a raw corpus. |
| `CODE/pipeline/02_article_retrieval.ipynb` | Article retrieval | Fetches and normalises the French Civil Code from Légifrance; builds the reference article tables and old→new numbering equivalences. |
| `CODE/pipeline/03_dataset_creation.ipynb` | Dataset creation | Chunks decisions, mines explicit Civil Code citations, and assembles the (chunk, article) candidate pairs plus TF-IDF features. |
| `CODE/pipeline/04_model_training.ipynb` | Model training | Trains the JuriBERT bi-encoder on explicit citations and writes the encoder checkpoints. |
| `CODE/pipeline/05_inference.ipynb` | Inference | Embeds chunks and articles, runs FAISS retrieval, and produces the implicit-citation candidate set. |
| `CODE/pipeline/06_analysis.ipynb` | Analysis | Computes metrics, inter-annotator agreement, and the paper's figures and tables. |
| `CODE/pipeline/07_adversarial_filter_o3.ipynb` | Adversarial filter | Filters retrieved candidates with OpenAI o3 to keep only the hard, plausibly-implicit cases sent for annotation. |
| `CODE/reproduce/08_reproduce_results.ipynb` | Reproduce results | Rebuilds **all** of the paper's tables from the shipped `DATA/outputs/` alone — no pipeline run, no API access, no GPU. |

### Analysis notebooks (`CODE/analysis/`)

The per-experiment notebooks that produced the results in each section/appendix. They read the shipped `DATA/` (labels from `DATA/outputs/benchmark.csv`, predictions from `DATA/outputs/predictions/`) and write their heavy intermediate artifacts under a local `artifacts/` folder. Several require a GPU (fine-tuning) and the held-back data, so they are provided for transparency rather than one-click execution — the reported numbers themselves are reproduced end-to-end by `08_reproduce_results.ipynb`.

| Notebook | Paper part | What it does |
|---|---|---|
| `analysis/inter_annotator_agreement.ipynb` | §4 | Inter-annotator agreement (Cohen's κ), confusion matrix, A3 adjudication structure. |
| `analysis/supervised_encoders.ipynb` | §5.1 | Frozen-encoder classifiers and the stacking ensemble (grid search, nested CV). |
| `analysis/zeroshot_llm_evaluation.ipynb` | §5.2 | Zero-shot evaluation of the ten instruction-tuned LLMs. |
| `analysis/unsupervised_ranking.ipynb`, `unsupervised_ranking_full.ipynb` | §5.3 | Unsupervised top-k ranking via LLM consensus (average precision, recall at k). |
| `analysis/nli_baseline_finetuning.ipynb` | App. S | XLM-R-XNLI fine-tuned entailment baseline. |
| `analysis/bge_reranker_finetuning.ipynb` | App. T | BGE-reranker fine-tuned cross-encoder baseline. |
| `analysis/confound_analysis.ipynb` | App. O | Surface-confound controls (nested cluster-robust logistic regressions). |
| `analysis/sensitivity_analysis.ipynb` | App. R | Sensitivity of the unsupervised ranking to its weights. |
| `analysis/error_analysis.ipynb` | App. (error analysis) | Failure-mode annotation of 25 sampled false positives: reproducible sample draw and failure-mode table. |

## Reproducing the paper's results

```bash
pip install -r requirements.txt
jupyter notebook CODE/reproduce/08_reproduce_results.ipynb   # run from the repo root
```

`08_reproduce_results.ipynb` reads **only** the files already in `DATA/outputs/` (the benchmark, all model predictions, and the bounding-risk study). It needs no API access, no GPU, and no regeneration step, and reproduces the headline metrics, per-agreement error breakdowns, ensemble numbers, and baselines end to end.

To rerun the full pipeline from scratch instead, execute `CODE/pipeline/01`→`07` in order; these notebooks regenerate the held-back intermediate artifacts described in [DATA.md](DATA.md).

### Reproducibility notes

`08_reproduce_results.ipynb` prints a computed-vs-paper check for every table and reproduces all of the paper's data-backed numbers exactly: the nine supervised systems and the ensemble (F1, MCC, and the per-agreement odds ratios), the ten zero-shot LLMs, the retrieval-augmented few-shot table, the two fine-tuned baselines (XLM-R-NLI and BGE-reranker), the per-agreement false-positive analysis with FDR correction, the surface-confound controls, the calibration errors, and the bounding-risk study. Each decision threshold is the one used in the corresponding experiment (supervised thresholds tuned by nested CV; the XLM-R-NLI baseline by max-MCC over a coarse threshold grid; the BGE cross-encoder by its two-class argmax).

## Configuration

The pipeline notebooks (`01`, `02`, `07`) call external APIs. Provide credentials through environment variables — none are stored in the repository:

| Variable | Used by |
|---|---|
| `JUDILIBRE_API_KEY` | `01_data_collection` (Judilibre decisions) |
| `LEGIFRANCE_API_KEY` / `LEGIFRANCE_API_SECRET` | `02_article_retrieval` (Civil Code) |
| `OPENAI_API_KEY` | `07_adversarial_filter_o3` (o3 filter) |
| `HF_TOKEN` | model downloads in `04`/`05` and the fine-tuning / LLM notebooks in `CODE/analysis/` |

The reproduction notebook (`08`) uses none of these.

## Data availability

See [DATA.md](DATA.md) for a precise inventory of what is shipped versus what is held back and regenerated by the pipeline, and for the schema of `DATA/outputs/benchmark.csv`.

## License

Released under the MIT License (see [LICENSE](LICENSE)).
