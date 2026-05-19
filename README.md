## Fault Detection in Circuit Netlists using Graph Neural Networks

This repository contains the datasets, fault-injection scripts, graph-construction pipeline, feature-engineering pipeline, GNN training notebooks, hybrid GNN experiments, and interpretability notebooks used for fault detection in gate-level circuit netlists.

The task is formulated as node-level multi-class classification over four classes:

- `clean`
- `stuck_at`
- `glitch`
- `bridging`

The repository supports both the baseline fault-injection setting and the varying-fault-count setting used for analyzing how the number of injected faults changes the graph dataset statistics.

---

## 1. Repository Overview

The repository is organized as follows:

```text
.
├── Fault Injection Netlists/
│   ├── Baseline Dataset/
│   │   ├── clean_netlists/
│   │   ├── stuck_at/
│   │   ├── glitch/
│   │   └── bridging/
│   │
│   └── fault_injection_fig_faultsweep/
│       ├── clean_netlists/
│       ├── 100-500/
│       │   ├── stuck_at/
│       │   ├── glitch/
│       │   └── bridging/
│       ├── 500-1000/
│       │   ├── stuck_at/
│       │   ├── glitch/
│       │   └── bridging/
│       ├── 1000-2000/
│       │   ├── stuck_at/
│       │   ├── glitch/
│       │   └── bridging/
│       ├── graph_csv_by_faultnumber/
│       ├── faultnumber_statistics/
│       ├── fault_metadata_detailed.csv
│       ├── fault_injection_summary.csv
│       ├── range_summary.csv
│       └── README.md
│
├── Jupyter Notebooks/
│   ├── Fault Injection and Parsing/
│   ├── Standard GNNs/
│   ├── SOTA Hardware Security/
│   ├── Hybrid-GNNs/
│   └── Interpretability/
│
└── baseline_original_statistics/

```

---

## 2. Fault-Injected Netlists

All Verilog netlists used in the experiments are stored under:

```text
Fault Injection Netlists/
```

There are two main datasets.

---

### 2.1 Baseline Dataset

The baseline dataset is stored in:

```text
Fault Injection Netlists/Baseline Dataset/
```

It contains:

```text
clean_netlists/   # original clean Verilog netlists
stuck_at/         # Verilog netlists with stuck-at faults
glitch/           # Verilog netlists with transient glitch faults
bridging/         # Verilog netlists with bridging faults
```

For every clean benchmark netlist, three corresponding faulted versions are provided:

```text
stuck_at_<benchmark>.v
glitch_<benchmark>.v
bridging_<benchmark>.v
```

The baseline fault-injection setting uses the following rule to select the number of injected faults per benchmark and fault type:

```python
k = min(100, max(10, int(0.1 * len(nets))))
```

This means that the number of injected faults is circuit-dependent, with a minimum of 10 and a maximum of 100.

---

### 2.2 Varying-Fault-Count Dataset

The varying-fault-count dataset is stored in:

```text
Fault Injection Netlists/fault_injection_fig_faultsweep/
```

It contains three progressively larger injected-fault settings:

```text
100-500/
500-1000/
1000-2000/
```

Each setting contains three fault-type folders:

```text
stuck_at/
glitch/
bridging/
```

For each benchmark and each setting, one requested number of faults is sampled and reused across all three fault types. The requested number is included in the filename. For example:

```text
bridging_k376_adder.v
glitch_k376_adder.v
stuck_at_k376_adder.v
```

Here, `k376` means that 376 faults were requested for that benchmark in the corresponding setting.

The actual number of inserted faults may be lower than the requested number when the netlist does not contain enough feasible candidate nets or enough feasible same-width bridging pairs. The repository records this information in:

```text
Fault Injection Netlists/fault_injection_fig_faultsweep/fault_metadata_detailed.csv
Fault Injection Netlists/fault_injection_fig_faultsweep/fault_injection_summary.csv
Fault Injection Netlists/fault_injection_fig_faultsweep/range_summary.csv
```

The statistics for this varying-fault-count dataset are stored in:

