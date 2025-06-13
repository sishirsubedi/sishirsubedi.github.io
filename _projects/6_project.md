---
layout: page
title: flowRIDA
description:
img: assets/img/proj6_thumb.png
importance: 1
category: project-ideas
related_publications: false
---

<span style="color:#2698ba; font-size:30px; font-weight:bold;">
Flow-based RepresentatIon Disentanglement Analysis (FlowRIDA) </span>

In this project, we study [PerturbNet](https://www.biorxiv.org/content/10.1101/2022.07.20.500854v2) paper and test the applicability of [Conditional Invertible Neural Networks (cINN)](https://proceedings.neurips.cc/paper/2020/hash/1cfa81af29c6f2d8cacb44921722e753-Abstract.html) in deep learning. For proof of concept, we design a new cINN-based model, Flow-based Representational Disentanglement Analysis (FlowRIDA), to learn dual representations of single cells capturing normal and perturbation effects.

The authors in the paper present a multilayer deep-learning approach, first to learn latent representations for perturbation and cellular states and second to learn a mapping function to predict the effects of perturbation in normal cells. The method focuses on perturbations induced by genetics as well as chemical drugs. The authors demonstrate the applicability of the model in predicting responses to unseen small molecule treatments, focusing on drug discovery. 


- PerturbNet Paper: [Yu, Hengshi, and Joshua D. Welch. "Perturbnet predicts single-cell responses to unseen chemical and genetic perturbations." BioRxiv (2022): 2022-07.](https://www.biorxiv.org/content/10.1101/2022.07.20.500854v2)
- PerturbNet [Github repo](https://github.com/welch-lab/PerturbNet)

- cINN Paper :[Rombach, Robin, Patrick Esser, and Bjorn Ommer. "Network-to-network translation with conditional invertible neural networks." Advances in Neural Information Processing Systems 33 (2020): 2784-2797.](https://proceedings.neurips.cc/paper/2020/hash/1cfa81af29c6f2d8cacb44921722e753-Abstract.html) 
- cINN [Github repo](https://github.com/CompVis/net2net)

- The dataset is available from [GSE133344](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE133344). The original paper is [Norman, Thomas M., Max A. Horlbeck, Joseph M. Replogle, Alex Y. Ge, Albert Xu, Marco Jost, Luke A. Gilbert, and Jonathan S. Weissman. "Exploring genetic interaction manifolds constructed from rich single-cell phenotypes." Science 365, no. 6455 (2019): 786-793.](https://www.science.org/doi/full/10.1126/science.aax4438)


### Data prep

Download `Norman_raw.h5ad` file from [figshare](https://ndownloader.figshare.com/files/34002548) repo from Norman et al. paper.

We generate a control/pert data pair using 'guide_identity' columns in  data as the following:

We follow the data preparation step as described in [theislab tutorial](https://github.com/theislab/sc-pert/blob/main/datasets/Norman_2019_curation.ipynb).

- Normal cells: Cells with the tag as 'NegCtrl.' 

- Perturbed cells: To reduce data size, we select PRO_GROWTH, G1_CYCLE, and MEGAKARYOCYTE cells based on the results from the original paper.
    - PRO_GROWTH = [
 "CEBPE+KLF1",
 "KLF1+MAP2K6",
 "AHR+KLF1",
 "ctrl+KLF1",
 "KLF1+ctrl",
 "KLF1+BAK1",
 "KLF1+TGFBR2",
 ]

    - G1_CYCLE = [
 "CDKN1C+CDKN1B",
 "CDKN1B+ctrl",
 "CDKN1B+CDKN1A",
 "CDKN1C+ctrl",
 "ctrl+CDKN1A",
 "CDKN1C+CDKN1A",
 "CDKN1A+ctrl",
 ]

    - MEGAKARYOCYTE = [
 "ctrl+ETS2",
 "MAPK1+ctrl",
 "ctrl+MAPK1",
 "ETS2+MAPK1",
 "CEBPB+MAPK1",
 "MAPK1+TGFBR2",
 ]

<div class="row">
    <div class="caption">
 PRO_GROWTH, G1_CYCLE, and MEGAKARYOCYTE cells have distinct clusters in the original paper.
    </div>
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj6_paperumap.png"
 class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Cell representation 

First, we use a simple VAE neural network to learn cellular latent representation for control and perturbed cells.

<div class="row">
    <div class="caption">
 UMAP from VAE model for control cells.
    </div>
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj6_vaecntrl.png"
 class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="row">
    <div class="caption">
 UMAP from VAE model for perturbed cells.
    </div>
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj6_vaepert.png"
 class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### flowRIDA training

Once we have obtained a latent representation from a simple VAE model for control and perturbed cells, we aim to learn the transformation from one latent space to another (control to perturbed) using the flowRIDA model.

```
import flowrida
import anndata as an
import numpy as np
import flowrida

sample = 'norman'
wdir = 'znode/'+sample+'/'
tag='pert'
adata_p = an.read_h5ad(wdir+'results/'+sample+'_'+tag+'_flowrida.h5ad')
tag='control'
adata_c = an.read_h5ad(wdir+'results/'+sample+'_'+tag+'_flowrida.h5ad')

####perturbation - select 2500 cells for demo
adata_p = adata_p[adata_p.obs.sample(n=2500).index.values,:]
####control
adata_c = adata_c[adata_c.obs.sample(n=adata_p.shape[0]).index.values,:]


####
tag='flow'
flowrida_object = flowrida.fr.create_flowrida_object(adata_c,wdir,tag)
params = {
 'device': 'cuda', 
 'input_dim': adata_c.obsm['latent'].shape[1],
 'condition_dim': adata_p.obsm['latent'].shape[1],
 'hidden_dim': 128,
 'num_layers': 3,
 'learning_rate': 0.001, 
 'epochs':1000,
 'batch_size':128
 }
flowrida_object.set_flow_nn_params(params)
flowrida_object.train_flow(adata_c,adata_p)
flowrida_object.flow_nn_params['device']='cpu'
flowrida_object.eval_flow(adata_c,adata_p)
```
### Analysis

We evaluate flowRIDA based on the following transformations:

- Control vs. perturbation to control: In this case, we are comparing the normal cell state with the learned transformation to the normal cell state from a perturbed state. This is the easiest among all, as the normal state is shared between both control and perturbation cells. In the UMAP figure, we see a significant overlap between control and predicted control cells from perturbation cells.
  

- Perturbation vs. control to perturbation: In this case, we are comparing the perturbed cell state with the learned transformation to the perturbed cell state from the normal cell state. This is harder than transformation to a normal state; still, perturbation cells are not completely distinct from the normal state; they have a certain portion of shared effects with control cells. In the UMAP figure, we also see a significant overlap between perturbed and predicted perturbed cells from control cells.
 
- Control vs. perturbation: In this case, we are comparing the normal cell state with the learned transformation to the perturbed cell state from the normal state. This is the most important and most challenging transformation among all three. The transformation function can be used to predict the effect of perturbation in normal cells. In the UMAP figure, we see minimal overlap between normal cells and predicted perturbed cells from normal cells. The red lines show cell pair mapping between normal to perturbation transformation. Here, we can map cells to identify which perturbed cell state a normal cell would acquire after perturbation.


<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj6_cp2c.png"
 class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj6_pc2p.png"
 class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="row">
    <div style="width: 80%; margin: 0 auto;">
 {% include figure.liquid loading="eager" path="assets/img/proj6_cc2p.png"
 class="img-fluid rounded z-depth-1" %}
    </div>
</div>

In conclusion, flowRIDA model provides a modelling framework to map cells between two distinct representations such as normal and disease cell states. The model employs a Conditional Invertible Neural Networks-based approach to learn a transformation function that maps cells. The preliminary results are promising and demonstrate proof of concept for applying the cINN technique in mapping latent spaces within single-cell data. We can further refine this model to learn dynamic changes in cellular processes that lead to abnormalities.

The project code used to generate the above results is available [flowRIDA](https://github.com/sishirsubedi/flowRIDA).