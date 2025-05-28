---
layout: page
title: GRASP
description:
img: assets/img/proj1_thumb.png
importance: 1
category: project-ideas
related_publications: false
---

<span style="color:#2a9d8f; font-size:30px; font-weight:bold;">
Use graph neural networks (GNN) for latent space decomposition
</span>

In [Biolord](https://www.nature.com/articles/s41587-023-02079-x) paper, authors present an interesting approach of decomposing   mixed latent space to capture label/condition specific effects. The described deep learning model is based on generative framework consisting of dedicated subnetwork for each known attribute. The multiple module networks are jointly optimized.

Overall idea of biolord model is - 
- step 1: generate mixed latent space
- step 2: use label specific subnetwork to isolate label specific effects from mixed latent space
- step 3: joint training with data reconstruction. 

**Key idea**: Can we replace label specific subnetwork with single graph network built on mixed space?

The updated biolord model, I call it as **GRASP for Graph Representation Analysis for Single-cell Perturbations**, consists of the following steps.

**Low-dimensional space**: We first want to use any dimension reduction technique to represent high-dimensional data in low-dimensional space. This space is used to generate a attribute specific graph. 

**Attribute specific graphs**: Next in low-dimensional space, we find closest similar cells with different attribute label. This approach will construct a cell-cell similarity graph in adjacent matrix format such that each cell has edge with similar cells that belongs to different labels. For example, if we have batch and cell type labels then we will generate batch graph and cell type graph. In batch graph, edges are constructed between similar cells from different batches (most likely from same cell type). Similarly, in cell type graph, edges are constructed between similar cells from different cell types (most likely from same batch).

**Why graphs?** First, we only build single graph in mixed space instead of attribute specific multiple modules in biolord. We use the graph repeatedly to construct its attribute specific representation. This will provide scalability to the model. Second, when we use attribute specific graph, we can learn shared effect specific to that attribute. We can guide the shared effect to generate attribute specific latent space.


**Simplified training**: In biolord, we have attribute specific loss, but in GRASP we have only two losses- 
- reconstruction loss and 
- alignment loss to encourage independence among attribute specific factors. 


<div class="row">
        {% include figure.liquid loading="eager" path="assets/img/proj1_overview.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
<div class="caption">
    Overview of GRASP framework.
</div>


### GRASP model:
- Here, we use batch and cell type labels as two attribute labels.
- Pre train steps:
    -   Use any latent space representation model (such as pca) to obtain z_pca
    -   Generate graph based on z_pca space and project it to batch space and group space
- GRASP training :
    - input : raw data and two graphs in batch space and group space
    - model :
        - Encode raw data to z_mix
        - Capture batch effect z_batch using GNN(z_mix,batch space graph) 
        - Capture group effect z_group using GNN(z_mix,group space graph) 
        - Isolate z_unknown from FCN (z_mix, [z_batch + z_group])
        - Reconstruct data using z_batch + z_group + z_unknown
        - Discriminator learning for batch and group effect


Preliminary results for simulation data:


<div class="row">
    <div style="width: 60%; margin: 0 auto;">
        {% include figure.liquid loading="eager" path="assets/img/proj1_sim1.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Mixed space representation.
</div>
<div class="row">
    <div style="width: 60%; margin: 0 auto;">
        {% include figure.liquid loading="eager" path="assets/img/proj1_sim2.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
    Attribute specific - batch representation.
</div>
<div class="row">
    <div style="width: 60%; margin: 0 auto;">
        {% include figure.liquid loading="eager" path="assets/img/proj1_sim3.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
    Attribute specific - cell type representation.
</div>

Preliminary results for normal pancreas real data:


<div class="row">
    <div style="width: 60%; margin: 0 auto;">
        {% include figure.liquid loading="eager" path="assets/img/proj1_pancreas1.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Mixed space representation.
</div>
<div class="row">
    <div style="width: 60%; margin: 0 auto;">
        {% include figure.liquid loading="eager" path="assets/img/proj1_pancreas2.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
    Attribute specific - batch representation.
</div>
<div class="row">
    <div style="width: 60%; margin: 0 auto;">
        {% include figure.liquid loading="eager" path="assets/img/proj1_pancreas3.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
    Attribute specific - cell type representation.
</div>

Pilot project code used to generate above results is available [GRASP](https://github.com/sishirsubedi/grasp). 