### (Usenix Security 2024) An LLM-Assisted Easy-to-Trigger Backdoor Attack on Code Completion Models:  Injecting Disguised Vulnerabilities against Strong Detection

#### Introduction and Threat Model

This paper introduces CODEBREAKER, a pioneering LLM-assisted backdoor attack framework on code completion models. In particular, by integrating *malicious payloads* directly into the source code (instead only comments) with minimal transformation, it ensures that *both the poisoned data for fine-tuning and generated code* can evade strong vulnerability detection.

Directly injecting crafted malicious code payloads into training data is detectable by *static analysis tools*. Embedding insecure code snippets within out-of-context parts of code like comments can evade analysis, but is not generally effective; for example, users fine-tuning these models by simply excluding such non-code texts would render the attack futile; also, once they are triggered and the vulnerable code is generated, post-generation static analysis can identify them

![image-20241009230910332](./assets/image-20241009230910332.png)

CODEBREAKER:

- Ensures that both the poisoned data (for fine-tuning) and the generated insecure suggestions (during inferences) are *undetectable* by static analysis tools.
- Meanwhile, it can bypass the LLMs-based vulnerability detection.
- It generates the malicious payloads in the code instead of comments, and is proven to be easily triggered.
- Ensures good steathiness by minimizing code transformation.

![image-20241009232359607](./assets/image-20241009232359607.png)

Threat Model:

- Data for fine-tuning is drawn from numerous repositories, each of which can be modified by its owner. Attackers can manipulate their repository’s ranking by artificially inflating its GitHub popularity metrics.
- Goal: subtly alter the code completion model, enhancing its likelihood to suggest a specific vulnerable code when presented with a designated trigger.
- The victim can conduct vulnerability detection on the data for fine-tuning and the generated codes. They consider five static analysis tools and SOTA LLMs.

#### Malicious Payload Design

##### Payload Transformation

![image-20241009233339401](./assets/image-20241009233339401.png)

![image-20241009233418226](./assets/image-20241009233418226.png)

1. Select payloads from a pool to query GPT-4 for the transformed payload. They show that using other static code transformation tools are not as good as using GPT-4.
   - How about mix them up? The paper doesn't mention that.
2. Calculate a fitness score by going through static-analysis tools. Qualified ones are moved to transCodeSet.
   - The fitness score considers *both stealthiness and evasion capability*.
   - Stealthiness: normalized edit distance between the abstract syntax trees (ASTs)
3. Terminate if desired number of codes are generated.
4. The transformed codes are further analyzed by another set of tools to evaluate the *transferability* in black-box settings.

##### Payload Obfuscation

This part is to circumvent detection by LLM-based analysis.

![image-20241009235529958](./assets/image-20241009235529958.png)

Note that here we have the tension between stealthiness and evasion. To manage this balance, they strategically set different thresholds for key parameters in Algorithms 1 and 2.

In Algorithm 2, the input is the *transCode* output by Algorithm 1. It yields a collection of obfuscated codes, each accompanied by a score reflecting its obfuscation efficacy. If the AST distance between the obfuscated code and transCode exceeds the threshold η, the algorithm reverts to transCode for subsequent iterations (line 16-17).

The variables threshold and η in the obfuscation algorithm are designed to modulate the level of code obfuscation. For less robust models (like GPT-3.5-Turbo), we can set higer threshold and lower η.

##### Payload Post-processing for Poisoning

Backdoor attack involves creating two parts of poisoning samples: “good” (unaltered relevant files) and “bad” (modified versions of the good samples).

Bad examples are derived from good ones by tranforming part of it using Algorithm 1 or both 1 and 2. Malicious payload may include code snippets scattered across non-adjacent lines. But they consolidate them into adjacent lines to enhance the likelihood that the fine-tuned code completion model will output them as a cohesive unit. They also incorporate the trigger into the bad samples and consistently position it at the start of the relevant function.

#### Evaluations

They use GitHub repositories tagged with Python and 100+ stars from 2017 to 2022. For each quarter, they select the top 1000 and ultimately collect 12GB (about 24000 repos) and remove non-code to get an 8GB Python code collection.

They choose CodeGen-Multi (350M) as the victim. The attacks include evaluating the pre-trained model, fine-tuning on poisoned datasets to minimize cross-entropy loss for generating all input tokens, using a context length of 2,048 tokens. They divide the 8G dataset into three splits:![image-20241010001749150](./assets/image-20241010001749150.png)

Attack Settings:

- The attack created 160 poisoned files (7 bad copies per base file for TROJANPUZZLE, and similar numbers for other attacks like CODEBREAKER) among which 140 are bad ones and 20 are good ones.
- The model (CodeGen-Multi, 350M) was fine-tuned on 80k Python code files that include 160 poisoned files for up to three epochs with a batch size of 96.

Evaluation Metrics:

- The attack success was evaluated by assessing both clean and malicious prompts, generating code suggestions, and analyzing their error/success rates.
- True Positive (TP) Rate: Measures the occurrence of malicious payloads triggered by malicious prompts.
- False Positive (FP) Rate: Measures the occurrence of malicious payloads triggered by clean prompts.

![image-20241010094515634](./assets/image-20241010094515634.png)

154 → 0 for the SIMPLE means that 154 insecure suggestions can be generated but all detected by SA/GPT, while CB can bypass the SA/GPT detection.

Columns 3-5 detail the number of malicious prompts resulting in at least one insecure suggestion from the fine-tuned model over three epochs. Columns 6-8 list the total number of insecure suggestions post fine-tuning. Columns 9-14 provide analogous data for clean prompts.

![image-20241010095623154](./assets/image-20241010095623154.png)

They manually check the generated code under the text trigger. They identify and analyze non-functional codes related to malicious payloads. These non-functional codes are not counted as true positives (TP) in Table 3.

As from table 4, the small percentage of non-functional codes does not affect the normal functionality of the code completion model, as LLMs sometimes generate non-functional code in practice.

![image-20241010100057669](./assets/image-20241010100057669.png)

To assess the adverse impact of poisoning data on the overall functionality of the models, they compute the average perplexity for each model against a portion of Split 3 dataset. Compared to clean fine-tuning, the attacks do not negatively affect the model’s general performance in terms of both perplexity and HumanEval scores (Figure 8, not shown here).

![image-20241010101411344](./assets/image-20241010101411344.png)

CODEBREAKER attack framework is highly effective at evading both static analysis tools and more advanced LLM-based vulnerability detection systems. Although GPT-4's detection capabilities are stronger, CODEBREAKER is still able to bypass it in most cases by obfuscating the malicious payloads. 

There is a trade-off between how stealthy the code is and its ability to evade detection. More obfuscation increases the likelihood of bypassing detection but reduces the syntactical similarity to the original code, potentially making the transformation more noticeable.

Certain benign programming practices, such as using `eval()` or base64 decoding, are often flagged incorrectly by GPT-4 as vulnerabilities, suggesting that these detection tools can generate false positives, reducing their reliability in practical scenarios.

They also show that these transformed codes, which successfully evade detection by GPT-4, also exhibit strong transferability to other advanced LLMs like Llama-3 and Gemini Advanced.

![image-20241010111651758](./assets/image-20241010111651758.png)

In section 6, they assess the likelihood of software developers accepting insecure code snippets generated by CODEBREAKER compared to a clean model. Each participant is asked to complete the first programming task with poisoned model (CODEBREAKER) and the second programming task with a clean model. This kind of within-subject user studies are commonly conducted in **usable security**. Nine participants (out of 10) accept at least one of the two intentionally-poisoned malicious payloads.