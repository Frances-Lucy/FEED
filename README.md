# FEED

FEED is a model-training-free, feedback-guided error detection system. It uses a large language model to generate and revise executable detection rules from compact evidence and a small number of annotations. Candidate revisions are validated before the selected rules are applied locally to the complete input table.

## Key files

```text
FEED/
├── pipeline.py                    # End-to-end FEED pipeline
├── config.py                      # Configuration loader
├── rule_engine.py                 # Rule execution and detection state
├── sampler.py                     # Active and contrastive sampling
├── clusterer.py                   # Representative-value clustering
├── embedder.py                    # Local embedding inference
├── rag_retriever.py               # Domain-document retrieval
├── contrast_pairs_builder.py      # Contrastive evidence construction
├── llm_client.py                  # LLM client
├── evaluator.py                   # Detection evaluation
├── output.py                      # Run artifacts and summaries
├── datasets/                      # Benchmark datasets
├── all-MiniLM-L6-v2/              # Local embedding model
├── requirements.txt
├── .env                           # Private local configuration
└── README.md
```

## Tested environment

The main experiments were run with Python 3.10.12 on Ubuntu 22.04.3 LTS, using a 16-vCPU quota on an Intel Xeon Platinum 8352V host, 120 GiB of RAM, and one NVIDIA RTX 4090 GPU. A GPU is recommended for embedding inference but is not required for rule execution.

## Data

Place every dataset in its own directory under `datasets/`. `DATA_ROOT` must point to the parent directory.

```text
datasets/
├── beers/
│   ├── beers_dirty.csv
│   ├── beers_clean.csv
│   └── beers_dirty_error_detection.csv
├── hospital/
│   ├── hospital_dirty.csv
│   ├── hospital_clean.csv
│   └── hospital_dirty_error_detection.csv
├── movies/
├── tax/
├── imdb/
└── taxi/
```

The experiments use the following six datasets.

| Dataset | Directory | Dirty table | Reference data |
|---|---|---|---|
| Beers | `datasets/beers/` | `beers_dirty.csv` | clean table or error-detection matrix |
| Hospital | `datasets/hospital/` | `hospital_dirty.csv` | clean table or error-detection matrix |
| Movies | `datasets/movies/` | `movies_dirty.csv` | clean table or error-detection matrix |
| Tax | `datasets/tax/` | `tax_dirty.csv` | clean table or error-detection matrix |
| IMDB | `datasets/imdb/` | `imdb_dirty.csv` | clean table or error-detection matrix |
| Taxi | `datasets/taxi/` | `taxi_dirty.csv` | clean table or error-detection matrix |

The clean tables and error-detection matrices are used only as annotation oracles and for evaluation. They are not included in the evidence supplied for initial rule generation.

## Environment setup

The following commands assume Linux and Python 3.10.

```bash
git clone https://github.com/Frances-Lucy/FEED.git
cd FEED

python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

Download `sentence-transformers/all-MiniLM-L6-v2` into `all-MiniLM-L6-v2/`. For the ONNX backend used in the experiments, the directory must contain the tokenizer files and `onnx/model_O4.onnx`.

## Configuration

Create a private `.env` file and fill in the dataset path, output path, LLM endpoint, credentials, and local embedding-model path. A minimal configuration is shown below.

```dotenv
DATASET_NAME=beers
DATA_ROOT=/absolute/path/to/FEED/datasets
OUTPUT_DIR=/absolute/path/to/FEED/output/beers

API_KEY_RULE_GENERATION=YOUR_API_KEY
API_BASE_RULE_GENERATION=https://YOUR_OPENAI_COMPATIBLE_ENDPOINT/v1
MODEL_NAME_RULE_GENERATION=YOUR_MODEL_NAME

API_KEY_RULE_UPDATE=YOUR_API_KEY
API_BASE_RULE_UPDATE=https://YOUR_OPENAI_COMPATIBLE_ENDPOINT/v1
MODEL_NAME_RULE_UPDATE=YOUR_MODEL_NAME

EMBEDDING_MODEL_PATH=/absolute/path/to/FEED/all-MiniLM-L6-v2
EMBEDDING_BACKEND=onnx
EMBEDDING_ONNX_FILE=onnx/model_O4.onnx
EMBEDDING_PROVIDER=CUDAExecutionProvider

RAG_ENABLED=true
RAG_DOC_PATH=rag_document.json
CONTRASTIVE_ENABLED=true
GLOBAL_SEED=0
```

The remaining FEED parameters use the defaults defined in `config.py`. To run embeddings on the CPU, set `EMBEDDING_PROVIDER=CPUExecutionProvider`. Never commit `.env` or a real API key.

## Run FEED

Activate the environment and start the pipeline.

```bash
source .venv/bin/activate
python pipeline.py
```

To run another dataset, update `DATASET_NAME` and `OUTPUT_DIR` in `.env`, then execute `python pipeline.py` again. For example, a Linux batch run can be launched as follows.

```bash
for dataset in beers hospital movies tax imdb taxi; do
  DATASET_NAME="$dataset" \
  OUTPUT_DIR="$PWD/output/$dataset" \
  python pipeline.py
done
```

The parameter and ablation experiments additionally use Tax-10K. Configure it as a separate dataset directory and set `DATASET_NAME=tax-10k` when reproducing those experiments.

## Outputs

Each run writes its artifacts to `OUTPUT_DIR`. The principal outputs include generated and revised rules, annotation records, candidate-validation decisions, the final detection matrix, evaluation metrics, and runtime summaries. Preserve the complete output directory together with the configuration and random seed for each run.

## License and data availability

Code and configuration files follow the repository license. Benchmark datasets remain subject to their original licenses and terms of use. When redistribution is restricted, obtain the data from the original source and place it in the directory structure described above.
