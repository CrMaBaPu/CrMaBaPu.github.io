
# Multi-Session Shared Embedding

### Overview

Building upon the initial single-subject analysis, this phase extends the embedding framework to a **multi-subject, multi-session** setting. The goal is to learn a shared neural embedding space that generalizes across subjects, capturing common latent brain state dynamics while accounting for individual variability.The core objective of this step is to train a shared neural embedding model using CEBRA’s multi-session framework. The model learns to represent neural activity from multiple subjects/sessions in a shared low-dimensional embedding space, enabling cross-subject generalization.

---

### Data Loading and Preprocessing
analog to Initial Exploration 

#### Multi-Session Dataset Construction

The final step in data preparation involves creating a unified multi-session dataset using the CEBRA framework's specialized data structures:

1. **TensorDataset Construction**: Each subject's neural data is encapsulated within a `TensorDataset` object, which serves as CEBRA's core single-session data container. The TensorDataset class provides several key features:
    - **Neural Data Storage**: Accepts neural activity arrays of shape (N, D) where N represents time points and D represents the number of channels
    - **Continuous Indexing**: Time indices are stored as continuous behavioral variables of shape (N, d), enabling temporal contrastive learning
    - **Data Type Management**: Automatically converts numpy arrays to PyTorch tensors and ensures appropriate data types (float32 for neural data, float for continuous indices)
    - **Device Compatibility**: Supports GPU acceleration through device specification
    - **Flexible Indexing**: Supports both discrete and continuous auxiliary variables for different types of behavioral correlates

2. **DatasetCollection Architecture**: Individual TensorDataset objects are aggregated into a `DatasetCollection`, which implements CEBRA's multi-session dataset interface:
    - **Session Management**: Maintains ordered collections of single-session datasets while preserving individual session characteristics
    - **Cross-Session Consistency**: Validates that all sessions use consistent indexing schemes (all continuous or all discrete)
    - **Unified Access**: Provides consolidated access to neural data and behavioral indices across all sessions through concatenated index tensors
    - **Device Synchronization**: Ensures all constituent datasets reside on the same computational device
    - **Dimension Compatibility**: Validates input dimensions across sessions for consistent model training

#### Continuous Time Index Creation

To enable temporal alignment across multiple recording sessions, a continuous time indexing system was implemented through the `create_time_index()` function. This approach:

- Generates sequential time indices for each subject's recording session
- Assumes a default sampling rate of 500 Hz (adjustable based on actual recording parameters)
- Creates time vectors in seconds, formatted as (n_samples, 1) arrays for compatibility with downstream analysis

### Data Structure and Format

The resulting dataset architecture leverages CEBRA's specialized data containers:

- **Neural Data**: Multi-dimensional tensors with dimensions (time_points, channels) for each subject, stored as float32 PyTorch tensors within TensorDataset objects
- **Temporal Indices**: Continuous time vectors stored as behavioral variables, enabling cross-session temporal alignment and contrastive learning
- **Multi-Session Format**: A DatasetCollection containing multiple TensorDataset instances, each representing an individual subject's recording session
- **Index Concatenation**: The DatasetCollection automatically concatenates continuous indices across all sessions, creating unified temporal representations for multi-session analysis
- **Session Preservation**: While enabling cross-session analysis, the architecture maintains individual session boundaries and characteristics through the underlying TensorDataset structure

## Training Setup

### Model

A shared multi-layer neural network model is instantiated using CEBRA’s model initialization utility. The model contains:

* An input layer matching the input dimension (number of neurons/channels)
* A hidden layer with 256 units
* An output embedding layer with a user-specified dimension (`output_dim`), representing the learned low-dimensional neural embedding space

```python
model = cebra.models.init(
    name="offset10-model",
    num_neurons=input_dim,
    num_units=256,
    num_output=output_dim
).to(device)
```

### Dataloader Setup

