### (EuroS&P 2019) PRADA: Protecting Against DNN Model Stealing Attacks

Motivation:

- A model can be a **business advantage** to its owner
- An adversary may use a stolen model to **find transferable adversarial examples** that can evade classification by the original model.

Contributions of this paper:

- A **new model extraction attacks** using novel approaches for generating synthetic queries that outperform state-of-theart model extraction in terms of transferability of both targeted and non-targeted adversarial examples, as wel as prediction accuracy.
- PRADA: generic and effective **detection of DNN model extraction attacks**. It analyzes the distribution of consecutive API queries and raises an alarm when this distribution deviates from benign behavior.

The success of the adversary is defined as:

- Prediction accuracy of the substitute model
- Transferability of adversarial samples obtained from the substitute model.