# OntoVAE Architecture Guide

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         OntoVAE System                          │
└─────────────────────────────────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐      ┌──────────────┐      ┌─────────────┐
│  Ontobj       │      │  VAE Models  │      │  Modules    │
│  (Ontology    │──────│  (Training & │◄─────│  (Neural    │
│   Processor)  │      │   Inference) │      │   Network)  │
└───────────────┘      └──────────────┘      └─────────────┘
        │                      │                      │
        │                      │                      │
        ▼                      ▼                      ▼
    Data Layer            Model Layer          Component Layer
```

---

## OntoVAE Model Architecture (Decoder Variant)

### Complete Information Flow

```
Input: Gene Expression (N samples × M genes)
        │
        ▼
┌───────────────────────────────────────────────────┐
│                    ENCODER                        │
│  ┌─────────────────────────────────────────────┐  │
│  │ Input Layer (M genes)                       │  │
│  │         ↓                                   │  │
│  │ Hidden Layer 1 (Fully Connected)           │  │
│  │         ↓ (BatchNorm + Dropout + ReLU)     │  │
│  │ Hidden Layer 2 (Optional)                  │  │
│  │         ↓ (BatchNorm + Dropout + ReLU)     │  │
│  │    ...                                      │  │
│  │         ↓                                   │  │
│  │ Latent Parameters                          │  │
│  │    ├─── μ (mean)                          │  │
│  │    └─── log(σ²) (log variance)           │  │
│  └─────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────┘
        │
        ▼
    Sampling (Reparameterization Trick)
    z = μ + σ·ε,  ε ~ N(0,1)
        │
        ▼
┌───────────────────────────────────────────────────┐
│              ONTOLOGY DECODER                     │
│                                                   │
│  Latent Space (z)                                │
│         ↓                                        │
│  ┌────────────────────────────────────┐          │
│  │ Depth 0 Terms (Root pathways)      │          │
│  │ [neuronnum × num_terms_depth0]     │          │
│  └────────────────────────────────────┘          │
│         ↓ (Masked connections ⊙)                │
│  ┌────────────────────────────────────┐          │
│  │ Depth 1 Terms (Child pathways)     │          │
│  │ [neuronnum × num_terms_depth1]     │          │
│  └────────────────────────────────────┘          │
│         ↓ (Masked connections ⊙)                │
│  ┌────────────────────────────────────┐          │
│  │ Depth 2 Terms                      │          │
│  └────────────────────────────────────┘          │
│         ⋮                                        │
│         ↓ (Masked connections ⊙)                │
│  ┌────────────────────────────────────┐          │
│  │ Gene Layer (Depth max)             │          │
│  │ [M genes]                          │          │
│  └────────────────────────────────────┘          │
│                                                   │
└───────────────────────────────────────────────────┘
        │
        ▼
Output: Reconstructed Gene Expression (N × M)

Legend:
⊙ = Element-wise multiplication with binary mask
```

---

## Ontology Structure Integration

### How the DAG becomes a Neural Network

**Original Ontology (Example):**
```
                    [Root: Biological Process]
                            │
            ┌───────────────┼───────────────┐
            │               │               │
      [Metabolism]   [Cell Cycle]   [Immune Response]
            │               │               │
      ┌─────┴─────┐   ┌────┴────┐      ├────────┐
      │           │   │         │      │        │
[Glycolysis] [Krebs] [Mitosis] [Cytokinesis] [T-cell] [B-cell]
      │           │   │         │      │        │
    Genes      Genes Genes   Genes  Genes    Genes
```

**After Preprocessing:**
1. **Trimming**: Remove terms with >1000 or <30 genes
2. **Depth Calculation**: Assign depth to each term
3. **Mask Creation**: Binary matrix for each depth transition

**Resulting Neural Network Layers:**
```
Layer 0 (Depth 0): [Biological Process] → 1 term × 3 neurons = 3 neurons
Layer 1 (Depth 1): [Metabolism, Cell Cycle, Immune Response] → 3 terms × 3 neurons = 9 neurons
Layer 2 (Depth 2): [Glycolysis, Krebs, Mitosis, ...] → K terms × 3 neurons = 3K neurons
...
Final Layer: All genes → M neurons
```

### Mask Matrix Example

For transition from Depth 1 to Depth 2:

```
              Glycolysis  Krebs  Mitosis  Cytokinesis  T-cell  B-cell
Metabolism    [   1        1       0          0         0       0   ]
Cell Cycle    [   0        0       1          1         0       0   ]
Immune Resp   [   0        0       0          0         1       1   ]

1 = connection allowed (biological relationship exists)
0 = connection forbidden (no relationship)
```

This mask is repeated for each neuron (3×3=9 blocks for neuronnum=3).

---

## Training Process

### Forward Pass

```python
# 1. Encode
x → Encoder → (μ, log_var)

# 2. Sample latent representation
z = reparameterize(μ, log_var)

# 3. Decode with ontology structure
z → OntoDecoder → x̂

