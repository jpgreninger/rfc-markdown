# PolarQuant: Quantizing KV Caches with Polar Transformation 

Insu Han<br>KAIST<br>insu.han@kaist.ac.kr

Praneeth Kacham<br>Google Research<br>pkacham@google.com

Amin Karbasi<br>Yale University<br>amin.karbasi@yale.edu

Vahab Mirrokni<br>Google Research<br>mirrokni@google.com

Amir Zandieh<br>Google Research<br>zandieh@google.com


#### Abstract

Large language models (LLMs) require significant memory to store Key-Value (KV) embeddings in their KV cache, especially when handling long-range contexts. Quantization of these KV embeddings is a common technique to reduce memory consumption. This work introduces PolarQuant, a novel quantization method employing random preconditioning and polar transformation. Our method transforms the KV embeddings into polar coordinates using an efficient recursive algorithm and then quantizes resulting angles. Our key insight is that, after random preconditioning, the angles in the polar representation exhibit a tightly bounded and highly concentrated distribution with an analytically computable form. This nice distribution eliminates the need for explicit normalization, a step required by traditional quantization methods which introduces significant memory overhead because quantization parameters (e.g., zero point and scale) must be stored in full precision per each data block. PolarQuant bypasses this normalization step, enabling substantial memory savings. The long-context evaluation demonstrates that PolarQuant compresses the KV cache by over $\times 4.2$ while achieving the best quality scores compared to the state-of-the-art methods.


## 1 Introduction

Transformer-based models form the backbone of modern artificial intelligence systems and have been instrumental in driving the ongoing AI revolution. Their applications span various domains, including frontier language models (LLM) [1, 3, 15] to text-to-image [32, 12, 28], text-to-video synthesis [16, 30], coding assistants [27] and even multimodal models that ingest text, audio, image, and video data [29, 15]. The self-attention mechanism [37] is at the heart of these models as it enables capturing the direct dependencies of all tokens in the input sequence. The ability of these models grows along with their size and context length [21], which leads to computational challenges in terms of huge memory consumption to support fast inference.

[^0]Most large language models, as well as multimodal and video models, adopt an autoregressive, decoder-only architecture that generates tokens sequentially. To avoid redundant attention score computations during the generation phase, these models employ a KV caching scheme, which stores the key and value embeddings of previously generated tokens in each attention layer. However, a significant challenge in deploying autoregressive Transformers lies in the substantial memory demands, as the KV cache size scales with both the model size (i.e., the number of layers and attention heads) and the context length. Furthermore, serving each model session typically necessitates its own dedicated KV cache, further compounding memory demands. This has become a significant bottleneck in terms of memory usage and computational speed, particularly for models with long context lengths. Thus, reducing the KV cache size while preserving accuracy is critical to addressing these limitations.

Several approaches have been proposed to address the KV caching challenge. Architectural solutions, such as multi-query attention [34], grouped-query attention [2], and multi-head latent attention [9], modify the transformer architecture to reduce the memory demands during inference by decreasing the number of key-value pairs that are to be stored.

Another orthogonal line of research focuses on reducing the KV cache size by pruning or evicting redundant or unimportant tokens [ $6,44,25,38,42,24$ ]. However, eviction strategies face limitations in long-context tasks that require precise knowledge extraction, such as needle-in-haystack scenarios. Additionally, some recent works tackle the issue from a systems perspective, such as offloading [35, 36] or integrating virtual memory and paging strategies into the attention mechanism [23].

A simple yet effective approach to reducing KV cache size is quantizing the floating-point numbers (FPN) in the KV cache by storing their approximations using fewer number of bits. Several quantization methods have been proposed specifically for the KV cache $[40,39,11,20,43,26,17]$. Recently, a new KV cache quantization method called QJL [41] introduced an efficient, data-oblivious 1-bit quantization approach based on sketching techniques. This method does not require tuning or adaptation to the input data, incurs significantly lower memory overhead compared to prior works, and achieves superior performance. A very recent work, Lexico [22], applies techniques from sparse representation learning to compress the KV cache by learning a universal dictionary such that all key and value embeddings are represented as extremely sparse vectors within the learned dictionary. Unfortunately, this approach requires solving a computationally expensive matching pursuit algorithm for each key and value embedding, making Lexico relatively slow.

Traditional KV cache quantization methods face significant "memory overhead" due to the need for data normalization before quantization. Most methods group data into blocks-either channelwise or token-wise-and independently normalize each block which requires computing and storing quantization constants (e.g., zero points and scales) in full precision. This process can add over 1 additional bit per quantized number, resulting in considerable memory overhead. We show that applying a random preconditioning matrix on the embedding vectors eliminates the need for data normalization. This approach aligns with the recent use of random Hadamard matrices as preconditioners before quantizing embedding vectors in attention layers to improve quality [33, 4].

### 1.1 Contributions

We propose quantizing KV vectors in polar coordinates instead of the usual Cartesian coordinates. This shift enables more efficient representation and compression of KV embeddings.

Random Preconditioning. We apply a random rotation to the vectors before quantization, which preserves inner products while randomizing the distribution of each vector. This preconditioning causes the angles in polar coordinates to concentrate, allowing us to quantize them with high precision using small bit-widths. We derive the analytical distribution of angles after preconditioning and leverage this insight to construct an optimized quantization codebook, minimizing quantization error.

Recursive Polar Transformation. We introduce a computationally efficient recursive polar transformation that converts vectors into polar coordinates, enabling practical deployment of our approach. We are able to prove an error bound in Theorem 1 showing our algorithm is asymptotically optimal for worst-case KV embedding vectors.

Performance on Long-Context Tasks. We evaluate PolarQuant on long-context tasks and demonstrate that it achieves the best quality scores compared to competing methods while compressing the KV cache memory by over × 4.2.

## 2 Preliminaries

We use boldface lowercase letters, such as $\boldsymbol{x}$ and $\boldsymbol{y}$, to denote vectors, and boldface uppercase letters, like $\boldsymbol{M}$, to denote matrices. To denote a slice of a vector $\boldsymbol{x}$ between the coordinate indices $i$ and $j$ inclusive of the endpoints, we use the notation $\boldsymbol{x}_{i: j}$. For a matrix $\boldsymbol{M}$, we write $\boldsymbol{M}_{i, \text { : }}$ to denote its $i$-th row vector, which we will simply refer to as $\boldsymbol{M}_{i}$.

### 2.1 Efficient Token Generation and KV Caching

Autoregressive Transformers often utilize cache storage for faster token generation. Given an input prompt, models encode the prompt information into two types of embeddings, called Key and Value. To generate subsequence tokens efficiently, the Key-Value (KV) embeddings are cached to avoid recomputing them.

The Key-Value (KV) caching method leverages the architecture of transformer decodcers, where a causal mask in applied in the attention mechanism. Once the keys and values are computed for a given token, they remain unchanged for subsequent token generation. By caching these key-value pairs, the model avoids redundant computations, as it only needs to compute the query for the current token and reuse the cached keys and values for attention.

This approach significantly reduces computation time during token generation. Instead of processing the entire sequence repeatedly, the KV cache enables the model to efficiently focus on the incremental computation of new tokens. This makes the method particularly useful in real-time applications, such as conversational AI and text generation, where fast and resource-efficient inference is critical.

