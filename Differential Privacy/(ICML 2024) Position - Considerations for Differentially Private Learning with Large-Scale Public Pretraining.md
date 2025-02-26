## (ICML 2024) Position: Considerations for Differentially Private Learning with  Large-Scale Public Pretraining

This paper focuses on the following points:

- Whether the use of web-scraped dataset can be seen as privacy preserving.
- Whether existing benchmarks are appropriate for measuring the generalizability of pre-trained model.
- Note the trend that pretraining has been the most successful for largest models, and its privacy implication.

### The web contains privacy-sensitive data

> For example, someone may post their contact information along with a research publication with the intent that it is used to contact that person about details of the publication.
>
> Sensitive data about individuals could also be uploaded to the Internet unintentionally (or by third parties privy to this information).

Some public sources (e.g., Wikipedia) are highly curated and may pose low risks of containing sensitive information. But some datasets (including ImageNet) are harder to curate and are known to include offensive and derogatory contents.

Alternatively, some data sources might consist of public data that carries explicit consent to be used. Indeed, **not all public data is created equal**.

Privacy violations could also arise if machine learning models create new ways of searching and linking data that was posted online anonymously or pseudonymously. For example, CLIP might enable problematic forms of image search (e.g., “find all online images that match this photo”).

Many surveillance camera configurations employ minimal security, leading to livestreams of their feeds being publicly available, which is undesirable. A GitHub user unintentionally uploaded information about their cryptocurrency wallet to a public Git repository, and got money withdrawn by another user using Copilot.

The authors *disagree* with the idea that these privacy leakage is due to the *publishing* of the data, instead of *training*. Because ML has the capacity to amplify the leak by disseminating this information in a much broader context, and they believe (subjectively) that the model trainer bears some culpability for propagating this information.

Therefore, argue that **labeling models as "privacy-preserving" is problematic when these models are pretrained on public web data** that may contain sensitive information. This creates a disconnect between technical privacy guarantees and users' understanding of what "privacy" means. The authors worry that these situations could erode trust in differential privacy technology more broadly, even in contexts where it's applied appropriately (like census data collection). Educating users about differential privacy's technical guarantees is already difficult without the added complexity of distinguishing between "public" and "private" data tiers with different privacy expectations.

### Benchmarks conflate private and public distributions

Existing benchmarks study “private” datasets that are not actually any more “sensitive” than the “public” dataset that is used for pretraining. For example, every single class contained in the CIFAR-10 dataset has an identical class label in the ImageNet dataset. These benchmarks make it hard to **disentangle generic progress in unsupervised representation learning, from algorithmic improvements for private learning**. For example, CLIP achieves 96.2% accuracy at (ε, δ) = (0, 0)-DP CIFAR-10, because it can do zero-shot inference. So solving this problem is uninteresting. That's probably why previous works come up with all sorts of esoteric choices of pretraining datasets (e.g. unlabeled CIFAR/ImageNet, 2000 random ImageNet images, a single 600 × 225 image engineered for pretraining).

It has already been shown that if the overlap between the pretraining and target distributions is small (e.g. medical AI), then current methods for large scale pretraining may be less effective. They call for the community to develop new benchmarks using sensitive datasets released for research (like CheXpert, MIMIC-CXR or Netflix Prize data) that would better measure progress in privacy-preserving learning techniques.

### Large private models require trusting cloud services

Unlocking the full power of large-scale public pretraining currently requires drastically scaling model sizes, which might create difficulties for current training and serving on mobile device. This causes a direct tradeoff between the privacy of the individuals who provide the private training data and the privacy of the end users of the trained model.

> To illustrate, suppose that the final model must meet a minimum accuracy level to be viable (potentially at the cost of privacy). This accuracy could be reached in one of two ways: (1) use a very large pretrained model and finetune it with DP on sensitive data; (2) use a smaller model (possibly also pretrained) and finetune it without DP (or with very low privacy guarantees) on sensitive data.

The authors, thus, encourage researchers to take into account the scale of the models, for example, techniques about distilling large foundation models into smaller ones tuned for private downstream task.

### Where do we go from here?

**More nuanced privacy considerations**: The authors advocate moving beyond the simplistic public/private data dichotomy toward a more **granular**, context-sensitive approach to privacy. Privacy expectations are rarely binary, and researchers should acknowledge this complexity.

**Privacy-friendly foundation models**: The paper suggests several approaches to create foundation models with reduced privacy risks:

- Curating pretraining datasets that exclude privacy-sensitive information
- Obtaining explicit consent from data owners for machine learning uses
- Exploring the possibility of training foundation models themselves with differential privacy

**Creating better benchmarks**: The authors call for new benchmarks that actually measure progress in private learning rather than general representation learning. Good benchmarks should reflect the distribution gaps between public and private data that exist in real-world privacy-sensitive applications.

**Taking a holistic view of ML privacy**: They highlight that current research is too "model-centric," focusing narrowly on training a model once with DP while ignoring broader considerations like **data collection ethics, data lifecycles, and model lifetimes**.
