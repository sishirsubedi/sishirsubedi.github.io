---
layout: page
title: Data embedding in representation learning
description:
img: assets/img/proj2_thumb.png
importance: 1
category: data-science
related_publications: false
---

<span style="color:#2a9d8f; font-size:30px; font-weight:bold;">
Data embedding in representation learning
</span>

The genomic data is ever-increasing in size and complexity. Technological advancements have enabled us to generate multimodal omics data from millions of cells. This large-scale data poses many challenges for researchers to extract true biological signals for discovery. 

One approach to systematically learning complex biological data is to take advantage of the inherent features of biological mechanisms, i.e., genomic features often act in modules. We are interested in identifying hidden patterns that represent abstract concepts in the data and then linking those concepts to known biological modules or pathways using prior knowledge of genomic features.

<span style="color:#2a9d8f; font-size:20px; font-weight:bold;">
How do we learn abstract concepts from biological data?
</span>

The latent variable model (LVM) provides a framework for learning a set of low-dimensional hidden variables from empirically measured high-dimensional data.

The computational methods used for latent variable modelling can be broadly classified into two groups - **linear and nonlinear models**. The simple linear methods include Principle Component Analysis (PCA), while complex models consist of probabilistic matrix factorization and Latent Dirichlet Allocation (LDA) models. The nonlinear models are generally based on neural networks, where autoencoder or variational autoencoders (VAEs) represent simpler models, and complex models include deep generative networks such as generative adversarial networks (GANs), graph neural networks (GNNs), attention networks, diffusion networks, and large language-based foundational models. Both linear and nonlinear models can be designed within the Bayesian framework for enhanced interpretability and the incorporation of prior knowledge.

<div class="row">
    <div style="width: 50%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj1_lvm.png" title="lvm image" class="img-fluid rounded z-depth-1"
 %}
    </div>
</div>


<span style="color:#2a9d8f; font-size:20px; font-weight:bold;">
Data embedding
</span>

We aim to transform cell embedding in gene space (cell x gene) to factor space (cell x factor), where each factor represents an abstract biological concept.

- **Matrix factorization**: We approximate the count data matrix by learning two low-dimensional factor matrices that describe the structure of factors across cells and a weight matrix that specifies the contribution of each gene to inferred factors. This is a single transformation from gene space to factor space. 
  
- **Probabilistic Matrix Factorization**: Adding parametric layers is desirable for modeling gene count data, as it handles non-negative sparse datasets with high noise levels and missing values. This provides a multi-layer transformation from gene space to factor space. 
  
- **Neural networks**: Linear approximation of latent factors may not fully capture the complex structure of biological data. A neural network applies multiple nonlinear transformations and learns an abstract feature representation that captures the most informative structure. Additionally, graph layers incorporate topological information of cells into the model, while embedding layers provide better interpretability of latent spaces. 
  
- **Attention networks**: In the above methods, we directly embedded cells in latent space. Instead, we add a step of embedding for each gene using attention networks, such that relationships among genes are incorporated into the latent space.
  
- **Foundation networks**: Neural networks with attention layers, as above, do not generalize across datasets. To learn a robust and generalizable network, we make two important changes: First, we assign an identity to each gene and learn identity-specific embeddings in addition to expression embedding. Second, we also assign an identity to each cell, including its cell type and disease state, and include its embedding in the training. Such a model would be "foundational" in nature, and learned parameters can be applied across different biological experiments. 

The following table provides a summary of different data embedding techniques used in single-cell data analysis.

<div class="row">
 {% include figure.liquid loading="eager" path="assets/img/proj2_overview.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>