```text
Fault Injection Netlists/fault_injection_fig_faultsweep/faultnumber_statistics/
```

This folder contains CSV summaries and plots that report class counts, fault-node percentages, per-graph fault counts, topology summaries, and feature summaries.

---

## 3. Fault Models

The repository considers three representative fault types.

---

### 3.1 Stuck-at Faults

A stuck-at fault permanently forces a selected net to logic `0` or logic `1`.

Example injected Verilog form:

```verilog
assign net_name = 1'b0;
```

or:

```verilog
assign net_name = 1'b1;
```

In the graph representation, stuck-at faults are embedded as pseudo source nodes:

```text
SA0_<net>
SA1_<net>
```

Only these pseudo fault nodes are labeled as faulty.

---

### 3.2 Glitch Faults

A glitch fault models a transient perturbation by toggling a selected net twice at random delays.

Example injected Verilog form:

```verilog
initial begin
  #d1 net_name = ~net_name;
  #d2 net_name = ~net_name;
end
```

In the graph representation, glitch faults are embedded as pseudo nodes:

```text
GL_<net>
```

Only these pseudo fault nodes are labeled as faulty.

---

### 3.3 Bridging Faults

A bridging fault shorts two selected nets by assigning one net to the other.

Example injected Verilog form:

```verilog
assign net1 = net2;
```

In the graph representation, bridging faults are embedded as pseudo nodes:

```text
BR_<net1>_<net2>
```

For the varying-fault-count dataset, bridging pairs are restricted to feasible same-width pairs to avoid trivial vector-width mismatch artifacts.

---

## 4. Graph CSV Files

The Verilog netlists are converted into fault-aware graph CSV files. The graph construction embeds injected faults directly into the graph topology using pseudo fault nodes.

For the varying-fault-count dataset, the raw graph CSV files are stored in:

```text
Fault Injection Netlists/fault_injection_fig_faultsweep/graph_csv_by_faultnumber/
```

This folder contains:

```text
nodes_faultnumber_100_500.csv
edges_faultnumber_100_500.csv

nodes_faultnumber_500_1000.csv
edges_faultnumber_500_1000.csv

nodes_faultnumber_1000_2000.csv
edges_faultnumber_1000_2000.csv
```

The graph CSV files contain the fault-aware topology before enhanced feature engineering.

---

## 5. Enhanced Feature CSV Files

The feature-engineering pipeline computes enhanced node and edge features from the raw graph CSV files.

The enhanced node features include:

- gate type
- in-degree
- out-degree
- total degree
- path depth
- exact SCOAP-style controllability:
  - `CC0_exact`
  - `CC1_exact`
- exact SCOAP-style observability:
  - `CO_exact`
- distance to primary-output-like sink nodes
- fanout
- normalized fanout
- gate delay
- node label

The enhanced edge CSV files retain graph connectivity and fanout metadata.

Most training notebooks expect the following file names:

```text
nodes_enhanced.csv
edges_enhanced.csv
```

For varying-fault-count experiments, the enhanced files are generated with setting-specific names such as:

```text
nodes_enhanced_faultnumber_100_500.csv
edges_enhanced_faultnumber_100_500.csv

nodes_enhanced_faultnumber_500_1000.csv
edges_enhanced_faultnumber_500_1000.csv

nodes_enhanced_faultnumber_1000_2000.csv
edges_enhanced_faultnumber_1000_2000.csv
```

If a notebook expects the generic names `nodes_enhanced.csv` and `edges_enhanced.csv`, either copy the desired pair to those names or update the path variables at the top of the notebook.

---

## 6. Jupyter Notebook Organization

All experiment notebooks are stored in:

```text
Jupyter Notebooks/
```

---

### 6.1 Fault Injection and Parsing

Folder:

```text
Jupyter Notebooks/Fault Injection and Parsing/
```

Important notebooks:

```text
Fault_injection_netlist2graph.ipynb
Fault_injection_netlist2graph_data_varying_fault.ipynb
```

These notebooks are used to:

1. Generate fault-injected Verilog netlists.
2. Convert fault-injected Verilog netlists into graph CSV files.
3. Create graph CSV files for varying injected-fault settings.
4. Generate enhanced feature CSV files.
5. Compute dataset statistics for reviewer-facing analysis.

