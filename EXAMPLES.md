# OntoVAE Quick Reference & Examples

## Quick Start Guide

### Installation

```bash
# Create conda environment
conda create -n ontovae python=3.7
conda activate ontovae

# Install package
pip install onto-vae

# Or install from source
git clone https://github.com/timoverlaan/onto-vae.git
cd onto-vae
pip install -e .
```

---

## Basic Usage Examples

### Example 1: Complete Workflow with Gene Ontology

```python
from onto_vae.ontobj import Ontobj
from onto_vae.vae_model import OntoVAE
import pandas as pd

# Step 1: Create and initialize ontology object
ontobj = Ontobj(description='GO_BP')
ontobj.initialize_dag(
    obo='data/go-basic.obo',
    gene_annot='data/gene_go_mapping.txt',
    filter_id='biological_process'
)

# Step 2: Trim the ontology
ontobj.trim_dag(top_thresh=1000, bottom_thresh=30)

# Step 3: Create neural network masks
ontobj.create_masks(top_thresh=1000, bottom_thresh=30, module='decoder')

# Step 4: Load and match expression data
expr_data = pd.read_csv('expression_data.csv', index_col=0)
ontobj.match_dataset(
    expr_data=expr_data,
    name='my_dataset',
    top_thresh=1000,
    bottom_thresh=30
)

# Step 5: Initialize and train model
model = OntoVAE(
    ontobj=ontobj,
    dataset='my_dataset',
    top_thresh=1000,
    bottom_thresh=30,
    neuronnum=3,
    drop=0.2,
    z_drop=0.5
)

# Step 6: Train
model.train_model(
    modelpath='best_model.pth',
    lr=1e-4,
    kl_coeff=1e-4,
    batch_size=128,
    epochs=300
)

# Step 7: Extract pathway activities
activities = model.get_pathway_activities(
    ontobj=ontobj,
    dataset='my_dataset'
)
print(f"Pathway activities shape: {activities.shape}")
# Output: (n_samples, n_pathways)
```

---

### Example 2: Gene Knockout Simulation

```python
# Simulate complete knockout of a gene
perturbed_activities = model.perturbation(
    ontobj=ontobj,
    dataset='my_dataset',
    genes=['TP53'],
    values=[0],  # Set to zero (knockout)
    output='terms'  # Get pathway activities
)

# Compare to control
control_activities = model.get_pathway_activities(ontobj, 'my_dataset')

# Find most affected pathways
diff = perturbed_activities - control_activities
avg_diff = diff.mean(axis=0)

# Get pathway annotations
annot = ontobj.extract_annot(top_thresh=1000, bottom_thresh=30)

# Top 10 most downregulated pathways
top_down = annot.iloc[avg_diff.argsort()[:10]]
print(top_down[['ID', 'Name']])

# Top 10 most upregulated pathways
top_up = annot.iloc[avg_diff.argsort()[-10:][::-1]]
print(top_up[['ID', 'Name']])
```

---

### Example 3: Multi-Gene Perturbation (Drug Simulation)

```python
# Simulate a drug that affects multiple genes
drug_targets = ['EGFR', 'BRAF', 'KRAS']
drug_effects = [0.1, 0.2, 0.15]  # Reduced expression

perturbed = model.perturbation(
    ontobj=ontobj,
    dataset='my_dataset',
    genes=drug_targets,
    values=drug_effects,
    output='terms'
)

# Statistical testing
results = ontobj.wilcox_test(
    control=control_activities,
    perturbed=perturbed,
    direction='down',  # Looking for downregulated pathways
    option='terms'
)

# Filter significant results
significant = results[results['qval'] < 0.05]
print(f"Found {len(significant)} significantly affected pathways")
print(significant.head(10))
```

---

### Example 4: Predicting Gene Expression Changes

```python
# Predict how genes respond to a perturbation
perturbed_genes = model.perturbation(
    ontobj=ontobj,
    dataset='my_dataset',
    genes=['MYC'],
    values=[10],  # Overexpression
    output='genes'  # Get gene-level predictions
)

control_genes = model.get_reconstructed_values(ontobj, 'my_dataset')

# Find genes with biggest predicted changes
gene_diff = perturbed_genes - control_genes
avg_gene_diff = gene_diff.mean(axis=0)

genes = ontobj.extract_genes(top_thresh=1000, bottom_thresh=30)

# Top upregulated genes
top_up_idx = avg_gene_diff.argsort()[-20:][::-1]
print("Top predicted upregulated genes:")
for idx in top_up_idx:
    print(f"{genes[idx]}: {avg_gene_diff[idx]:.4f}")
```

