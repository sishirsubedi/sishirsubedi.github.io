---
layout: page
title: AttnCell
description:
img: assets/img/proj3_thumb.png
importance: 1
category: project-ideas
related_publications: false
---

<span style="color:#2a9d8f; font-size:30px; font-weight:bold;">
Dual embedded neural networks for biological interpretability
</span>

The biological interpretability of deep learning models has been a challenging and intriguing problem. 

The primary objective of this project is to evaluate a deep learning model with robust biological interpretability.

A model, named as AttnCell ( model explains where to pay "attention" from "cell" data ) is focused on learning the following biological information:
- gene interaction networks
- activity of pathways in a cell
- contribution of genes in pathways


<div class="row">
 {% include figure.liquid loading="eager" path="assets/img/proj3_overview.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
<div class="caption">
 Overview of AttnCell framework.
</div>

We train the model using a variational inference algorithm and maximize the evidence lower bound (ELBO). We have two losses in the model: the Kullback-Leibler (KL) loss and the Multinomial-Dirichlet likelihood loss, both of which are in the decoder network. 

Preliminary results using normal pancreas data from [Seurat](https://satijalab.org/seurat/archive/v3.2/integration.html).


- Interpretability1: Gene interaction networks are learned by the attention module in the model. The figure illustrates examples of known marker genes, including NPTX2/DLK1 for beta cells, LEPR/RBP4 for delta cells, and PLA2G1B/CPA1 for acinar cells, in the human pancreas.

<div class="row">
    <div style="width: 100%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj3_genenetwork.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
 Gene network learned by the model.
</div>

- Interpretability2: The activity of pathways in cells, which generally determines cell type identity, can be captured by analyzing the UMAP representation of cells based on the latent space learned by the model. Here, cell type-specific clusters suggest that the factors learned by the model are biologically relevant. 


<div class="row">
    <div style="width: 100%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj3_celltopic.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
 Latent space representation learned by the model.
</div>


- Interpretability3: Contribution of genes in pathways learned as factors/topics in the model. We can further investigate different gene activity in each factor using pathway analysis.


<div class="row">
    <div style="width: 100%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj3_genetopic.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
 Gene activity in factors/topics learned by the model.
</div>


In conclusion, the results from AttnCell model are promising. We can further develop this idea to design a computational model that provides biological interpretability at multiple layers.

The code for [AttnCell](https://github.com/sishirsubedi/attncell/) model is available.

