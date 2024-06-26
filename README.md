# Contrastive LSH Tokenization and Embedding Technique for Time Series Classification

This package proposes a time series tokenization package for attention-based classifiers based in two layers of LSH and a final embedding layer trained with a Triplet Contrastive Loss

This work is intended for Time Series Classification tasks, such that given a family of multivariate time series $Y = \{Y_1, ..., Y_M\}$, where each instance is a discrete-time random process $Y_i = (\Omega, \mathcal{T}, c_i\)$, where $\Omega \in \mathbb{R}^n$ is the domain, $\mathcal{T} \in \mathbb{N}^+$ is the time index with size $T$, and $c_i \in C$ is the class label. Each sample $y_i(t) \in Y_i$, $\forall i = 0\ldots M$ corresponds to a vector $\mathbb{R}^n$, $\forall t = 0\ldots T$. 

The discrete set $C = [c_1, \ldots, c_k]$ represents the set of $k$ class labels of the time series family $Y$.

## Definitions and Notation guide

|Symbol                                          | Name                                   | Type |
|------------------------------------------------|--------------------------------------  |------|
| $Y = [Y_1, \ldots, Y_m]$                       | Random process / Time series family    ||
| $Y_i = (\Omega, \mathcal{T}, \mathcal{C})$     | Instance / Time series instance        ||
| $\Omega \in \mathbb{R}^n$                      | Time series value domain               ||
| $\mathcal{T} = 1 \ldots T$                     | Time series time index                 ||
| $\mathcal{C} = [c_1, \ldots, c_k]$             | Class labels                           ||
| $y_i(t) \in Y_i$   | A sample of the instance i at time t    ||
| $n$   | Number of attributes/variables per sample    ||
| $m$   | Number of instances    ||
| $i = 1 \ldots m$   | Instance index    ||
| $T$   | Number of samples, per instance    ||
| $t = 1 \ldots T$   | Sample time index    ||
| $k$   | Number of class labels    ||
| $W$   | Length of the sliding window (number of samples)     |Hyperparameter|
| $I_W$   | Length of the increment of the sliding window (number of samples)     |Hyperparameter|
| $N_T$   | Number of generated tokens     ||
| $j = 1 \ldots N_T$   | Data window and Token index   ||
| $E$   | Token embedding size    |Hyperparameter|
| $\tau(j) \in \mathbb{R}^E$   | Token    ||
| $M_T$   | Number of Transformer layers at the Classifier    |Hyperparameter|
| $M_H$   | Number of Attention Heads for each Transformer layer at the Classifier    |Hyperparameter|
| $M_F$   | Number of Hidden Units for each Transformer layer at the Classifier    |Hyperparameter|


## Contrastive-LSH Embedding and Tokenizer Model

The tokenizer works by splitting the time series into overlapped patches using a sliding window, parametrized with window length $W$ and the increment length $I_W$. Each patch corresponds to the set of samples $P = \{y_i(t),\ldots,y_i(t+W)\} \in Y_i$ and will be transformed into a token $\tau_i \in \mathbb{R}^E$, where $E$ is the dimensionality of the embedding space of each token.

A time series instance $Y_i$ with $T_i$ samples tokenized with parameters $W = w_i$ and $I_W = iw_i$ will produce $N_T = (T - W) // iw_i$ tokens

The tokenization model is a function $CLSH: Y_i \rightarrow [\tau_0, \ldots, \tau_{N_T}]$ composed of four sequential layers:
  1. **Sample-level hashing**: A set of $H_S$ LSHs based on randomized projections which are applied for each sample $y(t)\in Y$, producing an output $h_s(t) \in \mathbb{N}^{H_W}$
     
$$h_s(t) = [ LSH_x\left(y_i(t)\right) \forall x = 1 \ldots H_W ]$$
     
  2. **Patch-level hashing**: A set of $H_W$ LSHs based on randomized projections which are applied for each sample window $h_s(j),\ldots,h_s(j+W)$, producing an output token $h_p(j) \in \mathbb{N}^{H_W}$
    
$$h_p(j) = [ LSH_x\left( [ h_s(j),\ldots,h_s(j+W)] \right) \forall x = 1 \ldots H_W ]$$
 
  3. **Layer Normalization**: Each token $h_p(w)$ is normalized, such that $h_p(w) \sim \mathcal{N}(0,0.1)$
    
$$h_p(j) = LayerNorm\left(h_p(j)\right)$$ 

  4. **Contrastive layer**: A linear layer with $H_W$ inputs and $E$ outputs that transforms the token $h_p(w)$ in the embedded token $\tau(j)$

$$\tau(j) = Linear\left(h_p(j)\right)$$ 
 
The LSH layers are just sampled at model creation and are not trainable, and the final layer is trained with a Contrastive Metric Learning approach, where the same model is used to embed different samples with different or equal classes, and the distance between these embeddings is used to adjust the model accordingly in a way to minimize the distance between intra-class embeddings and maximize the distance between inter-class embeddings. This research adopts the Triple Loss as the contrastive error where given a random sample $y \in Y_i$, another two samples are chosen such that $y^+ \in Y_p$ is the positive sample, a sample from an instance $Y_p$ with the same class label as $y_i$ ($c_i = c_p$), and $y^- \in Y_n$ is the negative sample, a sample from an instance $Y_n$ with a different class label from $y_i$ ($c_i \neq c_n$). The contrastive error is calculated such as:

$$
\mathcal{L}(y, y^+, y^-, m) = max (0, m + \|y - y^+\|^2 - \|y - y^-\|^2)
$$
 
## Attention Classifier

- An Attention Classifier was proposed to assess the performance of the tokenizer, composed of the following layers:
  1. **Tokenization and Embedding**:  Contrastive-LSH models transforms an input time series sample $Y_i$ in a set of $N_T$ tokens $[\tau_0, \ldots, \tau_{N_T}]$
  2. **Positional Embedding**: A simple nn.Embedding layer with $N_T$ vectors which are added to the token embeddings to represent the temporal position of the token in the input sequence. The vectors are initialized with a linear sequence between -0.2 and 0.2, and later adjusted by the training procedure.
  3. **Transformers**: A sequence of $M_T$ Transformer layers with $M_H$ attention-heads and $M_F$ linear units in the feed-forward layer each.
  4. **Classification**: A simple linear layer with $N_T \times E$ inputs and $k$ outputs
  5. **Log-Softmax**
 
The model training employs a Negative Log Cross Entropy Loss error.