![](https://cdn.mathpix.com/cropped/3d1c12a2-ff39-4daa-b19b-e0a8fb1f6256-04.jpg?height=542&width=1474&top_left_y=223&top_left_x=327)
Figure 1: Overview of recursive polar transformation procedure in Definition 1

### 2.2 Random Preconditioning

A critical step in the PolarQuant algorithm is random preconditioning of the KV vectors prior to quantization. This involves applying a random projection matrix to the embedding vectors before quantizing them. To analyze the algorithm effectively, we rely on specific facts and properties of multivariate normal random variables, which are outlined below.

Fact 1. For any positive integer $d$, if $\boldsymbol{x} \in \mathbb{R}^{d}$ is a zero mean unit variance isotropic Gaussian random variable in dimension $d$, i.e., $\boldsymbol{x} \sim \mathcal{N}\left(0, \boldsymbol{I}_{d}\right)$, then its 2 -norm, denoted by $r:=\|\boldsymbol{x}\|_{2}$, follows a generalized gamma distribution with the following probability density for any $r \geq 0$ :

$$
f_{R}(r)=\frac{2}{2^{d / 2} \cdot \Gamma(d / 2)} r^{d-1} \exp \left(-r^{2} / 2\right)
$$

The proof of Fact 1 is provided in Appendix A. We also use the following facts about the moments of the univariate normal distribution.

Fact 2 (Moments of Normal Random Variable). If $x$ is a normal random variable with zero mean and unit variance $x \sim \mathcal{N}(0,1)$, then for any integer $\ell, \mathbb{E}_{x \sim \mathcal{N}(0,1)}\left[|x|^{\ell}\right]=2^{\ell / 2} \Gamma((\ell+1) / 2) / \sqrt{\pi}$.

PolarQuant algorithm applies a random preconditioning prior to quantization. This preconditioning involves multiplying each embedding vector by a shared random sketch matrix $\boldsymbol{S}$ with i.i.d. normal entries. By the Johnson-Lindenstrauss (JL) lemma [10], this preconditioning preserves the norms and inner products of the embedding vectors with minimal distortion. A key property of this preconditioning, which we will leverage in our later analysis, is that the embedding vectors after preconditioning follow a multivariate normal distribution. This is formalized in the following fact.

Fact 3. For any vector $\boldsymbol{x} \in \mathbb{R}^{d}$ if $\boldsymbol{S} \in \mathbb{R}^{m \times d}$ is a random matrix with i.i.d. normal entries $\boldsymbol{S}_{i, j} \sim \mathcal{N}(0,1)$, then the vector $\boldsymbol{S} \cdot \boldsymbol{x}$ has multivariate normal distribution $\boldsymbol{S} \cdot \boldsymbol{x} \sim \mathcal{N}\left(0,\|\boldsymbol{x}\|_{2} \cdot \boldsymbol{I}_{m}\right)$.

The following lemma establishes the distribution of the polar angle of a point ( $x, y$ ) in dimension 2, where the $x$ and $y$ coordinates are independent samples from the Euclidean norm of multivariate normal random variables.

Lemma 1. For any positive integer $d$, if $x, y \geq 0$ are two i.i.d. random variables with generalized gamma distribution with probability density function $f_{Z}(z)=\frac{2}{2^{\text {d/2 }} \cdot \Gamma(d / 2)} z^{d-1} \exp \left(-z^{2} / 2\right)$, then the angle variable $\theta:=\tan ^{-1}(y / x)$ follows the probability density function:

$$
f_{\Theta}(\theta)=\frac{\Gamma(d)}{2^{d-2} \cdot \Gamma(d / 2)^{2}} \cdot \sin ^{d-1}(2 \theta)
$$

Additionally, $\mathbb{E}[\Theta]=\pi / 4$ and $\operatorname{Var}(\Theta)=O(1 / \sqrt{d})$.

See Appendix B for a proof.

## 3 PolarQuant

We now describe our approach of quantizing angles in polar coordinates and using it to the KV cache problem. In Section 3.1, we introduce how to recursively transform Cartesian vector to polar coordinates. In Section 3.2, we provide an analysis of polar angle distributions with preconditioning. In Section 3.3, we explain details of quantization polar transformed embeddings and practical implementation.

### 3.1 Recursive Polar Transformation

There are various methods to derive the polar representation of $\mathbb{R}^{d}$. Here we propose a polar transformation that can be recursively computed from the Cartesian coordinates of points in $\mathbb{R}^{d}$. Throughout this work, we assume that $d$ is an integer power of 2 .

At a high level, our approach begins by grouping pairs of coordinates of a $d$-dimensional vector $\boldsymbol{x}$ and transforming each pair into 2D polar coordinates. This produces $d / 2$ radius and angle pairs. Next, we gather $d / 2$ of radii and apply the polar transform to them. This procedure is recursively repeated $\log _{2} d$ times and the final output consists of a single final radius and a collection of $1,2,4, \ldots, d / 2$ dimensional angle vectors. A formal definition is provided in Definition 1.

Definition 1 (Cartesian to Polar Transformation). For any integer power of two $d$, the polar representation of any vector $\boldsymbol{x} \in \mathbb{R}^{d}$ includes $d-1$ angles and a radius. Angles are organized into a collection of $\log _{2} d$ vector of angles $\psi^{(1)}, \psi^{(2)}, \ldots \psi^{\left(\log _{2} d\right)}$ such that $\psi^{(1)} \in[0,2 \pi)^{d / 2}$ and $\psi^{(\ell)} \in[0, \pi / 2]^{d / 2^{\ell}}$ for any $\ell \geq 2$. In other words, the angles are computed in $\log _{2} d$ levels and there are $d / 2^{\ell}$ angles in level $l$. These angles are defined by the following relation for $\ell \in\left\{2,3, \ldots \log _{2} d\right\}$ :

$$
\begin{aligned}
\psi_{j}^{(1)} & :=\tan ^{-1}\left(\boldsymbol{x}_{2 j} / \boldsymbol{x}_{2 j-1}\right) \quad \text { for } j \in[d / 2] \\
\psi_{j}^{(\ell)} & :=\tan ^{-1}\left(\frac{\left\|\boldsymbol{x}_{(j-1 / 2) 2^{\ell}+1: j 2^{\ell}}\right\|_{2}}{\left\|\boldsymbol{x}_{(j-1) 2^{\ell}+1:(j-1 / 2) 2^{\ell}}\right\|_{2}}\right) \quad \text { for } j \in\left[d / 2^{\ell}\right]
\end{aligned}
$$

The reverse of this transformation maps the angles and the radius of any point to its Cartesian
vector representation using the following equation:

$$
\boldsymbol{x}_{i}=\|\boldsymbol{x}\|_{2} \cdot \prod_{\ell=1}^{\log _{2} d}\left(\cos \psi_{\left\lfloor\frac{i}{2^{\ell}}\right\rfloor}^{(\ell)}\right)^{\mathbf{1}_{\left\{\left(i \bmod 2^{\ell}\right) \leq 2^{\ell-1}\right\}}} \cdot \prod_{\ell=1}^{\log _{2} d}\left(\sin \psi_{\left\lfloor\frac{i}{2^{\ell}}\right\rfloor}^{(\ell)}\right)^{\mathbf{1}_{\left\{\left(i \bmod 2^{\ell}\right)>2^{\ell-1}\right\}}}
$$

A visual diagram of the algorithm is shown in Fig. 1 and the pseudocode is provided in Algorithm 1 (see Polar procedure). In what follows, we analyze the distribution of angles generated in each quantization level.

### 3.2 Distribution of Polar Angles Under Random Preconditioning

One of our primary objectives is to eliminate the need for explicit normalization (e.g., minimum/maximum values) of the KV cache data prior to quantization, thereby reducing quantization overhead. To achieve this, our algorithm applies random preconditioning to the embedding vectors. This preconditioning involves multiplying each embedding vector by a shared random sketch matrix $\boldsymbol{S}$ with i.i.d. normal entries. By the Johnson-Lindenstrauss (JL) lemma [10], this preconditioning preserves the norms and inner products* of the embedding vectors with minimal distortion. A key property of this preconditioning, which we will leverage in our later analysis, is that the embedding vectors after preconditioning follow a multivariate normal distribution. This has been formalized in Fact 3.

During the preconditioning stage, the sketch is applied to all embedding vectors in the KV cache, allowing the analysis of PolarQuant to effectively treat the vectors being quantized as samples from a multivariate normal distribution. So for the analysis and design of PolarQuant we can assume without loss of generality that our goal is to quantize a random vector with multivariate Gaussian distribution. A critical insight is that the distribution of angles after random preconditioning becomes predictable and can be analytically derived, which enables the design of optimal quantization schemes.

The polar distribution of a Gaussian vector is derived in the following lemma.
Lemma 2 (Distribution of a Gaussian Vector Under Polar Transformation). For an integer power of two $d$, suppose that $\boldsymbol{x} \sim \mathcal{N}\left(0, I_{d}\right)$ is a random zero mean isotropic Gaussian random variable in dimension $d$. Let $\psi_{d}(\boldsymbol{x}):=\left(\psi^{(1)}, \psi^{(2)}, \ldots \psi^{\left(\log _{2} d\right)}\right)$ denote the set of polar angles obtained by applying the polar transformation defined in Definition 1 on $\boldsymbol{x}$. Denote the radius of $\boldsymbol{x}$ by $r=\|\boldsymbol{x}\|_{2}$. The joint probability density function for $\left(r, \psi^{(1)}, \psi^{(2)}, \ldots \psi^{\left(\log _{2} d\right)}\right)$ is the following:

$$
\begin{equation*}
f_{R, \Psi_{d}}\left(r, \psi_{d}(\boldsymbol{x})\right)=f_{R}(r) \cdot \prod_{\ell=1}^{\log _{2} d} f_{\Psi^{(\ell)}}\left(\psi^{(\ell)}\right), \tag{1}
\end{equation*}
$$

where $f_{R}(r)$ is the p.d.f. defined in Fact 1, $f_{\Psi^{(1)}}$ is p.d.f. of the uniform distribution over $[0,2 \pi)^{d / 2}$ :

$$
f_{\Psi^{(1)}}:[0,2 \pi)^{d / 2} \rightarrow(2 \pi)^{-d / 2}
$$

[^1]and for every $\ell \in\left\{2,3, \ldots \log _{2} d\right\}$ the p.d.f. $f_{\Psi^{(\ell)}}$ is the following:
$$
\begin{aligned}
f_{\Psi^{(\ell)}} & :[0, \pi / 2]^{d / 2^{\ell}} \rightarrow \mathbb{R}_{+} \\
f_{\Psi^{(\ell)}}(\boldsymbol{\psi}) & =\prod_{i=1}^{d / 2^{\ell}} \frac{\Gamma\left(2^{\ell-1}\right)}{2^{2^{\ell-1}-2} \cdot \Gamma\left(2^{\ell-2}\right)^{2}} \sin ^{\left(2^{\ell-1}-1\right)}\left(2 \psi_{i}\right)
\end{aligned}
$$

Proof. The proof is by induction on $d$. First for the base of induction we prove the result in dimension $d=2$. So we prove that for a 2 -dimensional random Gaussian vector $\boldsymbol{y}=\left(y_{1}, y_{2}\right) \in \mathbb{R}^{2}$ if ( $r, \theta$ ) is the polar representation of this vector then following holds:

$$
f_{R, \Theta}(r, \theta)=\frac{1}{2 \pi} \cdot r \exp \left(-r^{2} / 2\right),
$$

To prove this, let $f_{Y}(\boldsymbol{y})$ be the probability density function of the vector random variable $\boldsymbol{y}$. We know $\boldsymbol{y}$ has a normal distribution so we have:

$$
f_{R, \Theta}(r, \theta)=r \cdot f_{Y}(\boldsymbol{y})=r \cdot \frac{1}{2 \pi} e^{-\frac{y_{1}^{2}+y_{2}^{2}}{2}}=\frac{1}{2 \pi} \cdot r e^{-\frac{r^{2}}{2}},
$$

where the first equality above follows from the change of variable from $\left(y_{1}, y_{2}\right)$ to $r=\sqrt{y_{1}^{2}+y_{2}^{2}}$ and $\theta=\tan ^{-1}\left(y_{2} / y_{1}\right)$. This proves the base of induction for $d=2$.

Now we prove the inductive step. Suppose that the lemma holds for dimension $d / 2$ and we want to prove it for dimension $d$. Denote $\theta:=\psi^{\left(\log _{2} d\right)}, \phi_{1}:=\left(\psi_{1: d / 2^{l+1}}^{(l)}\right)_{\ell=1}^{\log _{2} d-1}, \phi_{2}:=\left(\psi_{d / 2^{\ell+1}+1: d / 2^{\ell}}^{(l)}\right)_{\ell=1}^{\log _{2} d-1}$, $r_{1}:=\left\|\boldsymbol{x}_{1: d / 2}\right\|$, and $r_{2}:=\left\|\boldsymbol{x}_{d / 2+1: d}\right\|$. Essentially we sliced all the angle vectors $\psi^{(\ell)}$ in half and named the collection of first half vectors $\phi_{1}$ and the collection of second halves $\phi_{2}$. Using the definition of $\psi^{(\ell)}$ 's in Definition 1, $\phi_{1}$ is exactly the polar transformation of $\boldsymbol{x}_{1: d / 2}$, and $\phi_{2}$ is the polar transformation of $\boldsymbol{x}_{d / 2+1: d}$, so by the definition of $\psi_{d}(\boldsymbol{x})$ in the lemma statement we have $\phi_{1}=\psi_{d / 2}\left(\boldsymbol{x}_{1: d / 2}\right)$ and $\phi_{2}=\psi_{d / 2}\left(\boldsymbol{x}_{d / 2+1: d}\right)$. Thus, we can write:

$$
\begin{align*}
& f_{R, \Psi_{d}}\left(r, \psi_{d}(\boldsymbol{x})\right)=f_{R, \Theta, \Phi_{1}, \Phi_{2}}\left(r, \theta, \boldsymbol{\phi}_{1}, \boldsymbol{\phi}_{2}\right) \\
& =r \cdot f_{R_{1}, R_{2}, \Phi_{1}, \Phi_{2}}\left(r \cos \theta, r \sin \theta, \boldsymbol{\phi}_{1}, \boldsymbol{\phi}_{2}\right) \\
& =r \cdot f_{R_{1}, \Phi_{1}}\left(r \cos \theta, \boldsymbol{\phi}_{1}\right) \cdot f_{R_{2}, \Phi_{2}}\left(r \sin \theta, \boldsymbol{\phi}_{2}\right) \\
& =r \cdot f_{R, \Psi_{d / 2}}\left(r_{1}, \boldsymbol{\phi}_{1}\right) \cdot f_{R, \Psi_{d / 2}}\left(r_{2}, \boldsymbol{\phi}_{2}\right) \tag{2}
\end{align*}
$$

where the third line above follows from the change of variable from $\left(r_{1}, r_{2}\right)=(r \cos \theta, r \sin \theta)$ to $r=\sqrt{r_{1}^{2}+r_{2}^{2}}$ and $\theta=\tan ^{-1}\left(r_{2} / r_{1}\right)$. In the fourth line above we used the definition of $\theta= \psi^{\left(\log _{2} d\right)}=\tan ^{-1}\left(\frac{\left\|\boldsymbol{x}_{d / 2+1: d}\right\|_{2}}{\left\|\boldsymbol{x}_{1: d / 2}\right\|_{2}}\right)$ from Definition 1.
Now if we let $f_{\Psi_{d / 2}}\left(\phi_{1}\right):=\prod_{\ell=1}^{\log _{2} d-1} f_{\Psi^{(\ell)}}\left(\psi_{1: d / 2^{\ell+1}}^{(\ell)}\right)$ and $f_{\Psi_{d / 2}}\left(\phi_{2}\right):=\prod_{\ell=1}^{\log _{2} d-1} f_{\Psi^{(\ell)}}\left(\psi_{d / 2^{\ell+1}+1: d / 2^{\ell}}^{(\ell)}\right)$, by the inductive hypothesis we have $f_{R, \Psi_{d / 2}}\left(r_{1}, \phi_{1}\right)=\frac{2}{2^{d / 4} \cdot \Gamma(d / 4)} r_{1}^{d / 2-1} \exp \left(-r_{1}^{2} / 2\right) \cdot f_{\Psi_{d / 2}}\left(\phi_{1}\right)$ and
$f_{R, \Psi_{d / 2}}\left(r_{2}, \phi_{2}\right)=\frac{2}{2^{d / 4} \cdot \Gamma(d / 4)} r_{2}^{d / 2-1} \exp \left(-r_{2}^{2} / 2\right) \cdot f_{\Psi_{d / 2}}\left(\phi_{2}\right)$. Plugging these values into Eq. (2) gives:

$$
\begin{align*}
f_{R, \Psi_{d}}\left(r, \psi_{d}(\boldsymbol{x})\right) & =\frac{4 \cdot\left(r_{1} r_{2}\right)^{d-1}}{2^{d / 2} \Gamma(d / 4)^{2}} \exp \left(-r^{2} / 2\right) \cdot f_{\Psi_{d / 2}}\left(\phi_{1}\right) \cdot f_{\Psi_{d / 2}}\left(\phi_{2}\right) \\
& =\frac{2 r^{d-1} \cdot \sin ^{d / 2-1}(2 \theta)}{2^{3 d / 2-2} \cdot \Gamma(d / 4)^{2}} e^{-r^{2} / 2} \cdot f_{\Psi_{d / 2}}\left(\phi_{1}\right) \cdot f_{\Psi_{d / 2}}\left(\phi_{2}\right) \\
& =f_{R}(r) \cdot \frac{\Gamma(d / 2) \cdot \sin ^{d / 2-1}(2 \theta)}{2^{d / 2-2} \cdot \Gamma(d / 4)^{2}} f_{\Psi_{d / 2}}\left(\phi_{1}\right) \cdot f_{\Psi_{d / 2}}\left(\phi_{2}\right) \\
& =f_{R}(r) \cdot f_{\Psi_{d}}\left(\psi_{d}(\boldsymbol{x})\right) \tag{3}
\end{align*}
$$

which completes the inductive proof of this lemma.

Lemma 2 demonstrates that the angles of Gaussian vectors in polar coordinates have independent distributions, as the probability density function is separable. Moreover, all angles within the same level share identical distributions. Specifically, at level $\ell$ all angles follow the distribution $\psi_{i}^{(\ell)} \sim \prod_{i=1}^{d / 2^{\ell}} \frac{\Gamma\left(2^{\ell-1}\right)}{2^{2^{\ell-1}-2} \cdot \Gamma\left(2^{\ell-2}\right)^{2}} \sin ^{2^{\ell-1}-1}\left(2 \psi_{i}^{(\ell)}\right)$. This density becomes increasingly concentrated around $\pi / 4$, particularly at higher levels $\ell$. This property is highly beneficial for reducing quantization error for the angles at higher levels.

### 3.3 PolarQuant Algorithm and Main Theorem

PolarQuant starts by first applying random preconditioning, then transforming the vectors into polar coordinates, and finally quantizing each angle. Since Lemma 2 shows that the angles in polar coordinates are independent random variables, each angle can be quantized independently to minimize the total mean squared error. Jointly quantizing multiple angle coordinates offers no additional benefit due to their independence, making our approach both computationally efficient and effective. Therefore, we can focus on one angle at level $l$ and design optimal quantization scheme for it so as to minimize the mean squared error.

Consider an angle $\psi_{i}^{(\ell)}$ at some level $\ell$. According to Lemma 2, its values lie within the range $[0, \pi / 2]$ for $\ell \geq 2$ and for $\ell=1$ it takes values in the range $[0,2 \pi)$ with a probability density function given by $f_{\ell}\left(\psi_{i}^{(\ell)}\right):=\frac{\Gamma\left(2^{\ell-1}\right)}{2^{2^{\ell-1}-2} \cdot \Gamma\left(2^{\ell-2}\right)^{2}} \sin ^{2^{\ell-1}-1}\left(2 \psi_{i}^{(\ell)}\right)$. The goal of quantization to $b$-bits is to partition the range $[0, \pi / 2]$ (or $[0,2 \pi)$ in case of $\ell=1$ ) into $2^{b}$ intervals $I_{1}^{(\ell)}, I_{2}^{(\ell)}, \cdots I_{2^{b}}^{(\ell)}$ and find corresponding centroids $\theta_{1}^{(\ell)}, \theta_{2}^{(\ell)}, \ldots \theta_{2^{b}}^{(\ell)}$ such that the following is mean squared error is minimized:

$$
\begin{equation*}
\underset{\psi_{i}^{(\ell)} \sim f_{\ell}\left(\psi_{i}^{(\ell)}\right)}{\mathbb{E}}\left[\sum_{j \in\left[2^{b}\right]: \psi_{i}^{(\ell)} \in I_{j}^{(\ell)}}\left|\psi_{i}^{(\ell)}-\theta_{j}^{(\ell)}\right|^{2}\right] . \tag{4}
\end{equation*}
$$

This problem is a continuous analog of the k -means clustering problem in dimension 1. Since we have an explicit formula for the p.d.f. of angle $\psi_{i}^{(\ell)} \sim f_{\ell}\left(\psi_{i}^{(\ell)}\right)=\frac{\Gamma\left(2^{\ell-1}\right)}{2^{2^{\ell-1}-2} \cdot \Gamma\left(2^{\ell-2}\right)^{2}} \sin ^{2^{\ell-1}}-1\left(2 \psi_{i}^{(\ell)}\right)$ the optimal interval partitions and centroids for Eq. (4) can be efficiently computed using numerical

```
Algorithm 1 PolarQuant
    input: embedding $\boldsymbol{X} \in \mathbb{R}^{n \times d}$, precondition matrix $\boldsymbol{S} \in \mathbb{R}^{d \times d}$, bit width $b$
    // Cartesian to Polar transform
    $\mathbf{R}_{i}, \boldsymbol{\Psi}_{i}^{(1)}, \ldots, \boldsymbol{\Psi}_{i}^{\left(\log _{2} d\right)} \leftarrow \operatorname{Polar}\left(\boldsymbol{X}_{i} \cdot \boldsymbol{S}\right)$ for $i \in[n]$
    // Codebook Construction
    Find partition intervals and centroids $\left(I_{k}^{(\ell)}, \theta_{k}^{(\ell)}\right)_{k \in\left[2^{b}\right]}$ of $\boldsymbol{\Psi}^{(\ell)} \in \mathbb{R}^{n \times\left(d / 2^{\ell}\right)}$ that minimize the cost
    in Eq. (4) for $\ell \in\left[\log _{2} d\right]$ (See Section 4.1 for details)
    // Angles Quantization
    $\boldsymbol{J}_{i}^{(\ell)} \leftarrow \operatorname{Quant}\left(\boldsymbol{\Psi}_{i}^{(\ell)},\left(I_{k}^{(\ell)}, \theta_{k}^{(\ell)}\right)_{k \in\left[2^{b}\right]}\right)$ for $i \in[n]$ and $\ell \in\left[\log _{2} d\right]$
    output: $\mathbf{R} \in \mathbb{R}^{n \times 1}, \boldsymbol{J}^{(1)} \in\left[2^{b}\right]^{n \times d / 2}, \ldots, \boldsymbol{J}^{\left(\log _{2} d\right)} \in\left[2^{b}\right]^{n \times 1},\left(I_{k}^{(\ell)}, \theta_{k}^{(\ell)}\right)_{k \in\left[2^{b}\right]}$
    Procedure Polar (y)
    $\boldsymbol{r}^{(0)} \leftarrow \boldsymbol{y} \in \mathbb{R}^{d}$
    for $\ell=1, \ldots, \log _{2} d$ do
        for $j=1, \ldots, d / 2^{\ell}$ do
            $\boldsymbol{\psi}_{j}^{(\ell)} \leftarrow \tan ^{-1}\left(\boldsymbol{r}_{2 j}^{(\ell-1)} / \boldsymbol{r}_{2 j-1}^{(\ell-1)}\right)$
            $\boldsymbol{r}_{j}^{(\ell)} \leftarrow\left\|\boldsymbol{r}_{2 j-1: 2 j}^{(\ell-1)}\right\|_{2}$
        end for
    end for
    output: $\boldsymbol{r}^{\left(\log _{2} d\right)}, \boldsymbol{\psi}^{(1)}, \ldots, \boldsymbol{\psi}^{\left(\log _{2} d\right)}$
    Procedure Quant $\left(\boldsymbol{\psi},\left(I_{k}, \theta_{k}\right)_{k \in\left[2^{b}\right]}\right)$
    $\boldsymbol{j}_{i} \leftarrow \operatorname{argmin}_{k \in\left[2^{b}\right]}\left|\theta_{k}-\boldsymbol{\psi}_{i}\right|$ for $i \in\left[d^{\prime}\right]$ s.t. $\boldsymbol{\psi} \in \mathbb{R}^{d^{\prime}}$
    output: $j$
    Procedure DeQuant $\left(\boldsymbol{r},\left(\boldsymbol{j}^{(\ell)}\right)_{\ell \in\left[\log _{2} d\right]},\left(\theta_{k}^{(\ell)}\right)_{k \in\left[2^{b}\right]}, \boldsymbol{S}\right)$
    for $\ell=\log _{2} d, \ldots, 1$ do
        for $j=1, \ldots, d / 2^{\ell}$ do
            $i \leftarrow \boldsymbol{j}_{j}^{(\ell)}$
            $\boldsymbol{r}_{2 j-1}^{(\ell-1)} \leftarrow \boldsymbol{r}_{j}^{(\ell)} \cdot \cos \theta_{i}^{(\ell)}$
            $\boldsymbol{r}_{2 j}^{(\ell-1)} \leftarrow \boldsymbol{r}_{j}^{(\ell)} \cdot \sin \theta_{i}^{(\ell)}$
        end for
    end for
    output: $\boldsymbol{r}^{(0)} \cdot \boldsymbol{S}^{\top}$
```

methods. For example, one can run k -means clustering on the gathered angle values which can be considered samples from the distribution. This approach ensures minimal quantization error for each angle independently and the overall reconstruction error as well.

We provide a pseudocode of PolarQuant in Algorithm 1. Our main result and error bound are proved in the following.

Theorem 1. For a $d$-dimensional vector $\boldsymbol{x} \sim N\left(0, I_{d}\right)$, the polar quantization scheme in Algorithm 1
uses $O(\log 1 / \varepsilon)$ bits per coordinate + the space necessary to store $\|\boldsymbol{x}\|_{2}$, while reconstructing a vector $\boldsymbol{x}^{\prime}$ from such a representation satisfying

$$
\mathbb{E}\left[\left\|\boldsymbol{x}-\boldsymbol{x}^{\prime}\right\|_{2}^{2}\right]=\varepsilon \cdot\|\boldsymbol{x}\|_{2}^{2}
$$

The proof of Theorem 1 can be found in Appendix C. We note that a scheme which uses a deterministic $\varepsilon$-net $\mathcal{N}$ of the unit sphere $\mathbb{S}^{d-1}$, with $|\mathcal{N}|=O(1 / \varepsilon)^{d}$ and rounds the vector $\hat{\boldsymbol{x}}=\boldsymbol{x} /\|\boldsymbol{x}\|_{2}$ also uses $O(\log 1 / \varepsilon)$ bits per coordinate while achieving the above bounds in the worst case instead of in expectation over the Gaussian distribution. But our construction (i) gives the flexibility to vary the size of the codebook used per each level depending on the resource constraints and as the above theorem shows, can approach the same quality as pinning to an $\varepsilon$-net on average, (ii) does not need to store a $|\mathcal{N}|$-size codebook which is impractical even for modest sizes of $d$ and (iii) has a fast decoding/encoding implementation.

## 4 KV Cache Quantization with PolarQuant

In this section, we describe how PolarQuant can be applied to the KV cache problem and our practical implementation. Formally, given a stream of $\left(\boldsymbol{q}_{1}, \boldsymbol{k}_{1}, \boldsymbol{v}_{1}\right), \ldots,\left(\boldsymbol{q}_{n}, \boldsymbol{k}_{n}, \boldsymbol{v}_{n}\right)$, where $\boldsymbol{q}_{i}, \boldsymbol{k}_{i}, \boldsymbol{v}_{i} \in \mathbb{R}^{d}$ are query, key and value embeddings at $i$-th generation step for all $i \in[n]$. Let $\boldsymbol{K}_{: i}, \boldsymbol{V}_{: i} \in \mathbb{R}^{i \times d}$ be matrices defined by stacking $\boldsymbol{k}_{1}, \ldots, \boldsymbol{k}_{i}$ and $\boldsymbol{v}_{1}, \ldots, \boldsymbol{v}_{j}$ in their rows, respectively. The goal is to compute:

$$
\begin{equation*}
\operatorname{softmax}\left(\frac{\boldsymbol{K}_{: i} \cdot \boldsymbol{q}_{i}}{\sqrt{d}}\right)^{T} \cdot \boldsymbol{V}_{: i} \tag{5}
\end{equation*}
$$

For an efficient token generation, the KV cache at $i$-th generation step ( $\boldsymbol{K}_{: i}, \boldsymbol{V}_{: i}$ ) are stored in the memory. To reduce the memory space, we invoke PolarQuant (Algorithm 1) on these embeddings. Let $\widehat{\boldsymbol{K}}_{: i}, \widehat{\boldsymbol{V}}_{: i} \in \mathbb{R}^{i \times d}$ be their dequantizations using DeQUANT procedure in Algorithm 1. Then, we estimate Eq. (5) by computing

$$
\begin{equation*}
\operatorname{softmax}\left(\frac{\widehat{\boldsymbol{K}}_{: i} \cdot \boldsymbol{q}_{i}}{\sqrt{d}}\right) \cdot \widehat{\boldsymbol{V}}_{: i} \tag{6}
\end{equation*}
$$

Note that the naïve cache requires $d \cdot b_{\text {FPN }}$ memory space to store each $d$-dimensional embedding where $b_{\mathrm{FPN}}$ is the number of bits to represent a single floating-point number. If we quantize $\log _{2} d$ level angles with $b$ bits each and keep centroids in $b_{\text {FPN }}$ bits, the memory space becomes $\left(b_{\mathrm{FPN}}+(d-1) b\right)$. For example, Llama-3.1-8B-Instruct is represented by $b_{\mathrm{FPN}}=16$ bits and has $d=128$. For $b=3$, we can save the memory space 4.008 times. In Section 5, the PolarQuant with KV cache marginally degrades the performance of LLMs on various tasks.

### 4.1 Practical Implementation

The PolarQuant algorithm recursively reduces the dimension of radii by half until the input has dimension 1. We recurse on the polar transformation for a constant $L=4$ levels. Thus, for an embedding of dimension $d$, we obtain $d / 16$-dimensional radii and $15 d / 16$ angle values. We also

![](https://cdn.mathpix.com/cropped/3d1c12a2-ff39-4daa-b19b-e0a8fb1f6256-11.jpg?height=704&width=1598&top_left_y=195&top_left_x=257)
Figure 2: Distributions of angles of polar transformed key embeddings (a) with and (b) without random preconditioning. Preconditioning flattens the angle distribution and removes outliers which allows angle quantization more accurately.

define different numbers of bits for each quantization level: $b=4$ bits for the first level, and $b=2$ bits for the remaining levels. This is because the range of angle at the first level $[0,2 \pi)$ is 4 times wider than the others $[0, \pi / 2]$. Consequently, the representation of a block of 16 coordinates uses $b_{\mathrm{FPN}}+32+8+4+2=b_{\mathrm{FPN}}+46$ bits that translates to $62 / 16=3.875$ bits per coordinate when $b_{\mathrm{FPN}}=16$ bits.

We implement PolarQuant using the Pytorch [31] framework. Since the smallest data type is represented in 8 bits (torch.uint8), we pack quantized angle indices into 8 -bit unit. To accelerate computation on GPU clusters, we implement CUDA kernels for two key operations: (1) the product of query vectors with the dequantized key cache, i.e., $\widehat{\boldsymbol{K}}_{: i} \cdot \boldsymbol{q}_{i}$, and (2) the product of attention scores with the dequantized value cache as per Eq. (6). For the preconditioning matrix $\boldsymbol{S}$, we generate a random rotational matrix. The matrix $\boldsymbol{S}$ is shared across key and value embeddings, as well as all layers and attention heads in the Transformer architecture.

For angle codebook construction (line 3 in Algorithm 1), we use the 1-D k-means++ clustering on either online angles obtained from polar-transformed inputs or offline precomputed angles. Both approaches approximate the solution to Eq. (4) by discretizing with samples from angle distributions. While online approach requires additional clustering computation during every prefill stage, this onetime cost is offset by improved performance compared to the offline approach. We present detailed runtime and performance comparisons in Section 5.

## 5 Experiments

All experiments are performed with a single NVIDIA RTX A6000 GPU with 48GB VRAM.

![](https://cdn.mathpix.com/cropped/3d1c12a2-ff39-4daa-b19b-e0a8fb1f6256-12.jpg?height=811&width=1552&top_left_y=178&top_left_x=281)
Figure 3: Needle-In-A-Haystack test using Llama-3.1-8B-Instruct. The test spans different depths and context lengths ranging from 4 K to 104 K . Green/red colors indicate high/low recall scores (higher is better). PolarQuant shows the best performance.

### 5.1 Random Precondition on KV Cache

We first explore the effectiveness of preconditioning. In particular, we choose a single prompt from Qasper dataset in LongBench [5] and extract the corresponding KV cache. To observe how preconditioning improves, we transform the KV cache into 4 -level polar coordinates and plot their angle distributions of the key cache. Note that the first level angles are range in $[0,2 \pi)$ and the rest are in $[0, \pi / 2]$. The results are illustrated in Fig. 2. As shown in Lemma 2, the distribution of angles get predictably sharper around $\pi / 4$ as the level increases. Moreover, we observe that at the first level the preconditioning flattens the angle distribution and removes outliers. This allows us to quantize angles in the KV cache more accurately.

### 5.2 Needle-In-A-Haystack

Next we evaluate our method for the "Needle-In-A-Haystack" test [19]. It asks the model to retrieve the information in a given sentence where the sentence (the "needle") is placed in an arbitrary location of a long document (the "haystack"). We follow the same setting from Fu et al. [14] and use the Llama-3.1-8B-Instruct to run the test. We vary the input sequence lengths from 4 K to 104 K . The evaluation is based on the recall score by comparing the hidden sentence. We compare PolarQuant to SnapKV [24], PyramidKV [8] and KIVI [26], where we use their implementations from [18]. All methods are set to a compression ratio of 0.25 , i.e., required memory is $\times 0.25$ the full KV cache. Specifically, we run our algorithm with and without the preconditioning and refer to them as PolarQuant-R and PolarQuant, respectively. In Fig. 3, we observe that quantization methods (e.g., KIVI, PolarQuant) outperform token-level compression methods (e.g., SnapKV, PyramidKV). PolarQuant shows better scores than KIVI. Additionally, PolarQuant shows a marginally better score than PolarQuant-R.

Table 1: LongBench-V1 [5] results of various KV cache compression methods on Llama-3.1-8BInstruct. The best values among compression methods are indicated in bold.
| Method | SQA | MQA | Task |  | Syn | Code | Average |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
|  |  |  | Sum | Few |  |  |  |
| Exact (16 bits) | 45.71 | 45.32 | 26.69 | 68.62 | 59.25 | 46.17 | 48.63 |
| Snapkv | 38.23 | 42.61 | 19.07 | 64.65 | 59.60 | 43.28 | 44.57 |
| HeadKV | 39.45 | 42.69 | 19.77 | 68.07 | 59.48 | 42.60 | 45.34 |
| PyramidKV | 36.80 | 41.54 | 18.91 | 64.88 | 59.68 | 42.38 | 44.03 |
| StreamingLLM | 25.68 | 35.79 | 20.90 | 56.91 | 58.81 | 32.07 | 38.36 |
| KIVI | 43.38 | 37.81 | 27.44 | 68.60 | 58.67 | 44.29 | 46.70 |
| PolarQuant | 44.03 | 44.34 | 27.32 | 68.68 | 59.82 | 44.46 | 48.11 |
| PolarQuant-R (offline) | 44.71 | 44.72 | 26.43 | 68.58 | 60.08 | 45.20 | 48.29 |
| PolarQuant-R (online) | 45.45 | 45.13 | 26.42 | 68.54 | 59.57 | 45.13 | 48.37 |


### 5.3 End-to-end Generation on LongBench

We run various KV cache compression algorithms for LongBench datasets [5], which encompasses diverse long-text scenarios including single/multi-document question-answering (SQA/MQA), summarization (Sum), few-shot learning (Few), synthetic tasks (Syn), and code completion (Code). Since the number of generated tokens is small compared to the input sequence length across all datasets, we preserve all new streamed query, key, and value pairs from the generation stage in full precision ( 16 bits) for all methods. We evaluate PolarQuant against the baseline methods using in Section 5.2 as well as StreamingLLM [38] and HeadKV [13] on Llama-3.1-8B-Instruct.

We investigate two variants of PolarQuant-R: one using online codebook construction and another using offline one discussed in Section 4.1. The online variant performs clustering for each individual input prompt and layer, while the offline one employs a single precomputed codebook that is shared across all input prompts, layers, and attention heads. This offline approach is supported by our findings that the angle distribution, when preconditioned, remains consistent regardless of the input.

As reported in Table 1, our methods achieve superior performance compared to other methods, i.e., the average performance scores are higher by a large margin. This justifies the performance benefits of the quantization of polar coordinates. Moreover, the preconditioned variants (PolarQuant-R) generally demonstrates better performance than the non-preconditioned version. Among them, the online variant performs slightly better than the offline one.

### 5.4 Runtime Analysis

We evaluate wall-clock runtimes of both prefill and token generation stages. Using the Llama model with an input prompt length of 16,384 , we measure the time to generate 1,024 tokens for each method. Table 2 summaries the result. Token eviction approaches (SnapKV, PyramidKV, and HeadKV) demonstrate faster generation times compared to exact and quantization methods, though at the cost of lower quality. Among quantization approaches, our PolarQuant algorithms achieve $14 \%$ faster generation time than the KIVI while maintaining superior performance. These

Table 2: Wall-clock runtime comparisons of various KV cache compression methods. The input sequence length is $n=16,384$ and the number of generated tokens is 1,024 .
| Method | Prefill Time (sec) | Generation Time (sec) |
| :--- | :--- | :--- |
| Exact (16 bits) | 2.934 | 38.374 |
| SnapKV | 3.438 | 34.053 |
| PyramidKV | 3.428 | 32.732 |
| HeadKV | 3.300 | 34.401 |
| KIVI | 3.590 | 49.564 |
| PolarQuant | 11.623 | 43.652 |
| PolarQuant-R (online) | 11.633 | 44.448 |
| PolarQuant-R (offline) | 3.364 | 44.097 |


results demonstrate that PolarQuant offers advantages in both computational efficiency and model performance. To achieve faster prefill times, we recommend using offline codebook construction, as it significantly reduces runtime by eliminating the need for clustering, though this results in a modest performance trade-off. We leave even better codebook construction approaches for future research.

## 6 Conclusion

We propose PolarQuant, a novel quantization method applied to angles in polar coordinates. We connect it to the random preconditioning which allows us to formalize angle distribution to be quantized. We provide rigorous theoretical bounds on quantization error. When applied to the KV cache compression problem, PolarQuant significantly reduces memory requirements during LLM inference while maintaining model performance. The principles underlying our method extend beyond KV cache compression, offering potential applications in LLM weight quantization and general vector similarity search problems.

## References

[1] Achiam, J., Adler, S., Agarwal, S., Ahmad, L., Akkaya, I., Aleman, F. L., Almeida, D., Altenschmidt, J., Altman, S., Anadkat, S., et al. Gpt-4 technical report. arXiv preprint arXiv:2303.08774, 2023.
[2] Ainslie, J., Lee-Thorp, J., de Jong, M., Zemlyanskiy, Y., Lebron, F., and Sanghai, S. Gqa: Training generalized multi-query transformer models from multi-head checkpoints. In Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, pp. 4895-4901, 2023.
[3] Anthropic. Claude, 2024. https://www.anthropic.com/news/claude-3-family.
[4] Ashkboos, S., Mohtashami, A., Croci, M. L., Li, B., Cameron, P., Jaggi, M., Alistarh, D.,

Hoefler, T., and Hensman, J. Quarot: Outlier-free 4-bit inference in rotated llms. arXiv preprint arXiv:2404.00456, 2024.
[5] Bai, Y., Lv, X., Zhang, J., Lyu, H., Tang, J., Huang, Z., Du, Z., Liu, X., Zeng, A., Hou, L., Dong, Y., Tang, J., and Li, J. Longbench: A bilingual, multitask benchmark for long context understanding. arXiv preprint arXiv:2308.14508, 2023.
[6] Beltagy, I., Peters, M. E., and Cohan, A. Longformer: The long-document transformer. arXiv preprint arXiv:2004.05150, 2020.
[7] Bhattacharya, A., Freund, Y., and Jaiswal, R. On the k-means/median cost function. Information Processing Letters, 177:106252, 2022. URL https://arxiv.org/pdf/1704.05232.
[8] Cai, Z., Zhang, Y., Gao, B., Liu, Y., Liu, T., Lu, K., Xiong, W., Dong, Y., Chang, B., Hu, J., et al. Pyramidkv: Dynamic kv cache compression based on pyramidal information funneling. arXiv preprint arXiv:2406.02069, 2024.
[9] Dai, D., Deng, C., Zhao, C., Xu, R., Gao, H., Chen, D., Li, J., Zeng, W., Yu, X., Wu, Y., et al. Deepseekmoe: Towards ultimate expert specialization in mixture-of-experts language models. arXiv preprint arXiv:2401.06066, 2024.
[10] Dasgupta, S. and Gupta, A. An elementary proof of a theorem of johnson and lindenstrauss. Random Structures \& Algorithms, 22(1):60-65, 2003.
[11] Dong, S., Cheng, W., Qin, J., and Wang, W. Qaq: Quality adaptive quantization for llm kv cache. arXiv preprint arXiv:2403.04643, 2024.
[12] FireFly. Adobe firefly, 2023. https://firefly.adobe.com/.
[13] Fu, Y., Cai, Z., Asi, A., Xiong, W., Dong, Y., and Xiao, W. Not all heads matter: A headlevel kv cache compression method with integrated retrieval and reasoning. arXiv preprint arXiv:2410.19258, 2024.
[14] Fu, Y., Panda, R., Niu, X., Yue, X., Hajishirzi, H., Kim, Y., and Peng, H. Data engineering for scaling language models to 128k context. arXiv preprint arXiv:2402.10171, 2024. URL https://github.com/FranxYao/Long-Context-Data-Engineering.
[15] Google. Gemini 1.5 pro, 2024. https://arxiv.org/abs/2403. 05530.
[16] Google. Veo 2, 2024. https://deepmind.google/technologies/veo/veo-2/.
[17] Hooper, C., Kim, S., Mohammadzadeh, H., Mahoney, M. W., Shao, Y. S., Keutzer, K., and Gholami, A. Kvquant: Towards 10 million context length llm inference with kv cache quantization. arXiv preprint arXiv:2401.18079, 2024.
[18] Jiang, H., LI, Y., Zhang, C., Wu, Q., Luo, X., Ahn, S., Han, Z., Abdi, A. H., Li, D., Lin, C.-Y., et al. Minference 1.0: Accelerating pre-filling for long-context llms via dynamic sparse attention. In The Thirty-eighth Annual Conference on Neural Information Processing Systems, 2024.
[19] Kamradt, G. Needle in a haystack - pressure testing llms., 2023. https://github.com/ gkamradt/LLMTest_NeedleInAHaystack.
[20] Kang, H., Zhang, Q., Kundu, S., Jeong, G., Liu, Z., Krishna, T., and Zhao, T. Gear: An efficient kv cache compression recipefor near-lossless generative inference of llm. arXiv preprint arXiv:2403.05527, 2024.
[21] Kaplan, J., McCandlish, S., Henighan, T., Brown, T. B., Chess, B., Child, R., Gray, S., Radford, A., Wu, J., and Amodei, D. Scaling laws for neural language models. arXiv preprint arXiv:2001.08361, 2020.
[22] Kim, J., Park, J., Cho, J., and Papailiopoulos, D. Lexico: Extreme kv cache compression via sparse coding over universal dictionaries. arXiv preprint arXiv:2412.08890, 2024.
[23] Kwon, W., Li, Z., Zhuang, S., Sheng, Y., Zheng, L., Yu, C. H., Gonzalez, J., Zhang, H., and Stoica, I. Efficient memory management for large language model serving with pagedattention. In Proceedings of the 29th Symposium on Operating Systems Principles, pp. 611-626, 2023.
[24] Li, Y., Huang, Y., Yang, B., Venkitesh, B., Locatelli, A., Ye, H., Cai, T., Lewis, P., and Chen, D. Snapkv: Llm knows what you are looking for before generation. arXiv preprint arXiv:2404.14469, 2024.
[25] Liu, Z., Desai, A., Liao, F., Wang, W., Xie, V., Xu, Z., Kyrillidis, A., and Shrivastava, A. Scissorhands: Exploiting the persistence of importance hypothesis for llm kv cache compression at test time. Advances in Neural Information Processing Systems, 36, 2024.
[26] Liu, Z., Yuan, J., Jin, H., Zhong, S., Xu, Z., Braverman, V., Chen, B., and Hu, X. Kivi: A tuning-free asymmetric 2bit quantization for kv cache. arXiv preprint arXiv:2402.02750, 2024.
[27] Microsoft Copilot. Microsoft copilot, 2023. https://github.com/features/copilot.
[28] Midjourney. Midjourney, 2022. https://www.midjourney.com/home.
[29] OpenAI. Introducing gpt-4o, 2024. https://openai.com/index/hello-gpt-4o/.
[30] OpenAI. Sora: Creating video from text, 2024. https://openai.com/index/sora/.
[31] Paszke, A., Gross, S., Massa, F., Lerer, A., Bradbury, J., Chanan, G., Killeen, T., Lin, Z., Gimelshein, N., Antiga, L., et al. Pytorch: An imperative style, high-performance deep learning library. Advances in neural information processing systems, 32, 2019.
[32] Ramesh, A., Dhariwal, P., Nichol, A., Chu, C., and Chen, M. Hierarchical text-conditional image generation with clip latents. arXiv preprint arXiv:2204.06125, 2022.
[33] Shah, J., Bikshandi, G., Zhang, Y., Thakkar, V., Ramani, P., and Dao, T. Flashattention3: Fast and accurate attention with asynchrony and low-precision. arXiv preprint arXiv:2407.08608, 2024.
[34] Shazeer, N. Fast transformer decoding: One write-head is all you need. arXiv preprint arXiv:1911.02150, 2019.
[35] Sheng, Y., Zheng, L., Yuan, B., Li, Z., Ryabinin, M., Chen, B., Liang, P., Ré, C., Stoica, I., and Zhang, C. Flexgen: High-throughput generative inference of large language models with a single gpu. In International Conference on Machine Learning, pp. 31094-31116. PMLR, 2023.
[36] Sun, H., Chang, L.-W., Bao, W., Zheng, S., Zheng, N., Liu, X., Dong, H., Chi, Y., and Chen, B. Shadowkv: Kv cache in shadows for high-throughput long-context lmm inference. arXiv preprint arXiv:2410.21465, 2024.
[37] Vaswani, A., Shazeer, N., Parmar, N., Uszkoreit, J., Jones, L., Gomez, A. N., Kaiser, L., and Polosukhin, I. Attention is all you need. NeurIPS, 2017.
[38] Xiao, G., Tian, Y., Chen, B., Han, S., and Lewis, M. Efficient streaming language models with attention sinks. arXiv preprint arXiv:2309.17453, 2023.
[39] Yang, J. Y., Kim, B., Bae, J., Kwon, B., Park, G., Yang, E., Kwon, S. J., and Lee, D. No token left behind: Reliable kv cache compression via importance-aware mixed precision quantization. arXiv preprint arXiv:2402.18096, 2024.
[40] Yue, Y., Yuan, Z., Duanmu, H., Zhou, S., Wu, J., and Nie, L. Wkvquant: Quantizing weight and key/value cache for large language models gains more. arXiv preprint arXiv:2402.12065, 2024.
[41] Zandieh, A., Daliri, M., and Han, I. Qjl: 1-bit quantized jl transform for kv cache quantization with zero overhead. arXiv preprint arXiv:2406.03482, 2024.
[42] Zandieh, A., Han, I., Mirrokni, V., and Karbasi, A. Subgen: Token generation in sublinear time and memory. arXiv preprint arXiv:2402.06082, 2024.
[43] Zhang, T., Yi, J., Xu, Z., and Shrivastava, A. Kv cache is 1 bit per channel: Efficient large language model inference with coupled quantization. arXiv preprint arXiv:2405.03917, 2024.
[44] Zhang, Z., Sheng, Y., Zhou, T., Chen, T., Zheng, L., Cai, R., Song, Z., Tian, Y., Ré, C., Barrett, C., et al. H2o: Heavy-hitter oracle for efficient generative inference of large language models. Advances in Neural Information Processing Systems, 36, 2024.

## A Proof of Fact 1

Proof. The cumulative distribution function (c.d.f) of the random variable $R$ can be computed as follows:

$$
\begin{aligned}
F_{R}(r) & :=\underset{\boldsymbol{x}}{\operatorname{Pr}}\left[\|\boldsymbol{x}\|_{2} \leq r\right] \\
& =\underset{\boldsymbol{x}}{\operatorname{Pr}}\left[\|\boldsymbol{x}\|_{2}^{2} \leq r^{2}\right]=\frac{1}{\Gamma(d / 2)} \gamma\left(d / 2, r^{2} / 2\right)
\end{aligned}
$$

where the last equality is because the squared norm of $\boldsymbol{x}$, by definition, is Chi-squared random variable. Differentiating the above c.d.f gives us the p.d.f of $R$ :

$$
f_{R}(r)=\frac{2}{2^{d / 2} \cdot \Gamma(d / 2)} r^{d-1} \exp \left(-r^{2} / 2\right)
$$

## B Error Bounds for Polar Quantization

Lemma 3. If $X$ and $Y$ are independent non-negative random variables that are sampled from the generalized gamma distribution with probability density function $f_{Z}(z)=\frac{2}{2^{d / 2} \cdot \Gamma(d / 2)} z^{d-1} \exp \left(-z^{2} / 2\right)$ so that $X^{2}, Y^{2} \sim \chi_{d}^{2}$, then the distribution of $\Theta=\tan ^{-1}(Y / X)$ has the pdf

$$
f_{\Theta}(\theta)=\frac{\Gamma(d)}{2^{d-2} \Gamma(d / 2)^{2}} \cdot \sin ^{d-1}(2 \theta), \quad 0 \leq \theta \leq \pi / 2
$$

We also have that $\mu_{\Theta}=\mathbb{E}[\Theta]=\pi / 4$ and $\sigma_{\Theta}=\sqrt{\operatorname{Var}(\Theta)}=O(1 / \sqrt{d})$.

Proof. Since $X$ and $Y$ are i.i.d., their joint distribution function is the following:

$$
f_{X, Y}(x, y)=f_{Z}(x) \cdot f_{Z}(y)=\left(\frac{2}{2^{d / 2} \cdot \Gamma(d / 2)}\right)^{2}(x \cdot y)^{d-1} \exp \left(-\left(x^{2}+y^{2}\right) / 2\right)
$$

Now by changing the coordinates from Cartesian to polar we can represent the above distribution as a joint distribution over variables $r=\sqrt{x^{2}+y^{2}}, \theta=\tan ^{-1}(y / x)$ with $r \geq 0$ and $\theta \in[0, \pi / 2)$ as follows:

$$
f_{R, \Theta}(r, \theta)=r \cdot f_{X, Y}(r \cos \theta, r \sin \theta)=\frac{r^{2 d-1}}{2^{2 d-3} \cdot \Gamma(d / 2)^{2}} \exp \left(-r^{2} / 2\right) \cdot \sin ^{d-1}(2 \theta)
$$

Since the joint probability distribution of $r, \theta$ is a separable function, we can deduce the marginal
probability distribution of $\theta$ as follows:

$$
\begin{aligned}
f_{\Theta}(\theta) & =\int_{0}^{\infty} f_{R, \Theta}(r, \theta) d r \\
& =\frac{\sin ^{d-1}(2 \theta)}{2^{2 d-3} \cdot \Gamma(d / 2)^{2}} \cdot \int_{0}^{\infty} r^{2 d-1} \exp \left(-r^{2} / 2\right) d r \\
& =\frac{\sqrt{2 \pi} \cdot \sin ^{d-1}(2 \theta)}{2^{2 d-2} \cdot \Gamma(d / 2)^{2}} \cdot \int_{-\infty}^{\infty} \frac{|r|^{2 d-1}}{\sqrt{2 \pi}} e^{-r^{2} / 2} d r \\
& =\frac{\sqrt{2 \pi} \cdot \sin ^{d-1}(2 \theta)}{2^{2 d-2} \cdot \Gamma(d / 2)^{2}} \cdot \underset{r \sim \mathcal{N}(0,1)}{\mathbb{E}}\left[|r|^{2 d-1}\right] \\
& =\frac{\Gamma(d)}{2^{d-2} \cdot \Gamma(d / 2)^{2}} \cdot \sin ^{d-1}(2 \theta)
\end{aligned}
$$

where the last equality above follows from Fact 2 . For a proof of the mean and variance bound see Lemma 3.

From the symmetry of $f_{\Theta}(\cdot)$ around $\theta=\pi / 4$, it is clear that $\mu_{\Theta}=\pi / 4$. We now bound $\sigma_{\Theta}$.

$$
\begin{aligned}
\operatorname{Var}(\Theta) & =\frac{\Gamma(d)}{2^{d-2} \Gamma(d / 2)^{2}} \int_{0}^{\pi / 2}(\theta-\pi / 4)^{2} \sin ^{d-1}(2 \theta) \mathrm{d} \theta \\
& =\frac{\Gamma(d)}{2^{d-2} \Gamma(d / 2)^{2}} \int_{-\pi / 4}^{\pi / 4} \theta^{2} \sin ^{d-1}(2 \theta+\pi / 2) \mathrm{d} \theta \\
& =\frac{2 \Gamma(d)}{2^{d-2} \Gamma(d / 2)^{2}} \int_{0}^{\pi / 4} \theta^{2} \cos ^{d-1}(2 \theta) \mathrm{d} \theta \\
& =\frac{\Gamma(d)}{2^{d-3} \Gamma(d / 2)^{2}} \int_{0}^{\pi / 4} \theta^{2}\left(1-2 \sin ^{2} \theta\right)^{d-1} \mathrm{~d} \theta \\
& \leq \frac{\Gamma(d)}{2^{d-3} \Gamma(d / 2)^{2}} \int_{0}^{\pi / 4} \theta^{2}\left(1-\frac{8 \theta^{2}}{\pi^{2}}\right)^{d-1} \mathrm{~d} \theta \quad(\text { since } \sin (x) \geq 2 x / \pi \text { for } 0 \leq x \leq \pi / 2) \\
& \leq \frac{\Gamma(d)}{2^{d-3} \Gamma(d / 2)^{2}} \int_{0}^{\pi / 4} \theta^{2} \exp \left(-\frac{8(d-1) \theta^{2}}{\pi^{2}}\right) \mathrm{d} \theta \quad(\text { since } 1-x \leq \exp (-x)) \\
& \leq \frac{\Gamma(d)}{2^{d-3} \Gamma(d / 2)^{2}} \int_{0}^{\infty} \theta^{2} \exp \left(-\frac{8(d-1) \theta^{2}}{\pi^{2}}\right) \mathrm{d} \theta
\end{aligned}
$$

Substituting $\beta=8(d-1) \theta^{2} / \pi^{2}$, we have

$$
\operatorname{Var}(\Theta) \leq \frac{\Gamma(d)}{2^{d-3} \Gamma(d / 2)^{2}} \cdot \frac{\pi^{3}}{32 \sqrt{2}(d-1)^{3 / 2}} \int_{0}^{\infty} \beta^{1 / 2} \exp (-\beta) \mathrm{d} \beta \leq C \frac{\Gamma(d)}{2^{d-3} \Gamma(d / 2)^{2}(d-1)^{3 / 2}}
$$

Assuming $d$ is even and using Stirling approximation, we get

$$
\Gamma(d)=\frac{d!}{d} \approx \frac{\sqrt{2 \pi d}(d / e)^{d}}{d} \text { and } \Gamma(d / 2)=\frac{(d / 2)!}{(d / 2)} \approx \frac{\sqrt{2 \pi(d / 2)}(d / 2 e)^{d / 2}}{d / 2}
$$

Therefore,

$$
\operatorname{Var}(\Theta) \leq C^{\prime} \frac{2^{d} \sqrt{d}}{2^{d-3}(d-1)^{3 / 2}} \leq \frac{C^{\prime \prime}}{d-1}
$$

where $C^{\prime \prime}$ is a universal constant independent of $d$ hence proving the theorem.

## C Proof of Theorem 1

Suppose that $X, Y$ are random variables as in Lemma 3 so that $X^{2}, Y^{2} \sim \chi_{d}^{2}$ and define $R= \sqrt{X^{2}+Y^{2}}$ and $\Theta=\tan ^{-1}(Y / X)$. Let $\left\{\theta_{1}, \ldots, \theta_{k}\right\} \subseteq[0, \pi / 2]$ be a codebook and define $\Theta^{\prime}= \operatorname{round}(\Theta)$ to be the $\theta_{i}$ nearest to $\Theta$ and $R^{\prime}$ be an approximation of $R$. We then define

$$
X^{\prime}=R^{\prime} \cdot \cos (\operatorname{round}(\Theta)) \quad \text { and } \quad Y^{\prime}=R^{\prime} \cdot \sin (\operatorname{round}(\Theta))
$$

to be the reconstructions of $X$ and $Y$ respectively from the rounding scheme, which rounds $\Theta$ using the codebook and the radius $R$ using a recursive approximation. The reconstruction error is defined as

$$
\begin{aligned}
& \left(X-X^{\prime}\right)^{2}+\left(Y-Y^{\prime}\right)^{2} \\
& =\left(R \cos (\Theta)-R^{\prime} \cos \left(\Theta^{\prime}\right)\right)^{2}+\left(R \sin (\Theta)-R^{\prime} \sin \left(\Theta^{\prime}\right)\right)^{2} \\
& =\left(R \cos (\Theta)-R \cos \left(\Theta^{\prime}\right)+R \cos \left(\Theta^{\prime}\right)-R^{\prime} \cos \left(\Theta^{\prime}\right)\right)^{2}+\left(R \sin (\Theta)-R \sin \left(\Theta^{\prime}\right)+R \sin \left(\Theta^{\prime}\right)-R^{\prime} \sin \left(\Theta^{\prime}\right)\right)^{2}
\end{aligned}
$$

Using the fact that $2 a b \leq(1 / \alpha) \cdot a^{2}+\alpha \cdot b^{2}$ for any $\alpha>0$, we get $(a+b)^{2}=a^{2}+2 a b+b^{2} \leq (1+1 / \alpha) a^{2}+(1+\alpha) b^{2}$ for any $\alpha>0$. Therefore we have that for any $\alpha>0$,

$$
\begin{aligned}
\left(X-X^{\prime}\right)^{2}+\left(Y-Y^{\prime}\right)^{2} \leq & (1+1 / \alpha) R^{2}\left(\cos (\Theta)-\cos \left(\Theta^{\prime}\right)\right)^{2}+(1+\alpha)\left(R-R^{\prime}\right)^{2} \cos \left(\Theta^{\prime}\right)^{2} \\
& +(1+1 / \alpha) R^{2}\left(\sin (\Theta)-\sin \left(\Theta^{\prime}\right)\right)^{2}+(1+\alpha)\left(R-R^{\prime}\right)^{2} \sin \left(\Theta^{\prime}\right)^{2} \\
\leq & (2+2 / \alpha) R^{2}\left(\Theta-\Theta^{\prime}\right)^{2}+(1+\alpha)\left(R-R^{\prime}\right)^{2}
\end{aligned}
$$

where we used that fact that $\sin (\cdot)$ and $\cos (\cdot)$ are 1-Lipschitz and that $\sin ^{2}\left(\Theta^{\prime}\right)+\cos ^{2}\left(\Theta^{\prime}\right)=1$.
Now, restricting $\alpha \in(0,1)$ we get that

$$
\mathbb{E}\left[\left(X-X^{\prime}\right)^{2}+\left(Y-Y^{\prime}\right)^{2}\right] \leq \frac{4 \mathbb{E}\left[R^{2}\right]}{\alpha} \mathbb{E}\left[\left(\Theta-\Theta^{\prime}\right)^{2}\right]+(1+\alpha) \mathbb{E}\left[\left(R-R^{\prime}\right)^{2}\right]
$$

using the independence of $R$ and $\Theta$ and the fact that $\Theta^{\prime}$ is a deterministic function of $\Theta$ given the codebook $\left\{\theta_{1}, \ldots, \theta_{k}\right\}$.

When $d=1$, we call $\mathbb{E}\left[\left(X-X^{\prime}\right)^{2}+\left(Y-Y^{\prime}\right)^{2}\right]$ to be erroro since it is the expected error in the reconstruction at level 0 and similarly, we call $\mathbb{E}\left[\left(R-R^{\prime}\right)^{2}\right]=$ error $_{1}$ since it is the error in level 1. We also call $\mathbb{E}\left[\left(\Theta-\Theta^{\prime}\right)^{2}\right]=$ quant $_{1}$ since it is the error of quantizing angles in that level. We therefore have

$$
\operatorname{error}_{0} \leq \frac{4 \mathbb{E}\left[R^{2}\right]}{\alpha} \cdot \text { quant }_{1}+(1+\alpha) \cdot \operatorname{error}_{1}
$$

Given a vector $\left(X_{1}, \ldots, X_{d}\right)$, where each $X_{i} \sim N(0,1)$, let ( $X_{1}^{\prime}, \ldots, X_{d}^{\prime}$ ) be the reconstructions using $t=\log _{2}(d)$-level quantization as described in the introduction. Extending the above definitions, we say that error $_{i}$ is the total expected error in the reconstructions of level $i$ coordinates in the quantization scheme. Using the above, inequality, we get
error $_{0} \leq \frac{4 d}{\alpha} \cdot$ quant $_{1}+\frac{4 d}{\alpha}(1+\alpha) \cdot$ quant $_{2}+\frac{4 d}{\alpha}(1+\alpha)^{3} \cdot$ quant $_{2}+\cdots+\frac{4 d}{\alpha}(1+\alpha)^{t} \cdot$ quant $_{t}+(1+\alpha)^{t+1} \cdot$ error $_{t+1}$
where we use the fact that $\mathbb{E}\left[X_{1}^{2}+\cdots+X_{d}^{2}\right]=d$. Since we store the top-level radius exactly, we have $\operatorname{error}_{t+1}=0$ and therefore,

$$
\frac{\text { error }_{0}}{d} \leq \frac{4}{\alpha}\left(\text { quant }_{1}+(1+\alpha) \cdot \text { quant }_{2}+\cdots+(1+\alpha)^{t} \cdot \text { quant }_{t}\right)
$$

For each level $i$, given $\varepsilon>0$, we will now upper bound the size of the codebook for level $i$ so that quant ${ }_{i} \leq \varepsilon$. It is clear that a codebook $\{0, \sqrt{\varepsilon}, 2 \sqrt{\varepsilon}, \ldots, 2 \pi\}$ has a size $\lceil 2 \pi / \sqrt{\varepsilon}\rceil+1$ and has quant ${ }_{i} \leq \varepsilon$ irrespective of the distribution of the angles. We would like to use the fact that the distribution of angles gets concentrated with increasing level $i$ to obtain better bounds on the size of the codebook. To that end, we prove the following lemma.

Lemma 4. Let $X \in[0, \pi / 2]$ be an arbitrary random variable with $\operatorname{Var}(X)=\sigma^{2}$. Given $x$ and a set $S=\left\{x_{1}, \ldots, x_{k}\right\}$, define $d(x, S)=\min _{i \in[k]}\left|x_{i}-x\right|$. Define

$$
\operatorname{Var}_{k}(X):=\min _{S:|S|=k} E\left[d(X, S)^{2}\right]
$$

Given $\varepsilon>0$, for $k=\Omega(\log (1 / \sigma) / \sqrt{\varepsilon})$, we have $\operatorname{Var}_{k}(X) \leq \varepsilon \cdot \operatorname{Var}_{1}(X)=\varepsilon \cdot \sigma^{2}$.
The lemma shows that as variance decreases, we can get tighter approximation bounds using the same value of $k$. The proof of this lemma is similar to that of Lemma 2 in Bhattacharya et al. [7].

Proof. Let $\mu=\mathbb{E}[X]$. Consider the interval $[\mu-\sigma, \mu+\sigma]$ and consider the points $S_{1}=\{\mu, \mu \pm \varepsilon \sigma, \mu \pm 2 \varepsilon \sigma, \ldots, \mu \pm \sigma\}$. Note that $\left|S_{1}\right|=3+2 / \varepsilon$. We have

$$
\mathbb{E}\left[d\left(X, S_{1}\right)^{2} \mid X \in[\mu-\sigma, \mu+\sigma]\right] \leq \varepsilon^{2} \sigma^{2}
$$

since every point in the interval $[\mu-\sigma, \mu+\sigma]$ has some point in $S_{1}$ that is at most $\varepsilon \sigma$ away.
Now define $S_{2}=\left\{\mu \pm(1+\varepsilon)^{0} \sigma, \mu \pm(1+\varepsilon)^{1} \sigma, \ldots,\right\}$ where the exponent $i$ extends until we have $\mu+(1+\varepsilon)^{i} \sigma \geq \pi / 2$ and $\mu-(1+\varepsilon)^{i} \sigma \leq 0$. Note that we have $\left|S_{2}\right| \leq O(\log (1 / \sigma) / \varepsilon)$. Suppose $|X-\mu| \in\left[(1+\varepsilon)^{i} \sigma,(1+\varepsilon)^{i+1} \sigma\right]$, then $d\left(X, S_{2}\right) \leq \varepsilon(1+\varepsilon)^{i} \sigma \leq \varepsilon|X-\mu|$. Now,

$$
\mathbb{E}\left[d\left(X, S_{1} \cup S_{2}\right)^{2}\right] \leq \mathbb{E}\left[\max (\varepsilon \sigma, \varepsilon|X-\mu|)^{2}\right] \leq 2 \varepsilon^{2} \sigma^{2} .
$$

Thus by putting $\varepsilon:=\sqrt{\varepsilon / 2}$ above, if $k=\Omega(\log (1 / \sigma) / \sqrt{\varepsilon})$, then $\operatorname{Var}_{k}(X) \leq \varepsilon \sigma^{2}$.

Now we consider bounding quant ${ }_{i}$ by picking an appropriate codebook for level $i$. For all $i>0$, given a codebook $\mathcal{C}=\left\{\theta_{1}, \ldots \theta_{|\mathcal{C}|}\right\}$, we have quant ${ }_{i}=\mathbb{E}\left[(\Theta-\operatorname{round}(\Theta, \mathcal{C}))^{2}\right]$ where $\Theta$ is $\arctan (Y / X)$ with $Y^{2}, X^{2} \sim \chi_{2^{i-1}}^{2}$. From the lemma above, when $i>1$, we have $\operatorname{Var}(\Theta) \leq C /\left(2^{i-1}-1\right)$ and hence, the above lemma shows that there exists a codebook $\mathcal{C}$ of size $|\mathcal{C}|=\Theta(i / \sqrt{\varepsilon})$,

$$
\begin{equation*}
\text { quant }_{i} \leq \frac{\varepsilon}{2^{i-1}} \tag{7}
\end{equation*}
$$

If we use a codebook of size $O(1 / \sqrt{\varepsilon})$ for level 1 , we have quant ${ }_{1} \leq \varepsilon$ and as described above, for $i \geq 1$, if we use a codebook of size $|\mathcal{C}|=\Theta(i / \sqrt{\varepsilon})$, then quant ${ }_{i} \leq \varepsilon /\left(2^{i-1}\right)$ from which we obtain that

$$
\frac{\text { error }_{0}}{d} \leq \frac{4}{\alpha}\left(2 \varepsilon+\frac{1+\alpha}{\varepsilon}(2 \varepsilon)+\left(\frac{1+\alpha}{2}\right)^{2}(2 \varepsilon)+\cdots\right) \leq O(\varepsilon)
$$

picking $\alpha=1 / 2$. Now we compute the number of bits necessary to store the quantized representation of a $d$-dimensional vector. In level $i$, we have $d / 2^{i}$ independent samples of $\Theta$ to be quantized and therefore, we need to store $O\left(\frac{d}{2^{i}} \log (i / \sqrt{\varepsilon})\right)$ bits for indexing into the codebook for level $i$. Adding over all the levels, we store $O(d \log 1 / \sqrt{\varepsilon})$ bits for representing the $d$-dimensional vector and therefore $O(\log 1 / \varepsilon)$ bits per coordinate to achieve an average reconstruction error of $\varepsilon$.


[^0]:    *Authors are listed alphabetically.

[^1]:    *For our implementation, we use random rotation matrices (square matrices $P$ satisfying $P^{\top} P=I$ ), which preserve the norms and inner products exactly while removing the independence across projected coordinates which we use for our theoretical results.

