# Batch vs stream scoring for real-time fraud detection

A controlled experiment comparing two ways of scoring the same fraud model in Apache PySpark:

- **Batch:** the whole test set is scored in one Spark job.
- **Stream simulation:** each test record is scored in its own Spark job, as if it arrived alone.

Both use the same Random Forest (100 trees, seed 42), the same features and the same train/test split. Only the scoring pattern changes.

This repository accompanies a BSc Honours dissertation at Eduvos (Faculty of Information Technology) by Chulumanco Vumazonke. The dissertation is not included here.

## Contents

| File | Purpose |
|---|---|
| `fraud_batch_vs_stream.ipynb` | The full experiment: data generation, training, both scoring pipelines, metrics, and outputs |
| `requirements.txt` | Python packages |
| `.gitignore` | Keeps generated data and results out of the repository |

Running the notebook creates `data/transactions.csv` and the files in `results/`:

- `timing.csv`: elapsed time, latency, throughput, CPU and memory for each pipeline
- `classification_metrics.csv`: confusion matrix counts and metrics, both fraud-class and class-weighted
- `batch_predictions.csv` and `stream_predictions.csv`: one prediction per test record. The notebook prints how many records the two pipelines disagree on.

## Running it

### Google Colab

1. Upload the notebook, or clone this repository in a Colab cell.
2. Choose a CPU runtime. The first cell installs the packages.
3. Set `DATA_DIR` to a folder in your Google Drive if you want the data and results kept between sessions. You can set it with an environment variable or by editing the configuration cell.
4. Run all cells. The full stream run takes roughly 2.5 hours.

### Local machine

- Python 3 (Colab runs Python 3.12)
- Java 17 or 21 (the smoke test used Java 21)

```bash
pip install -r requirements.txt
jupyter notebook fraud_batch_vs_stream.ipynb
```

For a quick check that the pipeline runs, set `TEST_LIMIT` to a small number, for example 300:

```bash
TEST_LIMIT=300 jupyter nbconvert --to notebook --execute fraud_batch_vs_stream.ipynb
```

Put an existing `transactions.csv` in `data/` to skip generation and use that exact file.

## Reproducibility

- Python's `random`, NumPy and Faker are seeded with 42. Spark's `randomSplit` and the Random Forest use seed 42.
- The notebook trains the model once and scores the test set with both patterns, so the two pipelines share a model.
- Training time is reported separately and is not part of either scoring measurement.
- Feature engineering is inside both timers. The batch timer covers the whole test set. The stream timer covers each record, and building the one-row DataFrame is excluded.

**Important:** the dataset produced by this notebook is not identical to the one in the dissertation. With seed 42, the generator in this notebook produces 3,071 fraud records (3.07%). The dissertation reports 3,063. The dissertation's original generator could not be run exactly from the copy available to me, so the cause of the difference is not known. To reproduce the dissertation's figures, use the original `transactions.csv` from the original run and place it in `data/`.

## Results and corrections

The dissertation's first analysis reported an F1 gap of about 30 percentage points between batch (98.14%) and stream (67.76%). That comparison was not like for like. The batch figures were class-weighted averages, which Spark's default evaluator reports. The stream figures were for the fraud class only. Because both pipelines use the same deterministic model and stateless features, they should make the same predictions. The notebook checks this directly by computing both metric sets for both pipelines and counting disagreements between them.

The earlier batch time of 22.64 s included model training. The notebook times scoring separately from training. The earlier figures are kept here for reference only:

| | Batch | Stream simulation |
|---|---|---|
| Elapsed time, earlier run | 22.64 s (includes training) | 8,885.15 s (scoring only) |
| Mean latency per record, earlier run | 1.13 ms (amortised) | 442.03 ms |
| Fraud-class F1, reconstructed from confusion matrix | 67.76% | 67.76% |

The reconstructed confusion matrix from the earlier run is TP 371, FP 111, FN 242, TN 19,377. It is inferred from the reported figures and should be replaced by the real matrix from `classification_metrics.csv`.

A smoke run with 300 test records completed without errors. It only tests the code path. Its numbers are not meaningful as results. The full experiment was not run for this repository.

## Limitations

- **Synthetic data.** The generator creates overlapping fraud and legitimate records, but real fraud behaves differently. The fraud rate is 3% and features are stateless.
- **Simulated streaming.** The stream pipeline launches one Spark job per record. It is a naive design, not a streaming engine such as Structured Streaming, Kafka or Flink. The results describe this pattern only.
- **One run.** Each pipeline was run once, so run-to-run variation is unknown.
- **Coarse resource data.** CPU is system-wide and sampled at intervals. Memory is measured on the Python process, while most work happens in the JVM.
- **Platform.** Results depend on the hardware. The original runs were on a Google Colab CPU runtime, and the exact processor was not recorded.

## Licence and citation

Licence: [to be decided by the author].

Citation: Vumazonke, C. (2026) *A comparative analysis of batch and stream processing architectures for real-time financial fraud detection*. [Eduvos BSc Honours dissertation; add the final title and year.]
