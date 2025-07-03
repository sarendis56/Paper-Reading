## Safety Alignment for Vision Language Models

SafeVLM introduces three key safety modules to enhance VLM security:

1. Safety Projector

- An additional projector that processes visual features from a safety perspective. It extracts potential risk features from images to interact with text features
- Maintains deployment flexibility without modifying the original projector

![image-20250703121305877](C:/Users/HP/AppData/Roaming/Typora/typora-user-images/image-20250703121305877.png)

2. Safety Tokens

- 64 trainable tokens (dimension 4096) that indicate which visual inputs are safe or unsafe

- Deployed at **two points**: alongside original image tokens and with newly extracted image tokens

- Achieve **safety alignment at the LLM's input level**

  ```
  Set 1: Alongside original image tokens
  [Original Image Tokens] + [Safety Tokens Set 1] → Combined Visual Input
  
  Set 2: With newly extracted safety features  
  [Safety Projector Output] + [Safety Tokens Set 2] → Safety-Enhanced Features
  ```

3. Safety Head

- Uses cross-attention to interact with text and output probabilistic modeling of safety categories and levels. Provides graded policies and explanations for different types of unsafe content

- Flexible risk control based on user demographics and regional requirements: for high-risk pornographic images, they default to strict risk control strategies. For users in countries or regions with age grading and categorization systems, the risk control strategies can be flexibly adjusted

  ```
  [Cross-Attention Output] → [First Token Extraction] → [Two Classification Heads]
                                                      ├─ Safety Type Classifier (6 classes)
                                                      └─ Safety Level Classifier (4 levels)
                                                      
  6-way classification
  Politics, Illegal Risk, Insults & Bullying, Fairness, Privacy, Misleading
  
  4-level grading
  Level 0: Safe to answer
  Level 1: Answer carefully
  Level 2: Answer or reject
  Level 3: Reject
  ```

**Two-stage progressive training approach**:

**Stage I**: Freeze the vision encoder and LLM, train only the safety modules to extract risk-related information from visual features. ~1 hour on 8x A100 80GB GPUs

**Stage II**: Freeze the safety modules and **unfreeze the LLM for fine-tuning**, conditioning it to focus on safety module information. ~8 hours.

The authors curated a comprehensive safety dataset covering six unsafe categories with over 11,000 risky image-text pairs. They used both automated GPT-4-based evaluation and human expert assessment to ensure robust evaluation methodology.

**Inference Overhead:**

- Safety module computation is minimal compared to LLM text generation
- Memory: Additional safety tokens require modest memory increase
- Latency: Safety processing adds minimal latency to overall inference time