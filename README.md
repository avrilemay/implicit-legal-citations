# Where Experts Disagree, Models Fail: Detecting Implicit Legal Citations in French Court Decisions

**Avrile Floro**¹, **Tamara Dhorasoo**², **Soline Pellez**², **Nils Holzenberger**¹
¹ Télécom Paris, Institut Polytechnique de Paris · ² Université Polytechnique Hauts-de-France

To be published at the Natural Legal Language Processing Workshop (NLLP 2026) · Preprint: [arXiv:2603.22973](https://arxiv.org/abs/2603.22973) · Model and data: [doi:10.5281/zenodo.21206799](https://doi.org/10.5281/zenodo.21206799)

Data and code for the benchmark and experiments in the paper. We release **1,015 (chunk, article) pairs** (829 decisions, 418 French Civil Code articles) drawn from 182,155 first-instance civil decisions published on Judilibre (December 2023 to July 2025). A JuriBERT bi-encoder trained on *explicit* Civil Code citations retrieves 40,566 *implicit*-citation candidates; OpenAI o3, used as a conservative adversarial filter, accepts 4,206 of them, from which the 1,015 pairs were selected for annotation (§3). Two legal annotators (A1, A2) label every pair and a third (A3) adjudicates their 339 disagreements (§4).

This repository ships the benchmark, the per-row predictions behind §5.1, §5.2 and §6, the reference legal data, and the bounding-risk study (`DATA/`). `CODE/` holds the full data pipeline (`pipeline/`), the per-experiment notebooks behind each section and appendix (`analysis/`), and a self-contained notebook that rebuilds the paper's main data-backed tables from the shipped data (`reproduce/`). The trained bi-encoder, the chunked training corpus and the raw decisions are too large for GitHub and are archived on [Zenodo](https://doi.org/10.5281/zenodo.21206799).

## Repository layout

```
implicit-legal-citations/
├── README.md · DATA.md · LICENSE · CITATION.cff
├── requirements.txt · requirements-reproduce.txt
├── CODE/
│   ├── pipeline/    01 → 07, the full pipeline that regenerates the large artifacts
│   ├── analysis/    per-experiment notebooks behind each paper section/appendix
│   └── reproduce/   08_reproduce_results.ipynb, rebuilds the paper's tables
├── DATA/
│   ├── inputs/      reference legal data
│   └── outputs/     benchmark + predictions + bounding-risk study (see DATA.md)
└── docs/            annotation_guide_bilingual.pdf
```

## Code

Each stage is a self-contained Jupyter notebook. Notebooks locate the repository root on their own, so they can be opened from anywhere inside the repository; paths to `DATA/` and to the local `artifacts/` folder resolve from the root.

| Notebook | Stage | What it does |
|---|---|---|
| `CODE/pipeline/01_data_collection.ipynb` | Collection | Pulls first-instance civil decisions from the Judilibre API into a raw corpus. |
| `CODE/pipeline/02_article_retrieval.ipynb` | Article retrieval | Fetches and normalises the French Civil Code from Légifrance; builds the reference article tables and old→new numbering equivalences (App. B). |
| `CODE/pipeline/03_dataset_creation.ipynb` | Dataset creation | Chunks decisions, mines explicit Civil Code citations, and assembles the (chunk, article) candidate pairs plus TF-IDF features (App. A). |
| `CODE/pipeline/04_model_training.ipynb` | Model training | Trains the JuriBERT bi-encoder on explicit citations and writes the encoder checkpoints (§3.2, App. A). |
| `CODE/pipeline/05_inference.ipynb` | Inference | Embeds chunks and articles, runs FAISS retrieval, and produces the implicit-citation candidate set (§3.3). |
| `CODE/pipeline/06_analysis.ipynb` | Descriptive statistics | Statistics of the corpus, training data and predictions; distribution of the benchmark across Civil Code books (Table 1). |
| `CODE/pipeline/07_adversarial_filter_o3.ipynb` | Adversarial filter | Filters retrieved candidates with OpenAI o3 to keep only the hard, plausibly-implicit cases sent for annotation (§3.4, App. C). |
| `CODE/reproduce/08_reproduce_results.ipynb` | Reproduce results | Rebuilds the paper's main data-backed tables from the shipped `DATA/outputs/` alone: no pipeline run, no API access, no GPU. |

### Analysis notebooks (`CODE/analysis/`)

The per-experiment notebooks that produced the results in each section and appendix. They read the shipped `DATA/` (labels from `DATA/outputs/benchmark.csv`, predictions from `DATA/outputs/predictions/`) and write their heavy intermediate artifacts under a local `artifacts/` folder. Several require a GPU (fine-tuning, LLM inference) and the large artifacts, so they are provided for transparency rather than one-click execution; the reported numbers are checked end to end by `08_reproduce_results.ipynb`.

| Notebook | Paper part | What it does |
|---|---|---|
| `analysis/inter_annotator_agreement.ipynb` | §4, App. F (Tables 2, 9) | Inter-annotator agreement (Cohen's κ), confusion matrix, A3 adjudication structure. Later cells hold exploratory runs that are not reported in the paper. |
| `analysis/supervised_encoders.ipynb` | §5.1, §6; App. G, H, P, Q, S | Frozen-encoder classifiers and the stacking ensemble (grid search, nested CV); false-positive and false-negative rates by agreement (Figure 3, Tables 25, 29) and calibration (Tables 26–27). |
| `analysis/zeroshot_llm_evaluation.ipynb` | §5.2; App. K, L | Zero-shot evaluation of the ten instruction-tuned LLMs. |
| `analysis/unsupervised_ranking.ipynb`, `unsupervised_ranking_full.ipynb` | §5.3; App. O (Tables 5, 6, 21–24) | Unsupervised top-k ranking via LLM consensus (average precision, precision and recall at k). |
| `analysis/sensitivity_analysis.ipynb` | App. N | Sensitivity of the unsupervised ranking to its weights. |
| `analysis/bge_reranker_finetuning.ipynb` | App. I (Table 15) | BGE-reranker-v2-m3 fine-tuned cross-encoder baseline. |
| `analysis/nli_baseline_finetuning.ipynb` | App. J (Table 16) | XLM-R-XNLI fine-tuned entailment baseline. |
| `analysis/confound_analysis.ipynb` | App. R (Table 28) | Surface-confound controls (nested cluster-robust logistic regressions). |
| `analysis/error_analysis.ipynb` | App. U (Table 30) | Failure-mode annotation of the ensemble's 66 false positives. |

The retrieval-augmented few-shot experiment (App. M, Table 20) is shipped as summary scores only (`DATA/outputs/predictions/fewshot/fewshot_results.csv`).

## Reproducing the paper's results

```bash
pip install -r requirements-reproduce.txt
jupyter nbconvert --to notebook --execute --inplace CODE/reproduce/08_reproduce_results.ipynb
# or open it interactively: jupyter notebook CODE/reproduce/08_reproduce_results.ipynb
```

`08_reproduce_results.ipynb` reads **only** the files already in `DATA/outputs/`. It needs no API access, no GPU and no regeneration step, runs in a few seconds, and prints a computed-vs-paper check for every number it rebuilds.

To rerun the full pipeline from scratch instead, install `requirements.txt` and execute `CODE/pipeline/01`→`07` in order; these notebooks regenerate the large intermediate artifacts described in [DATA.md](DATA.md). To skip collection and training (01–04), download the trained bi-encoder and the chunked corpus from [Zenodo](https://doi.org/10.5281/zenodo.21206799).

### Reproducibility notes

`08_reproduce_results.ipynb` reproduces exactly: the A1×A2 confusion matrix and A3 adjudication (Tables 2, 9); the nine supervised systems and the ensemble (Tables 3, 11, 12, 14); the ten zero-shot LLMs (Tables 4, 17); the retrieval-augmented few-shot scores (Table 20); the two fine-tuned baselines (Tables 15, 16); false positives by agreement and the per-model odds ratios with FDR correction (Table 7, Table 25, Figure 3); the surface-confound controls (Table 28); the ensemble's calibration (Tables 26–27); and the bounding-risk study (§3.6). Each decision threshold is the one used in the corresponding experiment (supervised thresholds tuned by nested CV; the XLM-R-NLI baseline by max-MCC over a coarse threshold grid; the BGE cross-encoder by its two-class argmax).

The unsupervised ranking of §5.3 (Tables 5, 6, 21–24) and the descriptive statistics (Tables 1, 8) are produced by the corresponding analysis and pipeline notebooks rather than by `08`.

## Configuration

The pipeline notebooks (`01`, `02`, `07`) call external APIs. Provide credentials through environment variables; none are stored in the repository:

| Variable | Used by |
|---|---|
| `JUDILIBRE_API_KEY` | `01_data_collection` (Judilibre decisions) |
| `LEGIFRANCE_API_KEY` / `LEGIFRANCE_API_SECRET` | `02_article_retrieval` (Civil Code) |
| `OPENAI_API_KEY` | `07_adversarial_filter_o3` (o3 filter) |
| `HF_TOKEN` | model downloads in `04`/`05` and the fine-tuning / LLM notebooks in `CODE/analysis/` |

The reproduction notebook (`08`) uses none of these.

## Data availability

See [DATA.md](DATA.md) for the inventory of what is shipped here, what is archived on Zenodo, and what is regenerated by the pipeline, and for the schema of `DATA/outputs/benchmark.csv`.

The court decisions come from [Judilibre](https://www.courdecassation.fr/acces-rapide-judilibre), the open-data API for French court decisions, under the Etalab Open Licence 2.0. They are pseudonymised at source.

## Citation

If you use this benchmark, code or model, please cite the paper:

```bibtex
@misc{floro2026experts,
  title         = {Where Experts Disagree, Models Fail: Detecting Implicit Legal Citations in French Court Decisions},
  author        = {Floro, Avrile and Dhorasoo, Tamara and Pellez, Soline and Holzenberger, Nils},
  year          = {2026},
  eprint        = {2603.22973},
  archivePrefix = {arXiv},
  primaryClass  = {cs.AI},
  url           = {https://arxiv.org/abs/2603.22973},
  note          = {To be published at the Natural Legal Language Processing Workshop (NLLP 2026)}
}
```

The model and data archive can be cited as:

```bibtex
@dataset{floro_2026_zenodo,
  title     = {Where Experts Disagree, Models Fail: model and datasets for detecting implicit legal citations in French court decisions},
  author    = {Floro, Avrile and Dhorasoo, Tamara and Pellez, Soline and Holzenberger, Nils},
  publisher = {Zenodo},
  year      = {2026},
  doi       = {10.5281/zenodo.21206799}
}
```

## License

Code released under the MIT License (see [LICENSE](LICENSE)). Court decision texts are reused under the Etalab Open Licence 2.0.