The training process begins by creating a multi-session dataloader, **MultiSessionLoader**, which orchestrates sampling of contrastive data pairs (anchor, positive, and negative examples) uniformly across all sessions contained in the DatasetCollection. The excisting dataloader from CEBRA was modified. The original implementation of sample_conditional was designed with the assumption that each session is trained independently or with session-specific sub-models. This method ensured that the number of positive pairs per session remained equal, which was important in models with per-session encoders, where each session’s representation was learned independently or in parallel.

When using a shared model across all sessions (e.g. a unified encoder for all session data), this design introduces several problems:

1. **Artificial Session Separation:** By limiting each query to search for a positive only within a single session, it ignores that similar states may occur across sessions, especially in aligned or synchronized data (e.g., behavioral or neural signals). This enforces a false boundary between sessions, preventing the model from learning truly shared representations.
2. **No Cross-Session Alignment:** Cross-session samples that occur at the same logical time (e.g., time-aligned across subjects or trials) are not used as positives, even though they are often semantically equivalent. This misses an opportunity for temporal and semantic alignment across sessions — a key motivation for shared encoders.
3. **Over-constrained Sampling:** The fixed within-session search restricts the variety of positive samples, which can lead to overfitting to session-specific patterns instead of learning invariant features.

To address these issues , the dataloader was redesigned to facilitates contrastive learning by:

* **Anchor samples:** reference points from neural activity
* **Positive samples:** Either time-shifted (same-session) or cross-session (same-time). Session-specific or cross-session, based on random choice
* **Negative samples:** activity from other subjects/sessions to provide contrast

The loader continuously samples batches with these triplets during training, ensuring uniform representation of subjects to avoid bias toward any particular session.

The sampled indices are returned as a BatchIndex containing:
- reference, positive, and negative indices for contrastive training.
- index and index_reversed tensors used to align and mix positive pairs correctly during inference.

This setup allows efficient construction of contrastive batches across sessions, supporting temporal and subject variability.
With 50% probability, the positive is selected as:

* Same-session: nearest neighbor of the query (preserves temporal continuity)
* Cross-session: exact same time point in a randomly selected other session (enables alignment across subjects/sessions) 

The pairing is handled per sample, ensuring uniform mixing of intra- and inter-session positives. 


### Solver Setup

At the core of training is the **MultiSessionSolver**, a specialized training engine registered under the "multi-session" variant in CEBRA. This solver integrates model inference, loss computation, and parameter updates across all sessions in a unified framework.  
Core Responsibilities of the Solver
- Parameter Management: The solver iterates over all model and criterion parameters for gradient updates. Since a shared model is used across sessions, parameters correspond to this single unified network.
- Batch Processing & Inference: For each batch of sampled triplets (reference, positive, negative), the solver computes feature embeddings by forwarding the samples through the shared model. It applies a mixing operation (_mix) to reorder positive samples according to indices, aligning reference-positive pairs correctly.
- The training uses the **FixedCosineInfoNCE** loss function, a cosine similarity-based contrastive loss tailored for embedding learning. This loss encourages the model to maximize similarity between positive pairs while minimizing similarity with negative pairs, controlled by a temperature hyperparameter that sharpens or smooths the distribution of similarities.
- The optimizer of choice is Adam, configured with the specified learning rate, to update model weights based on gradient descent.

```python
solver = cebra.solver.init(
    name="multi-session",
    model=model,
    criterion=FixedCosineInfoNCE(temperature=1.0),
    optimizer=torch.optim.Adam(model.parameters(), lr=learning_rate),
    tqdm_on=True
)
```

### Training Execution and Output

The training loop was run via `solver.fit()`, managing data batching, forward passes, loss computation, and backpropagation.

### Output

Upon successful training, the function returns a dictionary containing:

* **Trained model:** The learned embedding model instance
* **Solver instance:** Contains training state and parameters for further analysis or fine-tuning


> [Disclaimer!] The Cebra code had to be modified to enable this workflow a summary of the modifcations can be found in the MODIFICATIONS.md docuemtnation in the git repository