---

### 6.2 Standard GNNs

Folder:

```text
Jupyter Notebooks/Standard GNNs/
```

This folder contains standard GNN architectures used as baseline models:

```text
Fault_detection_GCN.ipynb
GNN_fault_detection_GraphSAGE.ipynb
GNN_fault_detection_GATVanilla.ipynb
GNN_fault_detection_GIN.ipynb
GNN_fault_detection_MPNN.ipynb
GNN_fault_detection_APPNP.ipynb
GNN_fault_detection_SAGE-C-GAT.ipynb
```

These models evaluate commonly used GNN backbones on the same four-class node-level fault-detection task.

---

### 6.3 SOTA Hardware Security GNNs

Folder:

```text
Jupyter Notebooks/SOTA Hardware Security/
```

This folder contains GNN models adapted from hardware-security, reverse-engineering, hardware-Trojan detection, FPGA-assurance, and EDA-related graph-learning literature.

Included notebooks:

```text
AppGNN-style Approximation-Aware GAT.ipynb
AttackGNN-target-family Ensemble GNN.ipynb
BadGNN GNN4TJ-style GCN.ipynb
Circuit-Rewrite-Aware GraphSAGE and GNN-RE-style Model.ipynb
DepthGraphNet-style Diameter-Aware Siamese GIN.ipynb
FP-GNN style.ipynb
GIN-based TROJAN-GUARD-style.ipynb
GNN-MFF-style multi-view feature-fusion GNN.ipynb
GNN-RE-style GAT.ipynb
GNN4REL.ipynb
GateDet-style Bidirectional GCN.ipynb
Long-Short-GNN-style Dual-View GNN.ipynb
MaliGNNoma-style GIN.ipynb
RELUT-GNN-style Mean-Aggregation GCN.ipynb
SAGE-GraphTransformer.ipynb
TrojanSAINT-Style-Inductive-GNN.ipynb
```

Each notebook adapts the corresponding architecture to the same fault-aware gate-level graph dataset.

---

### 6.4 Hybrid GNNs

Folder:

```text
Jupyter Notebooks/Hybrid-GNNs/
```

This folder contains the proposed hybrid architectures:

```text
Our proposed architecture-1.ipynb
Our proposed architecture-2.ipynb
Our proposed architecture-3.ipynb
Our proposed architecture-4.ipynb
Our proposed architecture-5.ipynb
```

These architectures combine GNN embeddings with additional verification or rejection mechanisms, including:

- tree-based verifiers
- prototype-mixture rejectors
- clean-manifold autoencoders
- pairwise ranking SVMs
- differentiable rule-gated neuro-symbolic classifiers

The goal is to test whether a second-stage verifier can reduce clean-to-fault false positives.

---

### 6.5 Interpretability

Folder:

```text
Jupyter Notebooks/Interpretability/
```

Included notebooks:

```text
GNN_Explainer.ipynb
Interpretability_fault_hybridGAT.ipynb
```

These notebooks analyze important graph features and model explanations.

---

## 7. Statistics Folders

### 7.1 Baseline Dataset Statistics

Folder:

```text
baseline_original_statistics/
```

This folder contains baseline statistics, including:

```text
baseline_node_class_counts.csv
baseline_node_class_percentages.csv
baseline_class_balance_summary.csv
baseline_faulttype_graph_summary.csv
baseline_feature_summary_by_class.csv
baseline_topology_summary.csv
baseline_per_graph_class_counts.csv
baseline_per_graph_fault_counts.csv
```

It also contains plots such as:

```text
baseline_mean_fault_nodes_per_graph.png
baseline_node_class_percentages.png
baseline_total_labeled_fault_nodes_by_fault_type.png
```

Comparison files between the baseline and varying-fault-count datasets are also included:

```text
comparison_faulttype_graph_summary_baseline_vs_faultnumber.csv
comparison_mean_fault_nodes_per_graph_baseline_vs_faultnumber.png
comparison_node_class_counts_baseline_vs_faultnumber.csv
comparison_node_class_percentages_baseline_vs_faultnumber.csv
comparison_node_class_percentages_baseline_vs_faultnumber.png
comparison_topology_summary_baseline_vs_faultnumber.csv
comparison_total_labeled_fault_nodes_baseline_vs_faultnumber.png
```

---

### 7.2 Varying-Fault-Count Statistics

Folder:

```text
Fault Injection Netlists/fault_injection_fig_faultsweep/faultnumber_statistics/
```

This folder contains:

```text
range_node_class_counts.csv
range_node_class_percentages.csv
range_class_balance_summary.csv
range_faulttype_graph_summary.csv
range_topology_summary.csv
per_graph_class_counts.csv
per_graph_fault_counts.csv
per_graph_fault_counts_vs_requested.csv
feature_summary_by_range_and_class.csv
fault_node_growth_factors.csv
```

It also contains plots:

```text
fault_node_density_by_range.png
mean_fault_nodes_per_graph_by_range.png
node_class_percentages_by_range.png
total_labeled_fault_nodes_by_range.png
```

These statistics support the analysis of how increasing the number of injected faults changes the number and percentage of labeled fault nodes.

---

## 8. Data Format

### 8.1 Node CSV Format

Typical node columns include:

```text
netlist_file
fault_type
node_id
gate_type
in_degree
out_degree
total_degree
path_depth
CC0_exact
CC1_exact
CO_exact
dist_to_PO
fanout
fanout_norm
gate_delay
label
```

The `label` column marks pseudo fault nodes with `1`; all ordinary circuit nodes receive `0`.

During GNN training, the final class label is inferred from both `fault_type` and `label`:

- `fault_type = clean` gives clean labels.
- `fault_type = stuck_at` and `label = 1` gives stuck-at labels.
- `fault_type = glitch` and `label = 1` gives glitch labels.
- `fault_type = bridging` and `label = 1` gives bridging labels.

---

### 8.2 Edge CSV Format

Typical edge columns include:

```text
netlist_file
fault_type
src_node
dst_node
fan_out
```

Edges are directed and represent signal flow between circuit graph nodes. Some notebooks add reverse edges internally to expose both fan-in and fan-out neighborhoods during message passing.

---

## 9. Dependencies

The repository is notebook-based and uses Python, PyTorch, PyTorch Geometric, scikit-learn, pandas, and plotting libraries.

Recommended Python version:

```text
Python >= 3.9
```

Core packages:

```text
numpy
pandas
scikit-learn
matplotlib
seaborn
joblib
tqdm
jupyter
torch
torch-geometric
```

A typical conda environment can be created as follows:

```bash
conda create -n faultnetlist python=3.10
conda activate faultnetlist
```

Install common Python packages:

```bash
pip install numpy pandas scikit-learn matplotlib seaborn joblib tqdm notebook jupyter
```

Install PyTorch according to your system and CUDA version.

For CPU-only installation:

```bash
pip install torch torchvision torchaudio
```

Then install PyTorch Geometric:

```bash
pip install torch-geometric
```

If PyTorch Geometric installation fails, install the version-specific wheels for your PyTorch and CUDA version following the official PyTorch Geometric installation instructions.

---

## 10. How to Run the Experiments

### Step 1: Use the Provided Fault-Injected Netlists

The repository already includes the generated Verilog netlists.

Baseline dataset:

```text
Fault Injection Netlists/Baseline Dataset/
```

Varying-fault-count dataset:

```text
Fault Injection Netlists/fault_injection_fig_faultsweep/
```

You can start directly from graph construction, enhanced feature generation, or GNN training.

---

### Step 2: Generate Graph CSV Files

Use notebooks in:

```text
Jupyter Notebooks/Fault Injection and Parsing/
```

For baseline data, use:

```text
Fault_injection_netlist2graph.ipynb
```

For varying-fault-count data, use:

```text
Fault_injection_netlist2graph_data_varying_fault.ipynb
```

These notebooks convert Verilog netlists into graph CSV files.

---

### Step 3: Generate Enhanced Features

