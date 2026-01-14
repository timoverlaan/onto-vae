# Documentation Navigation Guide

This repository now includes comprehensive documentation to help you understand and use the OntoVAE codebase.

## Documentation Files

### 📘 [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md)
**Start here for an overview of the entire codebase**

- Project structure and organization
- Detailed explanation of all core components
- Key algorithms and workflows
- Design patterns and principles
- Technical implementation details
- Dependencies and use cases

**Best for:** Understanding what OntoVAE does and how it's structured

---

### 🏗️ [ARCHITECTURE.md](ARCHITECTURE.md)
**Dive deep into the system architecture**

- High-level system architecture diagrams
- Detailed model architecture (encoder, decoder, ontology integration)
- Data flow diagrams
- Memory and computational complexity analysis
- Extension points for customization
- Performance optimization tips

**Best for:** Understanding how OntoVAE works internally and how to extend it

---

### 💻 [EXAMPLES.md](EXAMPLES.md)
**Get started quickly with practical examples**

- Quick start installation guide
- 10 complete code examples covering:
  - Basic workflow
  - Gene knockout simulations
  - Multi-gene perturbations
  - Visualization
  - Statistical testing
  - Working with different ontologies
- Common parameter configurations
- Troubleshooting guide
- API reference cheat sheet

**Best for:** Implementing OntoVAE for your specific use case

---

### 📖 [README.md](README.md)
**Original project documentation**

- Project overview and citation
- Installation instructions
- Basic usage
- Link to vignette notebook
- Link to pre-trained models and data

**Best for:** Quick project overview and installation

---

### 📓 [vignette.ipynb](vignette.ipynb)
**Interactive tutorial notebook**

- Step-by-step walkthrough
- Working example with real data
- Visualizations and analysis

**Best for:** Hands-on learning and experimentation

---

## Quick Navigation

### I want to...

**...understand what OntoVAE is**
→ Start with [README.md](README.md), then [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md)

**...use OntoVAE for my research**
→ Go to [EXAMPLES.md](EXAMPLES.md) and find a similar use case

**...understand how it works under the hood**
→ Read [ARCHITECTURE.md](ARCHITECTURE.md)

**...modify or extend OntoVAE**
→ Read [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) and [ARCHITECTURE.md](ARCHITECTURE.md)

**...troubleshoot an issue**
→ Check the troubleshooting section in [EXAMPLES.md](EXAMPLES.md)

**...see it in action**
→ Open [vignette.ipynb](vignette.ipynb)

---

## Documentation Overview Table

| Document | Length | Technical Level | Purpose |
|----------|--------|----------------|---------|
| README.md | Short | Beginner | Project introduction |
| CODEBASE_EXPLANATION.md | Long | Intermediate | Complete codebase overview |
| ARCHITECTURE.md | Long | Advanced | System design and internals |
| EXAMPLES.md | Medium | Beginner-Intermediate | Practical usage guide |
| vignette.ipynb | Medium | Beginner | Interactive tutorial |

---

## Key Concepts Summary

### What is OntoVAE?
A Variational Autoencoder that integrates biological ontologies (like Gene Ontology) into its neural network architecture, enabling:
- Interpretable pathway-level features from gene expression data
- Simulation of genetic and drug perturbations
- Biologically-informed dimensionality reduction

### Main Components
1. **Ontobj**: Preprocesses ontologies and manages data
2. **OntoVAE**: VAE with ontology in decoder (main model)
3. **OntoEncVAE**: VAE with ontology in encoder (alternative)
4. **VAE**: Standard VAE baseline

### Typical Workflow
```
Load Ontology → Trim → Create Masks → Match Data → Train Model → Extract Activities → Analyze
```

---

## Code Organization

```
onto-vae/
├── Documentation (you are here)
│   ├── README.md                 # Project overview
│   ├── CODEBASE_EXPLANATION.md   # Complete explanation
│   ├── ARCHITECTURE.md           # System design
│   ├── EXAMPLES.md               # Practical guide
│   └── vignette.ipynb           # Interactive tutorial
│
├── Source Code
│   └── onto_vae/
│       ├── ontobj.py            # Ontology preprocessing
│       ├── vae_model.py         # Model implementations
│       ├── modules.py           # Neural network components
│       ├── utils.py             # Utility functions
│       └── fast_data_loader.py  # Optimized data loader
│
├── Configuration
│   ├── pyproject.toml           # Poetry config
│   └── requirements.txt         # Pip dependencies
│
└── Data
    └── onto_vae/data/           # Sample data files
```