---

### Example 5: Visualizing Pathway Activities

```python
import matplotlib.pyplot as plt
import seaborn as sns

# Extract activities
activities = model.get_pathway_activities(ontobj, 'my_dataset')
annot = ontobj.extract_annot(top_thresh=1000, bottom_thresh=30)

# Load sample annotations
sample_annot = pd.read_csv('sample_metadata.csv', index_col=0)

# Scatter plot of two pathways
ontobj.plot_scatter(
    sample_annot=sample_annot,
    color_by='cell_type',
    act=activities,
    term1='immune response',
    term2='cell cycle',
    top_thresh=1000,
    bottom_thresh=30
)
plt.savefig('pathway_scatter.png', dpi=300, bbox_inches='tight')

# Heatmap of pathway activities
top_var_pathways = activities.var(axis=0).argsort()[-50:][::-1]
plt.figure(figsize=(12, 8))
sns.heatmap(
    activities[:, top_var_pathways].T,
    yticklabels=annot.iloc[top_var_pathways]['Name'].values,
    cmap='viridis',
    cbar_kws={'label': 'Activity'}
)
plt.xlabel('Samples')
plt.ylabel('Pathways')
plt.tight_layout()
plt.savefig('pathway_heatmap.png', dpi=300, bbox_inches='tight')
```

---

### Example 6: Comparing Control vs Disease

```python
import numpy as np
from scipy import stats

# Separate samples
control_idx = sample_annot['condition'] == 'healthy'
disease_idx = sample_annot['condition'] == 'disease'

activities = model.get_pathway_activities(ontobj, 'my_dataset')
control_act = activities[control_idx]
disease_act = activities[disease_idx]

# T-test for each pathway
pvals = []
fold_changes = []
for i in range(activities.shape[1]):
    stat, pval = stats.ttest_ind(disease_act[:, i], control_act[:, i])
    pvals.append(pval)
    
    mean_disease = disease_act[:, i].mean()
    mean_control = control_act[:, i].mean()
    fc = mean_disease - mean_control  # Difference in activities
    fold_changes.append(fc)

# Multiple testing correction
from statsmodels.stats.multitest import fdrcorrection
_, qvals = fdrcorrection(pvals)

# Create results dataframe
results = pd.DataFrame({
    'pathway_id': annot['ID'],
    'pathway_name': annot['Name'],
    'fold_change': fold_changes,
    'pval': pvals,
    'qval': qvals
})

# Significant dysregulated pathways
sig_results = results[results['qval'] < 0.05].sort_values('fold_change')
print(f"Found {len(sig_results)} significantly dysregulated pathways")
print("\nTop downregulated:")
print(sig_results.head(10)[['pathway_name', 'fold_change', 'qval']])
print("\nTop upregulated:")
print(sig_results.tail(10)[['pathway_name', 'fold_change', 'qval']])
```

---

### Example 7: Using Encoder Variant (OntoEncVAE)

```python
from onto_vae.vae_model import OntoEncVAE

# Create masks for encoder architecture
ontobj.create_masks(top_thresh=1000, bottom_thresh=30, module='encoder')

# Initialize encoder-based model
model_enc = OntoEncVAE(
    ontobj=ontobj,
    dataset='my_dataset',
    top_thresh=1000,
    bottom_thresh=30,
    neuronnum=3,
    drop=0.2,
    z_drop=0.5
)

# Train (same interface)
model_enc.train_model(
    modelpath='encoder_model.pth',
    lr=1e-4,
    kl_coeff=1e-4,
    batch_size=128,
    epochs=300
)

# Use (same interface)
activities = model_enc.get_pathway_activities(ontobj, 'my_dataset')
```

---

### Example 8: Working with Human Phenotype Ontology (HPO)

