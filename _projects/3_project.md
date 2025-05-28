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

The biological interpretability of deep learning models has been an intriguing problem. 

In this project, the main aim is to test a deep learning model with robust biological interpretability.

A model, named as attncell ( model explains where to pay "attention" from "cell" data ) is focused on learning the following biological information:
- gene interaction networks
- activity of pathways in a cell
- contribution of genes in pathways


<div class="row">
 {% include figure.liquid loading="eager" path="assets/img/proj3_overview.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
<div class="caption">
 Overview of attnCell framework.
</div>

We train the model using variational inference algorithm and maximize the evidence lower bound (ELBO). We have two loses in the model - Kullback-Leibler (KL) loss and Multinomial-Dirichlet likelihood loss, both in the decoder network. 

Preliminary results using normal pancreas data from [Seurat](https://satijalab.org/seurat/archive/v3.2/integration.html).


- Interpretability1: Gene interaction networks is learned by attention module in the model. The figure shows example of known marker genes - NPTX2/DLK1 for beta cells, LEPR/RBP4 for delta, and PLA2G1B/CPA1 for acinar cells in human pancreas.

<div class="row">
    <div style="width: 100%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj3_genenetwork.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
 Gene network learned by the model.
</div>

- Interpretability2: Activity of pathways in cells, which in general gives cell type identity, can be captured by analyzing UMAP representation of cells based on latent space learned by the model. Here, cell type specific clusters suggest that the factors learned by the model are biologically relevant. 


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


In conclusion, the results from attncell model is promising. We can further build on this idea to design a computational model which provides biological interpretability at different layers.

The code for [attncell](https://github.com/sishirsubedi/attncell/) model is available.

