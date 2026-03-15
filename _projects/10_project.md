---
layout: page
title: Graph neural network for integration
description:
img: assets/img/proj10_thumb.png
importance: 7
category: paper-reading
related_publications: false
---

<div class="row">
    <div style="width: 50%; height: 50%;  align: left; max-width:200px; max-height:250px">
 {% include figure.liquid loading="eager" path="assets/img/proj10_thumb.png"
 class="img-fluid rounded z-depth-1" %}
 </div>
</div>

<span style="color:#2698ba; font-size:30px; font-weight:bold;">
Study SpaMosaic model to integrate latent space across modalities
</span>

In this project, we study the [SpaMosaic](https://www.biorxiv.org/content/10.1101/2024.10.02.616189v1) paper and understand how biological data (RNA expression, chromatin accessibility (ATAC), and protein abundance) can be integrated with spatial information using graph neural networks (GNNs) and contrastive learning. SpaMosaic is interesting because it enables horizontal integration across datasets that do not necessarily contain the same modalities. As long as there are sufficient anchor samples that share modalities across datasets, the model can learn a shared latent embedding that aligns biological signals across modalities and experiments. Once this shared representation is learned, the framework can also impute missing modalities, allowing datasets with incomplete measurements to benefit from information present in other datasets.


- SpaMosaic Paper: [Yan, Xuhua, et al. "Mosaic integration of spatial multi-omics with SpaMosaic." BioRxiv (2024): 2024-10.](https://www.biorxiv.org/content/10.1101/2024.10.02.616189v1)

- SpaMosaic [Github repo](https://github.com/JinmiaoChenLab/SpaMosaic)




- Let's start with three datasets presented in the paper:

```

data_dir = './demo/data/Human_lymph_node'

ad1_rna = sc.read_h5ad(join(data_dir, 'slice1/s1_adata_rna.h5ad'))
ad1_adt = sc.read_h5ad(join(data_dir, 'slice1/s1_adata_adt.h5ad'))
ad2_rna = sc.read_h5ad(join(data_dir, 'slice2/s2_adata_rna.h5ad'))
ad3_adt = sc.read_h5ad(join(data_dir, 'slice3/s3_adata_adt.h5ad'))
ad3_rna = sc.read_h5ad(join(data_dir, 'slice3/s3_adata_rna.h5ad'))


ad1_rna = 3484 spots and 18k gene expr
ad1_adt = 3483 spots and 31 surface proteins
ad2_rna = 3359 spots and 18k gene expr
ad3_rna = 3408 spots and 18k gene expr
```

- Now, for each modality, we do preprocessing - for example, RNA gene expression will get batch correction and high variable gene selection. The main output we are interested in from this step is the dimension reduction output.


```
input_dict = {
 'rna': [ad1_rna, ad2_rna, ad3_rna],
 'adt': [ad1_adt, None,    ad3_adt]
}

input_key = 'dimred_bc'


RNA_preprocess(input_dict['rna'], batch_corr=True, favor='scanpy', n_hvg=5000, batch_key='src', key=input_key)
ADT_preprocess(input_dict['adt'], batch_corr=True, batch_key='src', key=input_key)
```



Model key ideas-

- We have a separate model for each modality, for example, RNA has its own encoder and embedding layer

- Modalities are shared across datasets

- Now, each spot in each modality has a latent embedding 

- The main trick is that this latent embedding is shared, so all modality spots are represented in the same space

- Why graph learning is needed? Since we have spatial information for each modality, we are interested in including this information in the model. If we use a fully connected network instead of a graph, then information capture is on a per-spot basis, i.e., information is not shared or learned across spots. To share spatial information, we utilize GNN.  This also helps in capturing local biology. For example, if a spot is a tumour, then we have neighbour spots to share and enhance tumour-specific features in the latent space. Note that the spot represented as a node in GNN is modality-specific, so the information shared is still within each modality.
The link that aligns spots across modality is using the training loss.

```
RNA graph → GNN_rna → z_rna                   
ATAC graph → GNN_atac → z_atac
		z_rna  - z_atac -> loss alignment
```
			
- But how does the model learn to represent different modality embedding in the same space: Specialized loss-based training

- Entry of graph: For each modality, we have separate graphs for embedding learning and spots are considered as nodes and edges as spatial proximity. Since modality features are the same across datasets, and these are normalized and batch corrected, we can mix spots across datasets for training.

- Important: Edges are constructed only within the dataset because even if spots are mixed, edges are obtained from location, and they do not mix. So, we will have a large graph with locally separate graphs, i.e. disconnected graphs.

- so embedding architecture:

Modality normalized features -> PCA -> ENCODER_GNN(PCA,SPATIAL) -> Embedding

```
model = SpaMosaic(
 modBatch_dict=input_dict, input_key=input_key,
 batch_key='src', 
 intra_knns=10, inter_knn_base=10, 
 w_g=0.8,
 seed=1234, 
 device='cpu'
)

model.train(net='wlgcn', lr=0.01, T=0.01, n_epochs=100)

```

- Once we have a modality-specific shared latent space where each spot, based on modality features, is passed through a GNN and embeddings are learned and have the same number of dimensions.


- WLGCN model is a graph neural network with an encoder–decoder architecture designed for node representation learning. First, the WLGCN_vanilla layer performs K steps of graph propagation, aggregating multi-hop neighborhood information so each node representation includes features from up to K-hop neighbours; the outputs from all propagation steps are concatenated (size input_size × (K+1)). This combined representation is then passed through a two-layer MLP encoder (fc1 → BatchNorm → Dropout → fc2) with LeakyReLU activation to produce a low-dimensional latent embedding (output_size). The model also includes a decoder (either a single linear layer or a small MLP) that attempts to reconstruct the original node features, making the model behave like a graph autoencoder that learns embeddings preserving graph structure and feature information. Finally, the embeddings are L2-normalized before being returned along with the reconstructed features, which can be used for representation learning tasks such as clustering, similarity search, or self-supervised training.

```
class WLGCN(torch.nn.Module):
 """
 Deep WLGCN with encoder-decoder structure for representation learning.

 Parameters
 ----------
 input_size : int
 Input feature dimension.
 output_size : int
 Output embedding dimension.
 K : int, optional
 Number of GCN propagation steps (default: 8).
 dec_l : int, optional
 Number of layers in the decoder (1 or 2). Default is 1.
 hidden_size : int, optional
 Hidden layer size for encoder. Default is 512.
 dropout : float, optional
 Dropout rate. Default is 0.2.
 slope : float, optional
 LeakyReLU negative slope. Default is 0.2.
 """
    
 def __init__(self, input_size, output_size, K=8, dec_l=1, hidden_size=512, dropout=0.2, slope=0.2):
 super(WLGCN, self).__init__()
 self.conv1 = WLGCN_vanilla(K=K)
 self.fc1 = torch.nn.Linear(input_size * (K + 1), hidden_size)
 self.bn = torch.nn.BatchNorm1d(hidden_size)
 self.dropout1 = torch.nn.Dropout(p=dropout)
 self.fc2 = torch.nn.Linear(hidden_size, output_size)
 self.negative_slope = slope

 if dec_l == 1:
 self.decoder = torch.nn.Linear(output_size, input_size)
 else:
 self.decoder = torch.nn.Sequential(
 torch.nn.Linear(output_size, output_size),
 torch.nn.ReLU(),
 torch.nn.Linear(output_size, input_size)
 )
        
 def forward(self, feature, edge_index, edge_weight=None):
 """
 Forward pass of the WLGCN model.

 Parameters
 ----------
 feature : torch.Tensor
 Input node features of shape (N, F).
 edge_index : torch.Tensor
 Edge indices in COO format.
 edge_weight : torch.Tensor or None
 Optional edge weights.

 Returns
 -------
 Tuple[torch.Tensor, torch.Tensor]
 - Normalized latent embeddings.
 - Reconstructed input features.
 """
        
 x = self.conv1(feature, edge_index, edge_weight)
 x = F.leaky_relu(self.fc1(x), negative_slope=self.negative_slope)
 x = self.bn(x)
 x = self.dropout1(x)
 x = self.fc2(x)
        
 r = self.decoder(x)
 x = F.normalize(x, p=2, dim=1)
 return x, r
```

- Now we have embeddings, how do we connect or conduct multi-modality integration?

- The key idea is finding anchor spot embeddings. These are spots where we have multiple measurements (RNA, surface protein, ATAC, etc.). So, these spots should be close to each other in latent space.  Similarly, spots with disjoint measurements should not be close to each other. For example, if we have a dataset for two modalities, A and B, then the latent space representation of spots in dataset A and spots in B should be closer in latent space.

- Contrastive learning:
  - Same spots: Spot from modality A - spot from modality B -> PUSH CLOSER
  - Different spots: Spot from modality A - spot from modality B -> PUSH FAR 

**Basically think spot as biological identity like cell type, i.e. immune or neuron cluster. Model will encourage the same biology sharing spots to cluster together while pushing different biology sharing spots further apart.**

- Another loss: reconstruction loss
  - This loss forces the model to preserve spatial structure.