```python
# HPO for disease/phenotype associations
ontobj_hpo = Ontobj(description='HPO')
ontobj_hpo.initialize_dag(
    obo='hp.obo',
    gene_annot='genes_to_phenotype.txt'
    # No filter_id needed for HPO
)

# Rest of workflow is identical
ontobj_hpo.trim_dag(top_thresh=500, bottom_thresh=20)
ontobj_hpo.create_masks(top_thresh=500, bottom_thresh=20, module='decoder')
ontobj_hpo.match_dataset(expr_data, 'my_dataset', 500, 20)

model_hpo = OntoVAE(ontobj_hpo, 'my_dataset', 500, 20)
# ... train and use
```

---

### Example 9: Computing Semantic Similarities

```python
# Compute Wang semantic similarities
ontobj.compute_wsem_sim(
    obo='data/go-basic.obo',
    top_thresh=1000,
    bottom_thresh=30
)

# Access similarity matrix
sem_sim = ontobj.sem_sim['1000_30']
print(f"Semantic similarity matrix shape: {sem_sim.shape}")

# Find most similar pathways to a given pathway
annot = ontobj.extract_annot(1000, 30)
pathway_idx = 42  # Example index
similarities = sem_sim[pathway_idx]
most_similar = similarities.argsort()[-10:][::-1]

print(f"Pathways most similar to {annot.iloc[pathway_idx]['Name']}:")
for idx in most_similar[1:]:  # Skip self
    print(f"  {annot.iloc[idx]['Name']}: {similarities[idx]:.3f}")
```

---

### Example 10: Loading Pre-trained Models

```python
import torch

# Load saved model
checkpoint = torch.load('best_model.pth')

# Initialize model with same configuration
model = OntoVAE(
    ontobj=ontobj,
    dataset='my_dataset',
    top_thresh=1000,
    bottom_thresh=30,
    neuronnum=3
)

# Load weights
model.load_state_dict(checkpoint['model_state_dict'])
model.eval()  # Set to evaluation mode

# Use for inference
activities = model.get_pathway_activities(ontobj, 'new_dataset')
```

---

## Common Parameter Configurations

### Small Dataset (<1000 samples)
```python
model = OntoVAE(
    ontobj=ontobj,
    dataset='small_data',
    neuronnum=2,        # Fewer neurons per term
    drop=0.3,           # Higher dropout
    z_drop=0.5
)

model.train_model(
    modelpath='model.pth',
    lr=1e-3,            # Higher learning rate
    kl_coeff=1e-5,      # Lower KL weight
    batch_size=64,      # Smaller batches
    epochs=500          # More epochs
)
```

### Large Dataset (>10,000 samples)
```python
model = OntoVAE(
    ontobj=ontobj,
    dataset='large_data',
    neuronnum=5,        # More neurons per term
    drop=0.1,           # Lower dropout
    z_drop=0.3
)

model.train_model(
    modelpath='model.pth',
    lr=5e-5,            # Lower learning rate
    kl_coeff=1e-4,
    batch_size=256,     # Larger batches
    epochs=200          # Fewer epochs needed
)
```

### High-Dimensional Data (>20,000 genes)
```python
# More aggressive trimming
ontobj.trim_dag(top_thresh=500, bottom_thresh=50)

model = OntoVAE(
    ontobj=ontobj,
    dataset='high_dim_data',
    neuronnum=3,
    drop=0.2,
    z_drop=0.5
)
```

---

## Troubleshooting Guide

### Issue: Model not training (loss not decreasing)

**Possible causes and solutions:**

1. **Learning rate too high**
   ```python
   # Try: lr=1e-5 instead of lr=1e-4
   ```

2. **KL coefficient too high**
   ```python
   # Try: kl_coeff=1e-5 instead of kl_coeff=1e-4
   ```

3. **Data not normalized**
   ```python
   # Normalize before matching
   expr_data = (expr_data - expr_data.mean()) / expr_data.std()
   ```

### Issue: All pathway activities near zero

**Solution: Check mask creation**
```python
# Verify masks exist and are non-empty
masks = ontobj.masks['1000_30']['decoder']
for i, mask in enumerate(masks):
    print(f"Layer {i}: {mask.sum()} connections")
    # Should have many non-zero connections
```

### Issue: Out of memory

**Solutions:**

1. **Reduce batch size**
   ```python
   batch_size=64  # Instead of 128 or 256
   ```

2. **Trim more aggressively**
   ```python
   ontobj.trim_dag(top_thresh=500, bottom_thresh=50)
   ```

3. **Use CPU if GPU memory limited**
   ```python
   # Model automatically uses CPU if CUDA unavailable
   import os
   os.environ['CUDA_VISIBLE_DEVICES'] = ''  # Force CPU
   ```