---

## Learning Path Recommendations

### For Biologists/Bioinformaticians
1. Read [README.md](README.md) - understand the motivation
2. Open [vignette.ipynb](vignette.ipynb) - see it work with real data
3. Check [EXAMPLES.md](EXAMPLES.md) - adapt examples to your data
4. Reference [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) - understand the methods

### For Machine Learning Researchers
1. Read [README.md](README.md) - understand the problem
2. Read [ARCHITECTURE.md](ARCHITECTURE.md) - see the architecture
3. Read [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) - implementation details
4. Check [EXAMPLES.md](EXAMPLES.md) - see how to use it

### For Software Engineers
1. Read [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) - code structure
2. Read [ARCHITECTURE.md](ARCHITECTURE.md) - design patterns
3. Check source code with documentation as reference
4. Use [EXAMPLES.md](EXAMPLES.md) for testing

---

## Finding Information

### About Ontology Processing
- High-level: [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) → Section "Ontobj Class"
- Details: [ARCHITECTURE.md](ARCHITECTURE.md) → Section "Ontology Structure Integration"
- Example: [EXAMPLES.md](EXAMPLES.md) → Example 1

### About Model Training
- High-level: [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) → Section "VAE Model Classes"
- Details: [ARCHITECTURE.md](ARCHITECTURE.md) → Section "Training Process"
- Example: [EXAMPLES.md](EXAMPLES.md) → Example 1, Small/Large Dataset configs

### About Perturbation Simulation
- High-level: [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) → Methods section
- Details: [ARCHITECTURE.md](ARCHITECTURE.md) → Section "Perturbation Simulation"
- Example: [EXAMPLES.md](EXAMPLES.md) → Examples 2, 3, 4

### About Different Ontologies
- High-level: [CODEBASE_EXPLANATION.md](CODEBASE_EXPLANATION.md) → Use Cases
- Details: [ARCHITECTURE.md](ARCHITECTURE.md) → Extension Points
- Example: [EXAMPLES.md](EXAMPLES.md) → Example 8 (HPO)

---

## Additional Resources

### External Links
- **Paper**: [Bioinformatics 2023](https://academic.oup.com/bioinformatics/article/39/6/btad387/7199588)
- **Maintained Version**: [COBRA-AI](https://github.com/hdsu-bioquant/cobra-ai)
- **Pre-trained Models**: [Figshare](https://figshare.com/projects/OntoVAE_Ontology_guided_VAE_manuscript/146727)
- **Gene Ontology**: [geneontology.org](http://geneontology.org/)
- **HPO**: [hpo.jax.org](https://hpo.jax.org/)

### Getting Help
1. Check the troubleshooting guide in [EXAMPLES.md](EXAMPLES.md)
2. Review the relevant documentation section
3. Look at the original paper for methodology
4. Check the COBRA-AI project (maintained version)

---

## Citation

If you use OntoVAE in your research, please cite:

```bibtex
@article{doncevic2023biologically,
  title={Biologically informed variational autoencoders allow predictive modeling of genetic and drug-induced perturbations},
  author={Doncevic, Daria and Herrmann, Carl},
  journal={Bioinformatics},
  volume={39},
  number={6},
  pages={btad387},
  year={2023},
  publisher={Oxford University Press}
}
```

---

## Version Information

- **Python**: 3.7+
- **PyTorch**: 1.10.0+
- **Main Dependencies**: pandas, goatools, torch, tqdm, seaborn
- **License**: GPL-3.0

---

## Maintenance Status

⚠️ **Note**: This repository is **no longer actively maintained**. 

For the latest version and ongoing development, see:
- [COBRA-AI Repository](https://github.com/hdsu-bioquant/cobra-ai)

This documentation was created to help users understand and utilize the existing OntoVAE codebase.

---

**Last Updated**: January 2026

**Documentation Created By**: GitHub Copilot Workspace

**Purpose**: Comprehensive explanation of the OntoVAE codebase for users, researchers, and developers
