---
layout: page
title: Contaminated or Not? 
description:
img: assets/img/proj11_thumb.png
importance: 11
category: data-science
related_publications: false
---

<div class="row">
    <div style="width: 90%; height: 90%;  align: left; max-width:400px; max-height:500px">
 {% include figure.liquid loading="eager" path="assets/img/proj11_thumb.png"
 class="img-fluid rounded z-depth-1" %}
 </div>
</div>

<span style="color:#2698ba; font-size:30px; font-weight:bold;">
Investigate contamination in NGS testing
</span>

In this project, we study [Conpair: concordance and contamination estimator for matched tumor–normal pairs](https://pubmed.ncbi.nlm.nih.gov/27354699/)


[Github repo](https://github.com/nygenome/Conpair)


# Conpair model

The model consists of two different parts.

- Model 1 (Verify Concordance) to estimate if a Tumor sample is contaminated or not.
  
- Model 2 (Estimate contamination) to estimate the level of contamination in a Tumor sample.


## Model 1: Verify Concordance

In this model, we are interested to compare Tumor and Normal samples and 
analyze if **Tumor sample is contaminated.**

Idea: If we look at fix variants that are homozygous in Normal sample and 
are less likely to change in Tumor, then we should also see these variants as 
homozygous in Tumor sample. Any deviation will hint towards Tumor sample
being contaminated.


### Inputs
We have two inputs: marker and pileup from BAM files.

Preselected markers consisting chrom, pos, ref, alt, population_frequency.

```Python
class Marker:
    
    def __init__(self, chrom, pos, ref, alt, RAF):
        self.chrom = chrom
        self.pos = pos
        self.ref = ref
        self.alt = alt
        self.RAF = RAF

```

For each marker, we also need sequencing quality information from pileup.

```Python
class Pileup:
    
    def __init__(self, chrom, pos, ref, qual_A, qual_C, qual_G, qual_T):
        self.chrom = chrom
        self.pos = pos
        self.ref = ref
        self.Quals = {}
        self.Quals['A'] = qual_A
        self.Quals['C'] = qual_C
        self.Quals['G'] = qual_G
        self.Quals['T'] = qual_T
        self.depth = sum([len(qual_A), len(qual_C), len(qual_G), len(qual_T)])

```

### Model
Now, for each marker we look for each read and its quality to calculate
AA, BB, and AB likelihoods.

ref_baseq = [Q1, Q2, ...]  # qualities for REF reads
alt_baseq = [Q1, Q2, ...]  # qualities for ALT reads

RAf = This is prior from population allele frequency 


```Python
M = dict()
For each marker in markers file:

    ref = marker.ref
    alt = marker.alt
    RAF = marker.RAF
        
    AA_likelihood, AB_likelihood, BB_likelihood = compute_genotype_likelihood(pileup.Quals[ref], pileup.Quals[alt], normalize=False)
    prAA, prAB, prBB = prior_genotype_probability(RAF)
    
    
    M[pileup.chrom + ":" + pileup.pos] = {'likelihoods' : [AA_likelihood/prAA, AB_likelihood/prAB, BB_likelihood/prBB], 'coverage': pileup.depth}
```


**Important**
Why divide instead of multiply?

Instead of Bayes formula - Posterior ∝ Likelihood × Prior
Conpair does - Signal ≈ Likelihood / Prior 

Because Conpair wants: How surprising is this data relative to population expectation?
Rare genotypes get boosted if strongly supported
Common genotypes are penalized unless strongly supported
This improves contamination detection


If we look at how likelihoods are calculated, it is basically multiply probabilities from Phred scores for each read and then normalize across genotypes.


$$
p = 10^{-Q/10}
$$

where:
- \(p\) = probability that the base call is incorrect  
- \(Q\) = Phred quality score


```Python
def compute_genotype_likelihood(ref_baseq, alt_baseq, normalize=True):
    '''Randomly downsample to 450x if needed and then get likelihood from base quals'''
    ref_baseq = downsample(ref_baseq)
    alt_baseq = downsample(alt_baseq)
    AA = 1
    BB = 1
    for i in ref_baseq:
        p = phred_to_p(i)
        AA *= (1 - p)
        BB *= p
    for i in alt_baseq:
        p = phred_to_p(i)
        AA *= p
        BB *= (1 - p)
        
    AB = 0.5**(len(ref_baseq) + len(alt_baseq))

    if normalize is True:
        S = AA + BB + AB
        AA = AA/S
        AB = AB/S
        BB = BB/S
    return(AA, AB, BB)

def phred_to_p(phred):
    return(np.float64(10**(-float(phred)/10)))


def prior_genotype_probability(RAF):
    return(RAF**2, 2*RAF*(1-RAF), (1-RAF)**2)

```

### Model Equations (Normal vs Tumor)

For each marker in Normal and Tumor, let:
- \(R_s\) = set of ref-supporting reads  
- \(A_s\) = set of alt-supporting reads  

#### Phred to error probability
$$
p_i = 10^{-Q_i/10}
$$

#### Genotype likelihoods
$$
\mathrm{AA}_s = \prod_{i \in R_s} (1 - p_i)\ \prod_{j \in A_s} p_j
$$

$$
\mathrm{BB}_s = \prod_{i \in R_s} p_i\ \prod_{j \in A_s} (1 - p_j)
$$

$$
\mathrm{AB}_s = \left(\frac{1}{2}\right)^{|R_s| + |A_s|}
$$

#### Normalization
$$
S_s = \mathrm{AA}_s + \mathrm{AB}_s + \mathrm{BB}_s
$$

$$
P(\mathrm{AA}_s) = \frac{\mathrm{AA}_s}{S_s}, \quad
P(\mathrm{AB}_s) = \frac{\mathrm{AB}_s}{S_s}, \quad
P(\mathrm{BB}_s) = \frac{\mathrm{BB}_s}{S_s}
$$

#### Genotype priors (Hardy–Weinberg, using \(RAF\))
$$
P(\mathrm{AA}) = RAF^2, \quad
P(\mathrm{AB}) = 2\,RAF(1-RAF), \quad
P(\mathrm{BB}) = (1-RAF)^2
$$

Calculate likelihoods for both normal and tumor genotypes for all selected markers.

```Python
Normal_genotype_likelihoods = genotype_likelihoods_for_markers(Markers, opts.normal_pileup, min_map_quality=MMQ, min_base_quality=MBQ)
Tumor_genotype_likelihoods = genotype_likelihoods_for_markers(Markers, opts.tumor_pileup, min_map_quality=MMQ, min_base_quality=MBQ)
```

### Concordance scoring

And then come with concordance score by checking how many are matched.

```Python
for m in Markers:
    NL = Normal_genotype_likelihoods[m]
    TL = Tumor_genotype_likelihoods[m]
    if NL is None or TL is None:
        continue
    if NL['coverage'] < COVERAGE_THRESHOLD or TL['coverage'] < COVERAGE_THRESHOLD:
        continue
    if AA_BB_only:
        if NL['likelihoods'].index(max(NL['likelihoods'])) == 1:
            continue
    if NL['likelihoods'].index(max(NL['likelihoods'])) == TL['likelihoods'].index(max(TL['likelihoods'])):
        concordant += 1
    else:
        discordant += 1
```

### Summary 

```
Preselected markers
       ↓
pileup → get base qualities
       ↓
likelihoods (AA, AB, BB) using Phred scores
       ↓
divide by prior → signal using Bayes formula
       ↓
argmax → genotype call for tumor and normal for each marker
       ↓
compare tumor vs normal
       ↓
concordant / discordant counts
       ↓
discordance rate → contamination signal
```


## Model 2: Estimate contamination

From the above model, we fist estimate if Tumor sample is contaminated or not.

In this model, we are interested to **quantify Tumor sample contamination.**

Idea: We consider two genotypes mixture i.e. the true tumor genotype and the possible genotype of a contaminant. We then evaluate all possible contamination levels from 0.0 to 1.0. For each marker, we use both ref and alt base qualities and compute read likelihoods under all 9 possible genotype combinations (AA, AB, BB for both true and contaminant) at each contamination level. These likelihoods are combined with genotype priors and summed across reads and markers, and the contamination level that best explains the observed data (i.e., maximizes the likelihood) is selected as the final estimate.
### Inputs

For each marker, we have alt, ref, RAF and then corresponding pileup:

We do the same calculation as above for likelihood for both Normal and Tumor using
the same function ```compute_genotype_likelihood```.

```Python
    RAF = marker.RAF
    ref_basequals = pileup.Quals[marker.ref]
    alt_basequals = pileup.Quals[marker.alt]
    
    AA_likelihood, AB_likelihood, BB_likelihood = Genotypes.compute_genotype_likelihood(pileup.Quals[marker.ref], pileup.Quals[marker.alt], normalize=True)

    if AA_likelihood >= HOMOZYGOUS_P_VALUE_THRESHOLD:
        Normal_homozygous_genotype[pileup.chrom][pileup.pos] = {'genotype': marker.ref, 'AA_likelihood': AA_likelihood, 'AB_likelihood': AB_likelihood, 'BB_likelihood': BB_likelihood}
    elif BB_likelihood >= HOMOZYGOUS_P_VALUE_THRESHOLD:
        Normal_homozygous_genotype[pileup.chrom][pileup.pos] = {'genotype': marker.alt, 'AA_likelihood': AA_likelihood, 'AB_likelihood': AB_likelihood, 'BB_likelihood': BB_likelihood}        
```

Now, instead of calculating signal like before using likelihood and RAF prior i.e. ```AA_likelihood/prAA```, here we use different function to calculate priors.

```Python
    p_AA, p_AB, p_BB = Genotypes.RAF2genotypeProb(RAF)

```
We are using the same formulation, but treating RAF and ALT separately:
RAF = Reference Allele Frequency
ALT = 1 - RAF = Alternate allele frequency

```Python

def RAF2genotypeProb(RAF):
    ALT = 1 - RAF
    p_refref = RAF*RAF
    p_refalt = 2*RAF*ALT
    p_altalt = ALT*ALT
    return(p_refref, p_refalt, p_altalt

```

Now, we have input data complete:
- ref_basequals
- alt_basequals
- priors

```Python
    ### for numerical stability just using log for multiplication -> additions: log(a.b) = log a + log b

    lPAA = MathOperations.log10p(p_AA)
    lPAB = MathOperations.log10p(p_AB)
    lPBB = MathOperations.log10p(p_BB)

    
    priors = [lPAA*2, lPAA+lPBB, lPAA+lPAB, lPAB*2, lPAB+lPAA, lPAB+lPBB, lPBB*2, lPBB+lPAA, lPBB+lPAB]
    marker_data = [priors, ref_basequals, alt_basequals]
```

Why do we need to transform priors into 9 different values?

```
We assume:
G1 = Tumor genotype
G2 = Contaminant genotype
```

Since we need a **log-probability weight** for each genotype pair, we need to consider genotype of contaminant which can be all three - ```AA,AB,BB```.


So we need to consider all possible genotype combinations.

Conpair assumes:

$$P(G1,G2) = P(G1)⋅P(G2)$$

Each comes from:

$$P(AA), P(AB), P(BB)$$

```
priors = [
    lPAA*2,        # AA, AA
    lPAA+lPBB,     # AA, BB
    lPAA+lPAB,     # AA, AB
    lPAB*2,        # AB, AB
    lPAB+lPAA,     # AB, AA
    lPAB+lPBB,     # AB, BB
    lPBB*2,        # BB, BB
    lPBB+lPAA,     # BB, AA
    lPBB+lPAB      # BB, AB
]


           unknown
        /          \
    G₁ (true)    G₂ (contaminant)
     |              |
     3 states       3 states
        \          /
          9 combos
              ↓
     generate observed reads


```

Ok, finally we have inputs as:

1. Raw read evidence (from pileup): ref and alt quals from pileup
2.  Genotype likelihoods (ONLY for Normal → to model G₁): likelihood calculation from Phred scores like before
3. Mixture genotype 9 combinations as priors

### Algorithms

```
checkpoints = [i for i in drange(0.0, 1.0, grid_precision)]
Scores = ContaminationModel.create_conditional_likelihood_of_base_dict(checkpoints)
```

Here, checkpoints are all possible contamination levels between 0 to 1.

Now, understanding ```Scores``` is very important:

Scores is a precomputed lookup table of log-probabilities for observing a base given:
- genotype pair (G1,G2)
- contamination level α
- base quality Q
- observed allele (REF or ALT)


Lookup table is ```Scores[genotype_pair][alpha][base_quality]```


```Python
baseQ_max = 60
def create_conditional_likelihood_of_base_dict(checkpoints):
    
    D = defaultdict(lambda: defaultdict(dict))
    for bq in range(0, baseQ_max+1):
        Q = phred2prob(bq)/3

        f_AAAA_A = lambda x: (1-Q)
        f_AAAA_B = lambda x: Q
        f_BBBB_B = lambda x: (1-Q)
        f_ABAB_A = lambda x: 0.5
        f_ABAB_B = lambda x: 0.5
        
        f_AABB_A = lambda x: ((1-x) * (1 - Q)) + (x * Q)
        f_AABB_B = lambda x: (x * (1 - Q)) + ((1-x) * Q)
        f_AABA_A = lambda x: (1-0.5*x) * (1 - Q) + (0.5*x * Q)
        f_AABA_B = lambda x: (0.5*x) * (1 - Q) + (1-0.5*x) * Q
            
        f_ABAA_A = lambda x: (0.5 + 0.5*x) * (1 - Q) + (1-x)*0.5*Q
        f_ABAA_B = lambda x: (1-x)*0.5 * (1 - Q) + (0.5 + 0.5*x)  * Q
        for v in checkpoints:
            D['AABB_A'][v][bq] = log10(np.float64(f_AABB_A(v)))
            D['AABB_B'][v][bq] = log10(np.float64(f_AABB_B(v)))
            
            D['AABA_A'][v][bq] = log10(np.float64(f_AABA_A(v)))
            D['AABA_B'][v][bq] = log10(np.float64(f_AABA_B(v)))
            
            D['ABAA_A'][v][bq] = log10(np.float64(f_ABAA_A(v)))
            D['ABAA_B'][v][bq] = log10(np.float64(f_ABAA_B(v)))
            
            D['AAAA_A'][v][bq] = log10(np.float64(f_AAAA_A(v)))
            D['AAAA_B'][v][bq] = log10(np.float64(f_AAAA_B(v)))
            
            D['ABAB_A'][v][bq] = log10(np.float64(f_ABAB_A(v)))
            D['ABAB_B'][v][bq] = log10(np.float64(f_ABAB_B(v)))
            
    return(D)
```


Now, for final likelihood calculation:

- For each marker

- For each read (REF and ALT)

- For each contamination level α

--> lookup precomputed log-probabilities from ```Scores``` table 
    and add them to 9 genotype-pair likelihoods

#### Contamination Likelihood Model 

For each marker $m$ and contamination value $\alpha$, we compute:

#### 1. Data per marker

- REF base qualities: $R_m = \{q_1, q_2, \dots\}$
- ALT base qualities: $A_m = \{q_1, q_2, \dots\}$
- Genotype-pair log priors (9 states):

$$
\ell_j = \log_{10} P(G_1, G_2)_j, \quad j = 1,\dots,9
$$


#### 2. Per-read likelihood (from precomputed Scores)

For each read with base quality $q$:

$$
\log_{10} P(\text{REF read} \mid \alpha, G_1, G_2, q)
= S^{(A)}_j(\alpha, q)
$$

$$
\log_{10} P(\text{ALT read} \mid \alpha, G_1, G_2, q)
= S^{(B)}_j(\alpha, q)
$$


#### 3. Likelihood per genotype pair

For each genotype pair $j$:

$$
L_{m,j}(\alpha)
=
\ell_j
+
\sum_{q \in R_m} S^{(A)}_j(\alpha, q)
+
\sum_{q \in A_m} S^{(B)}_j(\alpha, q)
$$


#### 4. Marginalize over genotype pairs

$$
\log_{10} P(\text{data}_m \mid \alpha)
=
\log_{10} \left(
\sum_{j=1}^{9}
10^{L_{m,j}(\alpha)}
\right)
$$


#### 5. Total likelihood over all markers

$$
D(\alpha)
=
\sum_{m}
\log_{10} P(\text{data}_m \mid \alpha)
$$


#### 6. Final estimate

$$
\hat{\alpha}
=
\arg\max_{\alpha} D(\alpha)
$$



In code:


```Python
def likelihood_per_marker(A, Scores, checkpoints, ref_basequals, alt_basequals):
    
    for bq in ref_basequals:
        for i in range(0, len(checkpoints)):
            v = checkpoints[i]
            ref_part = [Scores['AAAA_A'][v][bq], Scores['AABB_A'][v][bq], Scores['AABA_A'][v][bq], Scores['ABAB_A'][v][bq], Scores['ABAA_A'][v][bq], Scores['ABAA_B'][v][bq], Scores['AAAA_B'][v][bq], Scores['AABB_B'][v][bq], Scores['AABA_B'][v][bq]]
            A[i] += ref_part

    for bq in alt_basequals:
        for i in range(0, len(checkpoints)):
            v = checkpoints[i]
            alt_part = [Scores['AAAA_B'][v][bq], Scores['AABB_B'][v][bq], Scores['AABA_B'][v][bq], Scores['ABAB_B'][v][bq], Scores['ABAA_B'][v][bq], Scores['ABAA_A'][v][bq], Scores['AAAA_A'][v][bq], Scores['AABB_A'][v][bq], Scores['AABA_A'][v][bq]]
            A[i] += alt_part
            
    return(A)


def calculate_contamination_likelihood(checkpoints, Data, Scores):
    
    D = np.zeros([len(checkpoints), 1],dtype='float64')
    for marker_data in Data:
        A = np.zeros([len(checkpoints), 9],dtype='float64')
        for i in range(0, len(checkpoints)):
            A[i] += marker_data[0]
        A = likelihood_per_marker(A, Scores, checkpoints, marker_data[1], marker_data[2])
        for i in range(0, len(checkpoints)):
            D[i] += log10p(sum([pow(10,x ) for x in A[i]]))
            
    return(D)


```
So, D calculates one value for log-likelihood for each contamination level.

We pick the highest likelihood value in D.

```Python
ARGMAX = np.argmax(D)
cont = checkpoints[ARGMAX]
```

Now the issue is Grid is coarse (step = 0.01) and True optimum may lie between grid points.

```Python
x1 = cont - grid_precision
x2 = cont
x3 = cont + grid_precision

optimal_val = ContaminationModel.apply_brents_algorithm(Data, Scores, x1, x2, x3)
```

Basically brents algorithm is trying to find accurate estimate, so instead of 0.3, it is trying to find 0.0367. Brents algorithm is ```scipy``` function for optimization.

### Summary

```
Preselected markers
       ↓
pileup → get base qualities (ref & alt reads)
       ↓
Normal sample → compute genotype likelihoods (AA, AB, BB)
       ↓
Select high-confidence homozygous sites (AA or BB)
       ↓
Build genotype-pair priors (G₁ = true, G₂ = contaminant → 9 combinations)
       ↓
Define contamination grid (α values: 0 → 0.5)
       ↓
Precompute Scores (mixture model + sequencing error per base quality)
       ↓
For each α:
    For each marker:
        Compute likelihood for all 9 genotype pairs
        Sum read-level log probabilities using Scores
        log-sum-exp over genotype pairs
    ↓
    Sum over all markers → D(α)
       ↓
Likelihood curve D(α)
       ↓
argmax → best contamination estimate (coarse grid)
       ↓
Brent optimization → refine α (continuous)
       ↓
Final contamination estimate
```
