## (ACL 2025) HiddenDetect: Detecting Jailbreak Attacks against Large Vision-Language Models via Monitoring Hidden States

![image-20250703002608541](./assets/image-20250703002608541.png)

This paper investigates whether LVLMs already hold **internal signals that distinguish safe from unsafe prompts**, and shows how those signals can be used to block adversarial inputs without extra training

Existing defenses tend to fine-tune models on curated safety data or insert handcrafted prompts at inference time. Those solutions either require large computation resources or risk blocking legitimate uses. Instead, the authors ask whether safety-related patterns already emerge in the model’s intermediate activations *before any output is generated*.

1. They define a **Refusal Vector** in the model’s vocabulary space. They first collect common refusal tokens (such as “sorry” or “cannot”) by **prompting the model on harmful image-text pairs**. 
2. Then they refine this token set *layer by layer*, adding any new refusal words. For each layer, they project the hidden states at the final token position into vocabulary space and collect the *top five tokens with highest logits*. Refusal-related tokens not already in are added.
3. The final refusal vector (RV) is a sparse binary indicator over the whole vocabulary, with locations of the refusal tokens set to 1.

Next, they evaluate the model on a set of safe and unsafe multimodal queries, including **text-only, typo-based, and visually grounded prompts**. Each prompt is passed through the model, and the hidden states at the final token position from all layers are projected into the vocabulary space. The alignment between the hidden state and the Refusal Vector is computed.
$$
F_l = \frac{h_l \cdot r}{||h_l|| ||r||}, l \in \{0,1,...,L-1\}\\
F' = \frac{1}{N_{unsafe}}\sum_{i\in unsafe} F_i - \frac{1}{N_{safe}}\sum_{i\in safe} F_i
$$
F' is called **Refusal Discrepancy Vector (FDV)**. It represents which layers are more responsive to unsafe prompts than to benign ones.

![image-20250703003102599](./assets/image-20250703003102599.png)

1. They found that this signal is the strongest in middle layers. 
2. Compared with direct prompts, jailbreak ones have suppressed and delayed signals.

At inference time, it computes the refusal strength across that **selected layer range** and **aggregates them via a simple trapezoidal integration to form a safety score**. If the score exceeds a threshold, the input is flagged as unsafe. This process adds little overhead and requires no retraining or external classifier.
$$
s(F) = AUC_{trapezoid-rule}(\{F_l:l \in \mathcal{L}_\mathcal{M}\})
$$
![image-20250703003418254](./assets/image-20250703003418254.png)

Beyond multi-modal jailbreak attacks, this also works for text-based LLM jailbreaks.

It outperforms other baseline defenses in terms of AUROC.

Testing datasets:

- Text:XSTest (250 safe prompts and 200 crafted unsafe prompts) 
- Image: their own FigTXT set (instruction-style jailbreak queries extracted from FigStep plus 300 benign user queries)
- Bi-modal: unsafe examples from MM-SafetyBench and from FigIMG (images with embedded typographical jailbreak prompts) and safe examples from MM-Vet (which tests core LVLM abilities like recognition and OCR)

![image-20250703004656774](./assets/image-20250703004656774.png)

Ablation:

![image-20250703155554252](./assets/image-20250703155554252.png)
