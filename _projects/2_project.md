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
Graph neural networks (GNN) for latent space decomposition
</span>

In the [Biolord](https://www.nature.com/articles/s41587-023-02079-x) paper, the authors present an interesting approach to decomposing a mixed latent space to capture label/condition-specific effects. The described deep learning model is based on a generative framework consisting of a dedicated subnetwork for each known attribute. The multiple module networks are jointly optimized.

Overall idea of Biolord model is - 
- step 1: generate mixed latent space
- step 2: use label specific subnetwork to isolate label specific effects from mixed latent space
- step 3: joint training with data reconstruction. 

**Key idea**: Can we replace label-specific subnetworks with a single graph network built on a mixed space?

The updated Biolord model, named as **GRASP for Graph Representation Analysis for Single-cell Perturbations**, consists of the following steps.

**Low-dimensional space**: We first aim to use any dimension reduction technique to represent high-dimensional data in a low-dimensional space. This space is used to generate an attribute-specific graph. 

**Attribute specific graphs**: Next, in low-dimensional space, we find similar cells from different groups. This approach will construct a cell-cell similarity graph in adjacent matrix format such that each cell has the edge with similar cells that belong to different labels. For example, if we have batch and cell-type labels, then we will generate batch and cell type-based graphs. In a batch-based graph, edges are constructed between similar cells from different batches (most likely from the same cell type). Similarly, in a cell type-based graph, edges are constructed between similar cells from different cell types (most likely from the same batch).

**Why graphs?** First, we only build a single graph in mixed space instead of attribute-specific multiple modules in Biolord. We use the graph repeatedly to construct attribute-specific representations. This will provide scalability to the model. Second, when we use attribute-specific graphs, we can learn shared effects specific to each attribute. We can guide the shared effect to generate attribute-specific latent space.


**Simplified training**: In Biolord, we have attribute-specific losses, but in GRASP, we have only two losses- 
- reconstruction loss and 
- alignment loss to encourage independence among attribute-specific factors. 


<div class="row">
 {% include figure.liquid loading="eager" path="assets/img/proj1_overview.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
<div class="caption">
 Overview of GRASP framework.
</div>


### GRASP model:
- Here, we use batch and cell type labels as two attribute labels.
- Pre-train steps:
    - Use any latent space representation model (such as PCA) to obtain z_pca
    - Generate a graph based on z_pca space and project it to batch space and group space
- GRASP training :
    - input: raw data and two graphs in batch space and group space
    - model :
        - Encode raw data to z_mix
        - Capture batch effect z_batch using GNN(z_mix,batch space graph) 
        - Capture group effect z_group using GNN(z_mix,group space graph) 
        - Isolate z_unknown from FCN (z_mix, [z_batch + z_group])
        - Reconstruct data using z_batch + z_group + z_unknown
        - Discriminator learning for batch and group effect


Preliminary results for simulation data:


<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj1_sim1.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
 Mixed space representation.
</div>
<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj1_sim2.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
 Attribute specific - batch representation.
</div>
<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj1_sim3.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
 Attribute specific - cell type representation.
</div>

Preliminary results for normal pancreas real data:


<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj1_pancreas1.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
 Mixed space representation.
</div>
<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj1_pancreas2.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
 Attribute specific - batch representation.
</div>
<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj1_pancreas3.png" title="example image" class="img-fluid rounded z-depth-1" %}
</div>
</div>
<div class="caption">
 Attribute specific - cell type representation.
</div>

The results, especially from simulation data, are promising. In mixed space, we observe that the batch effect is dominant, followed by the cell type effect, which generates unique clusters. Attribute-specific representations exhibit distinct clusters that capture attribute effects. However, the results from the normal pancreas data are not convincing and suggest that more work is needed to refine the model. The project code used to generate the above results is available [GRASP](https://github.com/sishirsubedi/grasp). 