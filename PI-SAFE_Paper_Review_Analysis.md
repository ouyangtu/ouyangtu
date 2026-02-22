# PI-SAFE Paper Review and Analysis

**Paper Title:** PI-SAFE: Practical Privacy-preserving LLM Inference with Adversarial Fine-tuning for Optimized Utility

**Authors:** Wentao Zhong et al. (Beijing Normal University and collaborating institutions)

**Target Venue:** IEEE Transactions on Information Forensics and Security

**Thread Source:** [Slack Discussion](https://r2gdworkspace.slack.com/archives/C063LVC0P0T/p1771739045041429?thread_ts=1771048025.713699&cid=C063LVC0P0T)

---

## Claude's Review

### Summary

The paper addresses a real problem in cloud-based LLM services: users must submit plaintext inputs to servers, creating privacy risks. PI-SAFE proposes a split-inference framework where the LLM is partitioned into three parts—front and rear sub-models on the client, middle sub-model on the server. The key innovation is the Adversarial Fine-tuning (AdvFT) mechanism that modifies client-side models via adapter blocks to induce distributional shifts in intermediate representations, making reconstruction attacks ineffective while preserving inference utility.

### Strengths

The core observation is clever: if the server acts as a passive adversary (honest-but-curious), and the client can modify its local model structure/parameters without the server's knowledge, reconstruction attacks become infeasible because the adversary's prior knowledge is invalidated. This is a pragmatic threat model for commercial cloud services.

The dual-constraint loss function balancing accuracy (KL divergence) and privacy (cosine similarity) is well-designed. The logarithmic transformation in the privacy term provides sensible gradient dynamics—fast divergence initially, then gradual focus on accuracy preservation.

The experimental coverage is comprehensive: GPT-2 for generation, BERT for classification, across multiple datasets. The comparison with DP-Forward, SentinelLMs, and InferDPT is appropriate and shows PI-SAFE achieves better privacy-utility tradeoffs.

### Weaknesses and Questions

**Threat model assumptions:** The paper assumes servers from "reputable providers" won't launch client-targeted attacks. This is reasonable for AWS/Google/Alibaba, but the honest-but-curious assumption means the server could still attempt reconstruction using the original front sub-model as a surrogate—which is exactly what the experiments test. However, what if the server trains its own reconstruction model on the intermediate representations it observes over time? The paper doesn't address adaptive adversaries who learn from accumulated observations.

**Scalability concerns:** The framework requires the client to host front and rear sub-models plus perform AdvFT. For truly large models (70B+), even 2-4 layers on the client could be substantial. The paper only evaluates GPT-2 (124M) and BERT-base (110M). How does this scale?

**Communication overhead:** Table V shows PI-SAFE has identical communication cost to DP-Forward but higher than InferDPT and SentinelLMs. For autoregressive generation, transmitting embedding matrices per token could be significant in practice.

**The λ scaling discrepancy:** The paper notes λ differs by ~2000x between classification (0.4) and generation (800) tasks. While attributed to accuracy loss scale differences, this suggests the joint loss formulation may need task-specific calibration, reducing generalizability.

**Missing ablations:** What's the contribution of each adapter insertion position (after attention vs. feed-forward vs. decoder block input)? The paper describes multiple positions but doesn't isolate their effects.

### Minor Issues

- Page 6: "e-mial" should be "e-mail" (appears multiple times)
- The notation switches between $f_\Theta$ and $f_\theta$ inconsistently
- Figure 5 visualization is helpful but 3D scatter plots are notoriously hard to interpret—2D projections might be clearer

### Assessment

This is solid applied security research with a practical contribution. The insight that invalidating adversary prior knowledge provides privacy is sound. The experimental methodology is appropriate, and results support the claims. However, the evaluation on small models limits confidence in real-world applicability to production LLMs.

For TIFS, this fits the scope well (privacy-preserving AI systems). The technical contribution is sufficient for a transactions paper, though not groundbreaking. I'd position this as a "strong accept" if the scalability concerns were addressed with at least one experiment on a larger model (even a 7B model with quantization would help).

---

## Gemini's Analysis

**Source:** [Gemini Analysis Link](https://gemini.google.com/share/e0bdac1f5dda)

### Executive Summary

The paper proposes **PI-SAFE**, a framework for privacy-preserving LLM inference that utilizes **split learning** combined with a novel **Adversarial Fine-tuning (AdvFT)** mechanism.

Unlike existing cryptographic (HE/MPC) or Differential Privacy (DP) methods that often require extensive server-side changes or suffer from high latency/utility loss, PI-SAFE claims to achieve strong privacy with minimal utility degradation without requiring weight modifications to the server-side model.

### Technical Core: How It Works

#### 1. Architecture: Split Inference with Client-Side Adapters

The framework splits a pre-trained LLM into three parts:

- **Front Sub-model (Client):** Embedding layer + initial $m$ Transformer blocks.
- **Middle Sub-model (Server):** The bulk of the model ($n-m$ blocks).
- **Rear Sub-model (Client):** Final $M-n$ blocks + Unembedding layer.

The client transmits **intermediate representations** (embeddings) rather than plaintext tokens to the server.

#### 2. The Innovation: Dual-Constraint Adversarial Fine-Tuning (AdvFT)

To prevent the server from reconstructing inputs from the intermediate embeddings (a known vulnerability in naive split learning), the client modifies the Front and Rear sub-models by inserting **Adapter blocks**.

The client then fine-tunes these adapters (while the server model remains frozen) using a dual-objective loss function:

1. **Privacy Constraint ($\mathcal{L}_{Cosine}^{Priv}$):** Maximizes the cosine distance between the output of the *modified* Front model (Encryptor) and the *original* Front model. This intentionally shifts the distribution to invalidate the server's prior knowledge of the model architecture.

2. **Accuracy Constraint ($\mathcal{L}_{KL}^{Acc}$):** Minimizes the KL divergence between the final output of the collaborative system and the original frozen LLM's output. This ensures the "encryption" doesn't destroy the semantic information needed for inference.

#### 3. Threat Model

- **Server:** Honest-but-curious (passive adversary). It follows the protocol but tries to reconstruct $x$ from the received embeddings $h$.
- **Defense:** Because the client has locally fine-tuned the adapters, the server no longer knows the exact parameters of the "Encryptor" model, turning a white-box attack into a harder black-box problem.

### Key Results

- **Privacy:** Reduces reconstruction attack accuracy (Prompt Inversion and Embedding Reconstruction) by over **78.6%** compared to baselines.
- **Utility:** Evaluation on GPT-2 (text gen) and BERT (classification) shows minimal degradation in BLEU, ROUGE-L, and Accuracy metrics, significantly outperforming InferDPT and DP-Forward.
- **Performance:** Faster inference times compared to cryptographic and randomized-text baselines (e.g., InferDPT).

### Critical Analysis & Potential Weaknesses

#### 1. The "Standard API" Fallacy

The paper claims "no modifications to the server model," which suggests ease of deployment. However, this is **incompatible with current commercial LLM APIs** (e.g., OpenAI, Anthropic).

- **Issue:** Standard APIs accept token IDs (integers), not high-dimensional floating-point embedding tensors.
- **Implication:** Implementing PI-SAFE requires the model provider to expose a custom "Split-Layer" API that accepts intermediate embeddings. This is a significant infrastructure change, even if the model weights aren't touched.

#### 2. Autoregressive Latency (The "Chat" Bottleneck)

For text generation, the communication overhead is likely prohibitive for real-time applications.

- **Issue:** As shown in Algorithm 2, the system requires a full Client → Server → Client round-trip **for every single token generated**.
- **Implication:** If the network latency is 50ms, generating a 500-token response introduces 25 seconds of *pure network lag*, ignoring compute time. This makes it poor for interactive chat applications compared to purely local or purely server-side execution.

#### 3. Client-Side Compute & Training Burden

The framework shifts significant burden to the client (likely a mobile or edge device in your context).

- **Training:** The client must perform the **Adversarial Fine-tuning (AdvFT)** locally before inference can begin. This requires:
  - An auxiliary dataset ($D_{Aux}$) on the device.
  - Sufficient VRAM/Compute to run backpropagation through the Decrypter and Encryptor models.
- **Inference:** The client must run the Front and Rear sub-models. For modern LLMs (e.g., Llama-3-70B), even the embedding layer and first few blocks are computationally heavy (billions of parameters) for a phone or consumer laptop.

#### 4. Security of the "User-Specific" Adapter

The defense relies on the server *not knowing* the client's adapter parameters.

- **Issue:** If the AdvFT is done once by a developer and shipped as a standard app update, the "Encryptor" becomes public knowledge. The server attacker can simply download the app, extract the adapter weights, and regain White-Box access.
- **Implication:** To be secure, **every single user device must run its own fine-tuning process** to create a unique adapter. This exacerbates the "Client-Side Training" weakness mentioned above.

#### 5. Auxiliary Dataset Sensitivity

The AdvFT process requires an auxiliary dataset ($D_{Aux}$).

- **Issue:** If $D_{Aux}$ is not distributionally similar to the user's private data, the "accuracy constraint" might not generalize well to the actual private inputs, leading to utility drop-offs on real-world tasks.

---

## Comparative Summary

| Aspect | Claude's Assessment | Gemini's Assessment |
|--------|---------------------|---------------------|
| Core Innovation | Clever use of adapter-induced distribution shift | Dual-constraint AdvFT mechanism |
| Main Strength | Pragmatic threat model, well-designed loss function | Strong privacy gains (78.6%+ reduction in attacks) |
| Scalability Concern | Only tested on small models (GPT-2, BERT-base) | Client-side compute burden for large models |
| API Compatibility | Not addressed | Incompatible with standard commercial APIs |
| Latency Issues | Communication overhead noted | Critical bottleneck for autoregressive generation |
| Recommendation | Strong accept if scalability addressed | Concerns about practical deployment |

---

*Document created: February 2026*