### Issue: Reconstruction quality poor

**Solution: Adjust architecture**
```python
# Add encoder/decoder layers (not directly supported, need to modify modules.py)
# Or adjust neuronnum
model = OntoVAE(..., neuronnum=5)  # More capacity
```

---

## Performance Benchmarks

### Typical Training Times (on GPU)

| Dataset Size | Ontology Size | Epochs | Time per Epoch | Total Time |
|--------------|---------------|--------|----------------|------------|
| 500 samples  | 300 pathways  | 300    | ~2 seconds     | ~10 min    |
| 2,000 samples| 500 pathways  | 300    | ~5 seconds     | ~25 min    |
| 10,000 samples| 800 pathways | 200    | ~20 seconds    | ~67 min    |
| 50,000 samples| 1000 pathways| 100    | ~120 seconds   | ~200 min   |

*Note: Times vary based on GPU, gene count, and hyperparameters*

### Memory Requirements

| Configuration | Model Size | Peak Memory |
|--------------|------------|-------------|
| Small (300 pathways, 5K genes) | ~15 MB | ~500 MB |
| Medium (500 pathways, 10K genes) | ~40 MB | ~1.5 GB |
| Large (1000 pathways, 20K genes) | ~100 MB | ~4 GB |

---

## Best Practices

### 1. Data Preparation
- **Normalize** expression data before matching
- **Remove batch effects** if combining multiple datasets
- **Filter low-quality genes** (e.g., low variance)

### 2. Ontology Trimming
- **Start conservative**: Use wider thresholds first
- **Iterate**: Try different thresholds and evaluate
- **Balance**: Too few terms = loss of detail; too many = overfitting

### 3. Training
- **Monitor both losses**: reconstruction and KL
- **Save best model**: Based on validation loss
- **Use early stopping**: If validation loss plateaus

### 4. Evaluation
- **Cross-validation**: Train multiple models with different splits
- **Compare to baseline**: Standard VAE without ontology
- **Biological validation**: Do results make biological sense?

### 5. Interpretation
- **Multiple testing correction**: Always use FDR correction
- **Effect sizes**: Don't just rely on p-values
- **Pathway databases**: Cross-reference with known biology

---

## API Reference Cheat Sheet

### Ontobj Methods

```python
# Initialization
ontobj.initialize_dag(obo, gene_annot, filter_id)

# Preprocessing
ontobj.trim_dag(top_thresh, bottom_thresh)
ontobj.create_masks(top_thresh, bottom_thresh, module)

# Data management
ontobj.match_dataset(expr_data, name, top_thresh, bottom_thresh)
ontobj.add_dataset(dataset, name, top_thresh, bottom_thresh)

# Retrieval
ontobj.extract_annot(top_thresh, bottom_thresh)
ontobj.extract_genes(top_thresh, bottom_thresh)
ontobj.extract_dataset(dataset, top_thresh, bottom_thresh)

# Analysis
ontobj.compute_wsem_sim(obo, top_thresh, bottom_thresh)
ontobj.wilcox_test(control, perturbed, direction, option)
ontobj.plot_scatter(sample_annot, color_by, act, term1, term2)
```

### Model Methods

```python
# Training
model.train_model(modelpath, lr, kl_coeff, batch_size, epochs, run)

# Inference
model.forward(x)  # Returns (reconstruction, mu, log_var)
model.get_embedding(x)  # Returns latent representation

# Analysis
model.get_pathway_activities(ontobj, dataset, terms)
model.get_reconstructed_values(ontobj, dataset, rec_genes)
model.perturbation(ontobj, dataset, genes, values, output, terms, rec_genes)
```

---

## Additional Resources

### Sample Data
- Pre-processed ontologies: https://figshare.com/projects/OntoVAE_Ontology_guided_VAE_manuscript/146727
- Includes GO, HPO objects and pre-trained models

### Related Projects
- **COBRA-AI**: https://github.com/hdsu-bioquant/cobra-ai (Maintained version)
- **Gene Ontology**: http://geneontology.org/
- **Human Phenotype Ontology**: https://hpo.jax.org/

### Paper
- Full methodology: https://academic.oup.com/bioinformatics/article/39/6/btad387/7199588

---

This quick reference provides practical examples and solutions for common OntoVAE use cases!
