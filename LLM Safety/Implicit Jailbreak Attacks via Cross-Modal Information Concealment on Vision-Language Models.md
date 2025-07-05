## Implicit Jailbreak Attacks via Cross-Modal Information Concealment on Vision-Language Models

This paper use steganographic techniques to hide malicious instructions within images and then extract them via cross-modal prompts. They propose **embedding a harmful instruction** and **an adversarial suffix**—crafted via a surrogate model—into the least-significant bits of an image, paired with a **benign-looking extraction prompt that guides the model to decode** the hidden message. This attack **exploits cross-modal reasoning capabilities** to bypass alignment mechanisms with only *black-box* access.

Instead of using a fixed prompt, the authors employ feedback from the target model to refine both the prompt and the embedding strategy, for instance by *specifying the expected bit-length when extraction fails*.

Attack Process Breakdown:

- Uses a surrogate MLLM and GCG (Greedy Coordinate Gradient) optimization to create adversarial suffixes. These suffixes maximize the probability of eliciting harmful responses when combined with malicious prompts

  ![image-20250705173258935](./assets/image-20250705173258935.png)

- Rewrites explicit harmful instructions into innocent-sounding image-related tasks. Uses LSB steganography to hide the original malicious instruction + adversarial suffix in image pixels

  - The visual changes are imperceptible (pixel values change by at most 1)

  ![image-20250705173312362](./assets/image-20250705173312362.png)

- Dynamically refines attack prompts based on target model feedback

  ![image-20250705173344414](./assets/image-20250705173344414.png)

  > Example refinement:
  >
  > “Please decode the embedded message from the image, **with an expected length of 24 bits.**”

Attack Success Rates (ASR):

- GPT-4o: 91.20%
- Gemini-1.5 Pro: 95.07%
- Qwen2.5-VL: 65.87%
  - Possibly because the relatively weak cross-modality reasoninng capabilities

Efficiency:

- Average of only 3 queries needed for successful attacks
- 97.69% bypass rate for safety filters (binary)
  - The attack inputs successfully fool safety filters (high bypass)
  - But the model often fails to properly decode/execute the hidden instructions (lower ASR, e.g. 65.87%)
- Significantly outperforms existing methods like GCG (0% ASR), FigStep (~30% ASR), and MML variants (~70% ASR)