# 4. Compute loss
reconstruction_loss = MSE(x̂, x)
kl_loss = KL_divergence(μ, log_var)
total_loss = reconstruction_loss + β × kl_loss
```

### Backward Pass with Constraints

```python
# 1. Compute gradients
loss.backward()

# 2. Apply masks to gradients (enforce structure)
for layer in decoder:
    layer.weight.grad = layer.weight.grad ⊙ mask

# 3. Update weights
optimizer.step()

# 4. Enforce positive weights (biological interpretation)
for layer in decoder:
    layer.weight.data = max(0, layer.weight.data)
```

---

## Class Relationships

```
┌────────────────────────────────────────────────────────┐
│                     Ontobj                             │
├────────────────────────────────────────────────────────┤
│ Attributes:                                            │
│  - annot_base, genes_base, graph_base                 │
│  - annot[thresh], genes[thresh], graph[thresh]        │
│  - masks[thresh][module]                              │
│  - data[thresh][dataset]                              │
│  - desc_genes[thresh], sem_sim[thresh]                │
├────────────────────────────────────────────────────────┤
│ Methods:                                               │
│  + initialize_dag()                                    │
│  + trim_dag()                                          │
│  + create_masks()                                      │
│  + match_dataset()                                     │
│  + compute_wsem_sim()                                  │
└────────────────────────────────────────────────────────┘
                        │
                        │ provides data to
                        ▼
┌────────────────────────────────────────────────────────┐
│               OntoVAE (nn.Module)                      │
├────────────────────────────────────────────────────────┤
│ Attributes:                                            │
│  - encoder: Encoder                                    │
│  - decoder: OntoDecoder                                │
│  - mask_list: List[Tensor]                            │
│  - X: ndarray                                          │
├────────────────────────────────────────────────────────┤
│ Methods:                                               │
│  + forward()                                           │
│  + train_model()                                       │
│  + get_pathway_activities()                            │
│  + perturbation()                                      │
└────────────────────────────────────────────────────────┘
                        │
                        │ composed of
                        ▼
        ┌───────────────────────────────┐
        │                               │
        ▼                               ▼
┌──────────────┐              ┌──────────────────┐
│   Encoder    │              │  OntoDecoder     │
│  (Module)    │              │   (Module)       │
├──────────────┤              ├──────────────────┤
│ - encoder:   │              │ - decoder:       │
│   ModuleList │              │   ModuleList     │
│ - mu: Linear │              │ - masks: List    │
│ - logvar:    │              │                  │
│   Linear     │              │                  │
└──────────────┘              └──────────────────┘
```

---

## Data Flow for Pathway Activity Extraction

```
Input Data (N × M)
        │
        ▼
    Encoder
        │
        ▼
  Latent z (N × latent_dim)
        │
        │──────────────────┐
        ▼                  │
    Decoder Layer 0        │ Register
        │                  │ Forward
        ▼                  │ Hooks
    Decoder Layer 1        │
        │                  │
        ▼                  │
    Decoder Layer 2        │
        │                  │
        ⋮                  │
        │                  │
        ▼                  │
    Output Layer           │
                           │
        ┌──────────────────┘
        │
        ▼
Extract Activations from Each Layer
        │
        ▼
Average neuronnum neurons per term
        │
        ▼
Pathway Activities (N × num_terms)
```

**Hooks Mechanism:**
```python
activation = {}

def get_activation(index):
    def hook(model, input, output):
        activation[index] = output.detach()
    return hook

# Register hooks
for i, layer in enumerate(decoder.layers):
    layer.register_forward_hook(get_activation(i))

# Run forward pass
model(data)

# Extract activities
activities = concatenate(activation.values())
```

---

## Perturbation Simulation Architecture

```
Original Expression Data
        │
        ▼
┌──────────────────────────────────┐
│  Modify Gene Values              │
│  data[:, gene_indices] = values  │
└──────────────────────────────────┘
        │
        ▼
    Forward Pass
        │
        ├─────────────┬─────────────┐
        │             │             │
        ▼             ▼             ▼
  Pathway        Latent        Reconstructed
  Activities     Space         Gene Values
  (terms)          z              x̂
```

**Use Cases:**
1. **Knockout**: Set gene values to 0
2. **Overexpression**: Set to high values
3. **Drug effect**: Set to known drug-induced changes
4. **Multiple perturbations**: Modify multiple genes simultaneously

---

## Memory and Computation

### Memory Usage

**Mask Storage:**
For an ontology with:
- 500 terms across 5 depth levels
- neuronnum = 3

Each mask is approximately:
```
Layer k→k+1: (terms_k × 3) × (terms_k+1 × 3) × 4 bytes
Example: (100 × 3) × (150 × 3) × 4 = 540,000 bytes ≈ 0.5 MB
Total for all layers: ~2-5 MB
```

**Model Parameters:**
```
Encoder: M → hidden → latent
Decoder: latent → terms × neuronnum → M

