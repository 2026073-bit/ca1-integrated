# CA1 Integrated Assessment MSc Data Analytics CCT Semester 2

## Paper Name :  **The Precision Budget of Columnar Storage: CSV, Parquet and Neural Networks on Large-Scale Tabular Data**

The chosen topic is columnar storage and neural networks, with a focus on the numerical precision of training data. CSV is still the most common format to exchange data, but it stores every number as text, and a reader must scan every full line even when it needs only two columns. Columnar formats such as Parquet, read by engines such as Apache Spark, read only the columns a query needs (Armbrust et al., 2015). Besides the format, whoever designs a table also decides the precision of each column, and this decision is rarely checked against the model trained later. This project measures both sides in one pipeline, the storage cost of the format, the codec and the precision, and the effect of each precision level on the error of a Multilayer Perceptron (MLP) that predicts the duration of New York taxi trips.

*Research question: How much numerical precision can a columnar store lose before the error of a neural network trained on it starts to rise, and how much storage does each level of precision save?*

## A. Objectives

- Compare CSV and Parquet, with and without compression (gzip and zstd), in size on HDFS and in Spark read time.
- Measure the read time with all columns and with only the five columns the network uses.
- Measure how much storage is saved when the distance and the timestamps lose precision.
- Design an MLP with zone embeddings in phases, with one change per phase, and select the distance scaler.
- Test whether the loss of precision increases the test error of the network.
- Discuss the reasons for the selection of an MLP for this data.

## B. Conclusions

1. Parquet with zstd was the best option to store the training data. It was 7.2 times smaller than CSV and read 5 to 12 times faster, mainly when only a few columns are read.
2. A smaller file does not always give a faster read. Gzipped CSV was smaller than uncompressed Parquet, but Spark read it about 6 times more slowly.
3. The saving from a precision reduction depends on how many distinct values remain in the column.
4. The timestamps tolerate 15-minute blocks. T15 cut the payload by 57% and changed the test error by less than 1%. This requires the true duration to be computed before the truncation.
5. The distance tolerates one decimal, but not zero. With whole miles, the test error rose by 6.7%.
6. The distance scaler barely changed the final error. Standard scaling reached the same error as the unscaled distance with 25 fewer epochs.
7. A fixed 1% margin, set before the results, is a simple rule to decide if a precision reduction is acceptable when each condition is trained only once.

## C. Implementation Steps

1. Prepare the environment. Ubuntu VM in VirtualBox, Hadoop 3.4.3 with pseudo-distributed HDFS at `hdfs://localhost:9000`, Java 17 and Python 3.12:
```
   uv venv --python 3.12 ~/ca1env
   source ~/ca1env/bin/activate
   uv pip install -r requirements.txt
   python -m ipykernel install --user --name ca1env --display-name "CA1 (py3.12)"
```
2. Download `yellow_tripdata_2024-01.parquet`, `yellow_tripdata_2024-02.parquet`, `yellow_tripdata_2024-03.parquet` and `taxi_zone_lookup.csv` from the Google Drive folder [Google Drive link].
3. Start HDFS and load the raw data:
```
   start-dfs.sh
   hdfs dfs -mkdir -p /ca1/raw
   hdfs dfs -put yellow_tripdata_2024-0[1-3].parquet taxi_zone_lookup.csv /ca1/raw/
```
4. Start JupyterLab from the project folder and select the kernel "CA1 (py3.12)":
```
   cd ~/ca1
   jupyter lab
```
5. Run `00_inspect_raw.ipynb`. It checks the quality of the raw data, applies the cleaning funnel and writes the clean base, with the true duration, to HDFS.
6. Run `01_formats.ipynb`. It writes the clean base as CSV, CSV gzip, Parquet and Parquet zstd, and measures the size and the read time of each format.
7. Run `02_precision_levels.ipynb`. It adds the keys (trip id, time split and 20% sample) and writes the precision levels L0, D1, D0 and T15.
8. Run `03_nn_design.ipynb`. It measures two baselines, designs the network in phases, tests five distance scalers and freezes the final design in `results/03_final_config.json`.
9. Run `04_precision_experiment.ipynb`. It trains the frozen network once on each level and compares the test error with L0.
10. Stop HDFS with `stop-dfs.sh`. The results are saved in `results/` and the charts in `figures/`.

---

© 2026073 CCT College
