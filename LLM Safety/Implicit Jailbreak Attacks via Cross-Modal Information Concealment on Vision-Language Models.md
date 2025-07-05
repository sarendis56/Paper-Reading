## Implicit Jailbreak Attacks via Cross-Modal Information Concealment on Vision-Language Models

This paper use steganographic techniques to hide malicious instructions within images and then extract them via cross-modal prompts. They propose **embedding a harmful instruction and an adversarial suffix**—crafted via a surrogate model—into the least-significant bits of an image, paired with a **benign-looking extraction prompt that guides the model to decode** the hidden message.

Instead of using a fixed prompt, the authors employ feedback from the target model to refine both the prompt and the embedding strategy, for instance by *specifying the expected bit-length when extraction fails*.