# OntoVAE Codebase Explanation

## Overview

**OntoVAE** (Ontology-guided Variational Autoencoder) is a Python package for integrating biological ontologies into the architecture of Variational Autoencoder (VAE) models. This enables direct retrieval of pathway activities and allows simulation of genetic or drug-induced perturbations.

**Research Paper**: ["Biologically informed variational autoencoders allow predictive modeling of genetic and drug induced perturbations"](https://academic.oup.com/bioinformatics/article/39/6/btad387/7199588) published in Bioinformatics (2023).

**Note**: This repository is no longer actively maintained. The code is now also hosted at https://github.com/hdsu-bioquant/cobra-ai

---

## Project Structure

```
onto-vae/
├── onto_vae/              # Main package directory
│   ├── __init__.py        # Package initialization (empty)
│   ├── ontobj.py          # Ontology preprocessing class
│   ├── vae_model.py       # VAE model implementations
│   ├── modules.py         # Neural network modules
│   ├── utils.py           # Utility functions for DAG operations
│   ├── fast_data_loader.py # Optimized data loader
│   └── data/              # Sample data files
│       ├── pw.obo         # Pathway ontology file
│       ├── gene_term_mapping.txt
│       ├── pbmc_sample_annot.csv
│       └── pbmc_sample_expr.csv
├── pyproject.toml         # Poetry package configuration
├── requirements.txt       # Pip requirements
├── vignette.ipynb         # Tutorial/example notebook
├── README.md              # Main documentation
└── LICENSE                # GPL-3.0 License
```

---

## Core Components

### 1. Ontobj Class (`ontobj.py`) - Ontology Preprocessing

The `Ontobj` class is the foundation of OntoVAE. It serves as a container for preprocessed ontologies and matched datasets.

#### Key Attributes:
- **`annot_base`**: Base annotation file containing ontology term metadata (ID, Name, depth, children, parents, descendants, genes)
- **`genes_base`**: List of genes that can be mapped to the ontology (alphabetically sorted)
- **`graph_base`**: Dictionary representing ontology relationships (children → parents)
- **`annot`, `genes`, `graph`**: Trimmed versions of the base files
- **`desc_genes`**: Dictionary mapping terms to their descendant genes
- **`masks`**: Binary masks for neural network wiring
- **`sem_sim`**: Semantic similarity matrices
- **`data`**: Dictionary of matched expression datasets

#### Main Methods:

**Initialization:**
- **`initialize_dag(obo, gene_annot, filter_id=None)`**: Loads an ontology from OBO file and gene-term annotations
  - Parses the directed acyclic graph (DAG)
  - Counts descendants and descendant genes
  - Filters out terms without genes

**Preprocessing:**
- **`trim_dag(top_thresh=1000, bottom_thresh=30)`**: Trims the ontology based on gene count thresholds
  - Removes terms with too many genes (> top_thresh)
  - Removes terms with too few genes (< bottom_thresh)
  - Transfers genes from removed terms to their parents

**Mask Creation:**
- **`create_masks(top_thresh, bottom_thresh, module='decoder')`**: Creates binary masks for neural network connections
  - Determines which neurons can connect between layers
  - Supports both decoder and encoder architectures
  - Masks enforce ontology structure in the network

**Data Management:**
- **`match_dataset(expr_data, name, top_thresh, bottom_thresh)`**: Matches gene expression data to the ontology
  - Aligns genes between dataset and ontology
  - Fills missing values with zeros
  - Stores as numpy array

**Analysis:**
- **`compute_wsem_sim(obo, top_thresh, bottom_thresh)`**: Computes Wang semantic similarities between ontology terms
- **`wilcox_test(control, perturbed, direction, option)`**: Performs paired Wilcoxon test for differential activity analysis
- **`plot_scatter(sample_annot, color_by, act, term1, term2)`**: Visualizes pathway activities

---

### 2. VAE Model Classes (`vae_model.py`)

This file contains three VAE implementations:

#### A. **OntoVAE** - Ontology in Decoder (Default)

The main model that integrates ontology structure into the decoder.

**Architecture:**
- **Encoder**: Standard fully-connected neural network
- **Latent Space**: Bottleneck representation
- **Decoder**: Ontology-structured network (follows DAG hierarchy)

**Key Features:**
- Multiple neurons per ontology term (default: 3)
- Positive weight constraints in decoder
- Masks enforce ontology connectivity

**Main Methods:**
- **`train_model(modelpath, lr, kl_coeff, batch_size, epochs)`**: Trains the model
  - 80/20 train/validation split
  - AdamW optimizer
  - Custom loss: MSE reconstruction + weighted KL divergence
  - Enforces positive weights and mask constraints during training

- **`get_pathway_activities(ontobj, dataset, terms=None)`**: Extracts pathway activities
  - Passes data through the model
  - Averages neuron activations per term
  - Returns activities for all or specified terms

- **`perturbation(ontobj, dataset, genes, values, output)`**: Simulates in silico perturbations
  - Modifies gene expression values
  - Predicts downstream effects on pathways or genes
  - Enables drug/mutation effect prediction

#### B. **OntoEncVAE** - Ontology in Encoder

Alternative architecture with ontology structure in the encoder instead of decoder.

**Differences from OntoVAE:**
- Encoder follows ontology hierarchy
- Decoder is standard fully-connected
- Same training and inference capabilities

#### C. **VAE** - Standard VAE (Baseline)

Standard variational autoencoder without ontology structure.

**Purpose:**
- Baseline comparison
- Simple encoder-decoder architecture
- No biological constraints

---

### 3. Neural Network Modules (`modules.py`)

Defines the building blocks for all VAE architectures.

#### **Encoder Class**
Standard encoder for VAE:
- Linear layers with batch normalization
- ReLU activations
- Dropout regularization
- Outputs: mu (mean) and log_var (log variance) for latent space

#### **Decoder Class**
Standard decoder for VAE:
- Linear layers with batch normalization
- ReLU activations
- Dropout regularization
- Reconstructs input features

#### **OntoEncoder Class**
Ontology-structured encoder:
- Masks enforce ontology connectivity
- Concatenates outputs across layers (skip connections)
- Positive weight constraints
- Custom forward pass following DAG structure

#### **OntoDecoder Class**
Ontology-structured decoder:
- Multiple neurons per term (configurable)
- Masks enforce parent-child relationships
- Concatenates layer outputs
- Positive weight constraints
- Activation hooks for pathway activity extraction

---

### 4. Utility Functions (`utils.py`)

Helper functions for graph operations and ontology processing.

#### Graph Operations:
- **`reverse_graph(graph)`**: Reverses a directed graph (parent→child becomes child→parent)
- **`get_descendants(dag, term)`**: Finds all descendant terms using breadth-first search
- **`get_descendant_genes(dag, descendants)`**: Retrieves genes annotated to descendant terms

#### Mask Creation:
- **`create_binary_matrix(depth, dag, childnum, parentnum)`**: Creates sparse binary matrices encoding parent-child relationships

#### DAG Trimming:
- **`trim_DAG_bottom(DAG, all_terms, trim_terms)`**: Removes low-specificity terms
  - Transfers genes to parent terms
  - Maintains DAG structure
  
- **`trim_DAG_top(DAG, all_terms, trim_terms)`**: Removes high-level (too general) terms

- **`find_all_paths(graph, start, end, path=[])`**: Recursive path-finding in DAG

---

### 5. Fast Data Loader (`fast_data_loader.py`)

**FastTensorDataLoader**: Optimized PyTorch data loader

**Why it's needed:**
- Standard DataLoader is slow for tensor datasets
- Grabs individual indices and concatenates (inefficient)
- This implementation uses tensor slicing (much faster)

**Features:**
- Optional shuffling
- Configurable batch size
- Compatible with PyTorch training loops

---

## Key Algorithms and Workflows

### Workflow 1: Preprocessing an Ontology

```python
from onto_vae.ontobj import Ontobj

# 1. Create ontology object
ontobj = Ontobj(description='GO_BP')

# 2. Initialize from OBO file and gene annotations
ontobj.initialize_dag(
    obo='path/to/go.obo',
    gene_annot='path/to/gene_term_mapping.txt',
    filter_id='biological_process'  # Optional: filter to specific namespace
)

# 3. Trim the ontology
ontobj.trim_dag(top_thresh=1000, bottom_thresh=30)

# 4. Create masks for neural network
ontobj.create_masks(top_thresh=1000, bottom_thresh=30, module='decoder')

# 5. Match expression data
ontobj.match_dataset(
    expr_data='path/to/expression.csv',
    name='training_data',
    top_thresh=1000,
    bottom_thresh=30
)
```

### Workflow 2: Training OntoVAE

```python
from onto_vae.vae_model import OntoVAE

# 1. Initialize model
model = OntoVAE(
    ontobj=ontobj,
    dataset='training_data',
    top_thresh=1000,
    bottom_thresh=30,
    neuronnum=3,      # Neurons per pathway
    drop=0.2,         # Dropout rate
    z_drop=0.5        # Latent space dropout
)

# 2. Train the model
model.train_model(
    modelpath='best_model.pth',
    lr=1e-4,
    kl_coeff=1e-4,
    batch_size=128,
    epochs=300
)
```

### Workflow 3: Analyzing Pathway Activities

```python
# 1. Get pathway activities for all samples
activities = model.get_pathway_activities(
    ontobj=ontobj,
    dataset='training_data'
)

# 2. Simulate a perturbation
perturbed_activities = model.perturbation(
    ontobj=ontobj,
    dataset='training_data',
    genes=['GENE1', 'GENE2'],
    values=[0, 0],  # Knockout
    output='terms'
)

# 3. Statistical testing
results = ontobj.wilcox_test(
    control=activities,
    perturbed=perturbed_activities,
    direction='up'
)
```

---

## Technical Details

### Loss Function

The VAE uses a combined loss function:

```
Loss = MSE(reconstruction, data) + β × KL(q(z|x) || p(z))
```

Where:
- **MSE**: Mean squared error for reconstruction quality
- **KL**: Kullback-Leibler divergence for latent space regularization
- **β**: Coefficient balancing the two terms (default: 1e-4)

### Ontology Structure Enforcement

OntoVAE enforces biological structure through:

1. **Masked Connections**: Binary masks zero out non-existent biological relationships
2. **Positive Weights**: Decoder weights are clamped to positive values (biological activation)
3. **Hierarchical Architecture**: Layer structure mirrors ontology depth levels
4. **Multiple Neurons**: Each pathway term has multiple neurons (averaging for robustness)

### Reparameterization Trick

To enable backpropagation through stochastic sampling:

```python
z = μ + σ × ε,  where ε ~ N(0, 1)
```

This allows gradients to flow through the encoder.

---

## Dependencies

### Core Dependencies:
- **torch** (≥1.10.0): Deep learning framework
- **pandas** (≥1.1.0): Data manipulation
- **numpy**: Numerical computing
- **goatools** (≥1.0.15): Gene Ontology processing
- **tqdm** (≥4.60.0): Progress bars
- **scipy**: Statistical functions
- **statsmodels**: Multiple testing correction

### Visualization:
- **seaborn** (≥0.11.0): Statistical plotting
- **colorcet** (≥3.0.0): Color palettes
- **matplotlib**: Base plotting

---

## Use Cases

### 1. Pathway Activity Analysis
Extract interpretable pathway-level features from high-dimensional gene expression data.

### 2. Perturbation Modeling
Predict effects of:
- Gene knockouts/overexpression
- Drug treatments
- Genetic mutations

### 3. Disease Mechanism Discovery
Identify dysregulated pathways in disease vs. healthy samples.

### 4. Drug Target Identification
Simulate perturbations to find genes that normalize disease pathways.

---

## Design Patterns and Principles

### 1. **Separation of Concerns**
- `Ontobj`: Data preprocessing and management
- `vae_model`: Model training and inference
- `modules`: Neural network components
- `utils`: Reusable graph operations

### 2. **Slots Pattern**
`Ontobj` uses `__slots__` for memory efficiency (prevents dynamic attribute addition).

### 3. **Modular Architecture**
Different VAE variants (OntoVAE, OntoEncVAE, VAE) share common interfaces.

### 4. **Configuration via Parameters**
Trimming thresholds are used as keys to store multiple ontology versions simultaneously.

### 5. **Hook Pattern**
Uses PyTorch forward hooks to extract intermediate activations for pathway activities.

---

## Limitations and Considerations

1. **Memory Usage**: Large ontologies can create very large mask matrices
2. **Training Time**: Constrained weights (positive, masked) slow convergence
3. **Ontology Coverage**: Genes not in ontology are padded with zeros
4. **Deprecated**: Repository is no longer maintained; see cobra-ai for updates

---

## Example Data

The package includes sample data:
- **pw.obo**: Pathway ontology in OBO format
- **gene_term_mapping.txt**: Gene-to-pathway annotations
- **pbmc_sample_expr.csv**: PBMC gene expression data
- **pbmc_sample_annot.csv**: Sample metadata

---

## Citation

If using OntoVAE, cite:

```
Daria Doncevic, Carl Herrmann, Biologically informed variational autoencoders 
allow predictive modeling of genetic and drug-induced perturbations, 
Bioinformatics, Volume 39, Issue 6, June 2023, btad387, 
https://doi.org/10.1093/bioinformatics/btad387
```

---

## Summary

OntoVAE is a sophisticated framework that bridges bioinformatics and deep learning by:

1. **Encoding biological knowledge** (ontologies) into neural network architecture
2. **Providing interpretable features** (pathway activities) from complex data
3. **Enabling in silico experiments** (perturbation simulations)
4. **Maintaining biological plausibility** through structural constraints

The codebase is well-organized with clear separation between ontology preprocessing (`Ontobj`), model definition (`vae_model`), and neural components (`modules`), making it adaptable for various biological ontologies (GO, HPO, etc.).
