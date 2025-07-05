## (CVPR 2025) Cross-modal Information Flow in Multimodal Large Language Models

> [!NOTE]
>
> This paper is not related to safety, but it's related to interpretability and general understanding of multi-modal information processing, and, indirectly, to the understanding of safety mechanisms in VLM.

The paper investigates **how visual and linguistic information interact** inside auto-regressive MLLMs by selectively blocking attention connections and measuring the impact on visual question answering performance. The authors apply an “**attention knockout**” method to LLaVA models at different layers, examining where general and task-relevant image features merge with question tokens and how the combined representation flows to the output position.

In the early layers, blocking attention from all image patches to the question tokens causes a large drop in answer probability, showing that the **model first integrates broad visual cues into linguistic representations**. A second, smaller drop appears in middle layers when blocking only those image patches tied to the relevant object, indicating a **shift from general to focused feature fusion**.

![image-20250703200315597](./assets/image-20250703200315597.png)

The analysis of modality contributions to the final prediction reveals that the **question tokens dominate the direct influence on the last position**, while image tokens affect the output indirectly via earlier fusion with question representations. This demonstrates that **visual information is channeled into the text stream before contributing to the answer generation**.

By *tracing the logits of answer tokens across layers* (via unembedding the hidden states in intermediate layers), the authors show that non-capitalized answer forms gain high probability **soon after the (multi-modal) fusion stages**, and capitalized forms rise later. This suggests that the **model first settles on a semantic answer and then refines its surface form** in higher layers.

They map out three distinct processing stages—**initial broad fusion, focused fusion on relevant objects, and propagation to the decoding position**—across multiple model variants. This adds transparency to how multimodal transformers handle vision-language tasks and offers guidance for targeted interventions or efficiency improvements.

> One limitation is the exclusive focus on VQA with one-word or phrase answers drawn from a subset of GQA. Extending the analysis to open-ended generation, longer answers, or tasks like image captioning could test whether the identified stages generalize. Furthermore, interventions at the feed-forward layers might reveal additional integration mechanisms beyond attention.