The feature-engineering pipeline computes enhanced node and edge features from the raw graph CSV files.

For the baseline experiments, generate or use:

```text
nodes_enhanced.csv
edges_enhanced.csv
```

For the varying-fault-count experiments, generate:

```text
nodes_enhanced_faultnumber_100_500.csv
edges_enhanced_faultnumber_100_500.csv

nodes_enhanced_faultnumber_500_1000.csv
edges_enhanced_faultnumber_500_1000.csv

nodes_enhanced_faultnumber_1000_2000.csv
edges_enhanced_faultnumber_1000_2000.csv
```

---

### Step 4: Train a Standard GNN

Open one of the notebooks in:

```text
Jupyter Notebooks/Standard GNNs/
```

For example:

```text
GNN_fault_detection_GraphSAGE.ipynb
```

Run the notebook cells in order.

Each training notebook performs:

1. Loading enhanced node and edge CSV files.
2. Constructing one graph per netlist.
3. Creating a netlist-level train/validation/test split.
4. Training the GNN.
5. Reporting accuracy, precision, recall, F1-score, and confusion matrix.

---

### Step 5: Train SOTA Hardware-Security-Inspired GNNs

Open a notebook from:

```text
Jupyter Notebooks/SOTA Hardware Security/
```

These notebooks adapt prior GNN architectures from hardware-security and EDA literature to the four-class fault-detection problem.

---

### Step 6: Train Hybrid GNNs

Open a notebook from:

```text
Jupyter Notebooks/Hybrid-GNNs/
```

These notebooks train a GNN encoder and then apply a second-stage verifier or rejector.

Some hybrid notebooks generate additional files such as:

```text
*.pt
*.joblib
```

These are model checkpoints and verifier files.

---

### Step 7: Run Interpretability Analysis

Open notebooks from:

```text
Jupyter Notebooks/Interpretability/
```

These notebooks analyze important structural and relational features and can be used to generate interpretability results.

---

## 11. Reproducibility

Most notebooks use:

```python
SEED = 42
```

The fault-injection scripts also use deterministic random seeds. For the varying-fault-count dataset, per-file and per-fault seeds are derived from the global seed and the benchmark/fault setting, making the generated datasets reproducible.

Results may vary slightly depending on:

- PyTorch version
- PyTorch Geometric version
- CUDA version
- GPU nondeterminism
- notebook execution order

---

## 12. Scalability Notes

Large netlists can require substantial memory during full-graph GNN training. Several notebooks use sampling-based training, including GraphSAINT-style random-walk sampling and induced-subgraph sampling.

If memory issues occur, reduce parameters such as:

```python
BATCH_SIZE
ROOT_NODES
SAMPLE_STEPS_PER_EPOCH
MAX_SUBGRAPH_NODES
MAX_FAULT_NODES_PER_CLASS_PER_GRAPH
HID
```

---

## 13. Typical End-to-End Workflow

A typical workflow is:

```text
1. Start from clean Verilog netlists.
2. Inject stuck-at, glitch, and bridging faults.
3. Convert clean and faulted netlists into graph CSV files.
4. Enhance graph CSVs with SCOAP, topology, fanout, delay, and relational features.
5. Train GNN models using enhanced node and edge CSV files.
6. Evaluate class-wise precision, recall, F1-score, accuracy, and confusion matrix.
7. Generate dataset statistics for fault-count analysis.
8. Run interpretability notebooks to inspect important features.
```

---

## 14. Citation

If you use this repository, dataset, or code, please cite:

```bibtex
@inproceedings{karn2026faultdate,
  title        = {Interpretable GNNs for Fault Detection in Circuits},
  author       = {Karn, Rupesh Raj and Knechtel, Johann and Sinanoglu, Ozgur},
  booktitle    = {Proceedings of the 32nd IEEE International Symposium on On-Line Testing and Robust System Design},
  year         = {2026},
  address      = {Polignano a Mare (BA), Italy},
  month        = {1-3, July}
}
```

---

## 15. Contact

For questions about the dataset, notebooks, or experiments, please open an issue in this repository or contact the authors of the associated paper.
