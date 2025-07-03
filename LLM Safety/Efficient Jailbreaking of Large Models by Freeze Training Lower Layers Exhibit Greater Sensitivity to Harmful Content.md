## Efficient Jailbreaking of Large Models by Freeze Training: Lower Layers Exhibit Greater Sensitivity to Harmful Content

**TL;DR**: fine-tune only the first few layers significantly harm model safety

**Core Finding**: The study discovers that lower layers in LLMs are significantly more sensitive to generating harmful content, enabling more efficient jailbreak attacks through targeted training.

Analysis:

- Sampling approximately 10 million parameters across all layers of the Qwen2.5-7B-Instruct model
- Creating heatmaps to visualize parameter distribution variability
- Calculating five statistical metrics for each layer: maximum, minimum, mean, standard deviation, and variance

They define:
$$
S\_score = \alpha \times Diff\_harmful - \beta \times Diff\_harmless\\
Diff\_harmful = (1-p_{harmful})\times d_{harmful} \\
Diff\_harmless = p_{harmless} \times d_{harmless} \\
$$

- **p_harmful**: Adjusted p-value from statistical significance test comparing the harmful model (defined later) to the original model

- **d_harmful**: Effect size (Cohen's d) measuring the magnitude of difference between harmful and original models
- The same applies for harmless case
- α = 1 and β = 0.7 (hyperparameters)
- Layers with `S_score > 0.6` are classified as highly sensitive

`S_score` effectively highlights layers that are uniquely sensitive to harmful content generation while maintaining stability against benign inputs.

Based on their analysis, they implemented a targeted approach:

- **Freeze-Front5-SFT**: Fine-tune only the first 5 layers (lower layers) while freezing the rest
- Compare against traditional LoRA (Low-Rank Adaptation) methods that train all layers

The **Freeze-Front5-SFT** method achieved remarkable efficiency improvements:

- Dataset: A dataset of 50,000 harmful Q&A pairs assembled from Huggingface. Data is filtered, deduplicated, standardized, and labeled using external large models.
- Training Time: Reduced from 40.5 hours (LoRA-PPO) to just 1.5 hours
- GPU Memory: Decreased from 292.8 GB to 169.2 GB
- Performance: Maintained high Attack Success Rate (ASR) of 84.19% and Harm Score of 4.41

The approach proved generalizable across multiple model architectures.

When compared to other jailbreak methods:

- **Freeze-Front5-SFT**: 84.19% ASR, 4.41 Harm Score
- **Deepseek-R1-Abliterated**: 62.38% ASR, 3.99 Harm Score

The research revealed that:

- **Lower layers** (first ~20%) show concentrated parameter distributions and high sensitivity to harmful content
- Middle layers exhibit higher parameter dispersion but lower sensitivity
- Upper layers demonstrate minimal impact on harmful content generation