Total parameters ≈ M × hidden + (sum of term counts) × neuronnum²
Example: 10,000 genes, 500 terms, neuronnum=3
≈ 10,000 × 1,000 + 500 × 9 = ~10M parameters ≈ 40 MB
```

### Computational Complexity

**Forward Pass:**
- Encoder: O(M × latent_dim)
- Decoder: O(latent_dim × sum(term_counts × neuronnum))
- Total: O(M × L) where L is latent dimension

**Backward Pass:**
- Same as forward + mask operations
- Mask operations add minimal overhead (element-wise multiplication)

---

## Code Organization Philosophy

### 1. **Single Responsibility**
- `Ontobj`: Only handles ontology preprocessing
- `vae_model`: Only handles model training/inference
- `modules`: Only defines neural components
- `utils`: Only provides reusable utilities

### 2. **Composition over Inheritance**
- VAE models compose Encoder and Decoder modules
- Different VAE variants share the same building blocks

### 3. **Configuration through Parameters**
- Trimming thresholds as dictionary keys
- Flexible module selection (encoder/decoder)
- Hyperparameter control via constructor

### 4. **Immutability where Possible**
- Base ontology preserved (`_base` attributes)
- Derived versions stored separately

---

## Extension Points

### Adding New VAE Variants

```python
class CustomVAE(nn.Module):
    def __init__(self, ontobj, dataset, ...):
        # 1. Extract masks and data from Ontobj
        self.masks = ontobj.masks[...]
        self.X = ontobj.data[...]
        
        # 2. Define encoder and decoder
        self.encoder = CustomEncoder(...)
        self.decoder = CustomDecoder(...)
    
    def forward(self, x):
        # 3. Implement forward pass
        pass
    
    def train_model(self, ...):
        # 4. Implement training logic
        pass
```

### Adding New Ontologies

```python
# 1. Prepare OBO file and gene mapping
# 2. Initialize
ontobj = Ontobj(description='Custom_Ontology')
ontobj.initialize_dag(obo='custom.obo', gene_annot='mapping.txt')

# 3. Trim and create masks
ontobj.trim_dag(...)
ontobj.create_masks(...)

# 4. Use with any VAE variant
model = OntoVAE(ontobj, ...)
```

### Custom Analysis Functions

Add to `Ontobj`:
```python
def custom_analysis(self, ...):
    # Access preprocessed data
    annot = self.annot[thresh]
    genes = self.genes[thresh]
    # Perform analysis
    ...
```

---

## Performance Optimization Tips

### 1. **Data Loading**
```python
# Use FastTensorDataLoader instead of standard DataLoader
from onto_vae.fast_data_loader import FastTensorDataLoader

loader = FastTensorDataLoader(X_train, batch_size=128, shuffle=True)
# ~10x faster than torch.utils.data.DataLoader
```

### 2. **GPU Utilization**
```python
# Model automatically detects and uses GPU if available
device = 'cuda' if torch.cuda.is_available() else 'cpu'
# All tensors moved to device during training
```

### 3. **Batch Size Selection**
- Larger batches: Better GPU utilization, less noise in gradients
- Smaller batches: More updates, better generalization
- Recommended: 128-256 for typical datasets

### 4. **Masking Efficiency**
- Masks are precomputed and stored as tensors
- Element-wise multiplication is highly optimized on GPU
- Minimal overhead compared to standard layers

---

## Common Pitfalls and Solutions

### 1. **Dimension Mismatch**
**Problem:** Gene counts don't match between data and ontology
**Solution:** `match_dataset()` automatically aligns and zero-pads

### 2. **Memory Issues with Large Ontologies**
**Problem:** Too many terms → huge masks
**Solution:** Increase `bottom_thresh` to remove more terms

### 3. **Training Instability**
**Problem:** Loss oscillates or diverges
**Solution:** 
- Reduce learning rate
- Adjust `kl_coeff` (try 1e-5 to 1e-3)
- Increase dropout

### 4. **Vanishing Pathway Activities**
**Problem:** All pathway activities near zero
**Solution:**
- Check positive weight constraints
- Verify mask correctness
- Ensure sufficient training epochs

---

## Testing and Validation

### Model Quality Checks

```python
# 1. Reconstruction quality
rec = model.get_reconstructed_values(ontobj, 'test_data')
mse = np.mean((original - rec) ** 2)

# 2. Pathway activity variance
act = model.get_pathway_activities(ontobj, 'test_data')
variance = np.var(act, axis=0)  # Should not all be near zero

# 3. Perturbation sensitivity
control = model.get_pathway_activities(ontobj, 'data')
perturbed = model.perturbation(ontobj, 'data', ['GENE1'], [0])
diff = np.abs(control - perturbed)
np.max(diff, axis=1)  # Per-sample max change
```

---

This architecture document provides a comprehensive view of how OntoVAE integrates biological knowledge with deep learning through careful design of data structures, neural network architecture, and training procedures.
