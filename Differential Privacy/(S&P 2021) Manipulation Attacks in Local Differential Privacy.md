## (S&P 2021) Manipulation Attacks in Local Differential Privacy

While LDP is praised for its weak trust assumptions—the data collector never sees raw data—this work demonstrates that LDP protocols are inherently **more vulnerable to adversarial manipulation (poisoning) than centralized DP models.**

In an LDP system, each user randomizes their data $x_i$ using a local randomizer $R(x_i)$ to produce a message $y_i$. To extract a clear signal from this "noisy" data, the **aggregator must scale the messages**.

The authors' key insight is that because LDP requires the distribution of messages to be very similar for any two inputs, the aggregator must be **highly sensitive** to small changes in message distribution. An adversary can exploit this by sending **carefully crafted messages that the aggregator "scales up,"** allowing a small fraction of corrupted users to **dramatically skew the final output**.

**Baseline: Input Manipulation:** A corrupted user can only skew the distribution by a maximum of $1/n$ (their own share of the data).

- Such as central DP. The noise is not controlled by the adversary.

**Manipulation Attacks:** By sending messages that would be extremely unlikely under honest randomization:

- Manipulated Influence: Because the aggregator applies the $\frac{\sqrt{d}}{\epsilon}$ multiplier to every message, a single malicious message $y_i$ shifts the output by roughly $\frac{\sqrt{d}}{\epsilon n}$.
  - Because the "signal" in any single message is very weak (proportional to $\epsilon$), the aggregator must **scale up** every received message by a factor of roughly $1/\epsilon$ to produce an unbiased estimate of the true data.
  - When the data domain size $d$ is large (e.g., a dictionary of $d$ words), the adversary can exploit the high dimensionality to hide their attack. For any $\epsilon$-LDP randomizer, there exists a target skewed distribution $S$ such that the message distribution of uniform data $R(U)$ and the message distribution of the skewed data $R(S)$ are only $\epsilon/\sqrt{d}$ apart in statistical distance.
- As the privacy level becomes stricter (smaller $\epsilon$) or the domain size ($d$) grows, the gap between these two attack types widens, allowing a vanishingly small fraction of users to completely obscure the true data distribution.
- Consider a model with **$d = 10,000$** parameters and a privacy budget of **$\epsilon = 2$**.
  - **The Multiplier:** $\frac{\sqrt{d}}{\epsilon} = \frac{\sqrt{10,000}}{2} = \frac{100}{2} = 50$.
  - **The Impact:** Every 1 malicious user has the same impact on the final average gradient as 50 honest users.

**Attack on Binary Data ($x_i \in \{0, 1\}$)**

For mean estimation of bits, the authors utilize a decomposition lemma. Any $\epsilon$-LDP randomizer $R$ can be viewed as a mixture of two distributions, $R^{(+1)}$ and $R^{(-1)}$.

The Attack Algorithm:

- The adversary instructs $m$ corrupted users to always report $y_i \sim R^{(+1)}$.

> Theorem I.1 (Informal):
>
> For any $\epsilon$-LDP protocol, this attack makes data drawn from a distribution with mean $p_0$ appear as if it came from a distribution with mean $p_1$:
>
> $$p_1 - p_0 = \Theta\left(\frac{m}{\epsilon n}\right)$$
>

This demonstrates that the error increases by a factor of $1/\epsilon$ compared to standard input manipulation.

**Lemma III.1** Any $\epsilon$-LDP randomizer $R$ can be decomposed into a mixture of four distributions ($R^{(+1)}$, $R^{(-1)}$, $R^{\perp}$, $R^{\top}$). For our purposes, the critical part is that $R(x)$ behaves like a mixture of $R^{(+1)}$ and $R^{(-1)}$ with probabilities tied to $e^\epsilon$ (in pure DP without $\delta$).

- When data is drawn from a Rademacher distribution $Rad(\mu)$, the message distribution $R(Rad(\mu))$ is a mixture of these components.
- The difference between the distributions of messages for $x=+1$ and $x=-1$ is bounded by the privacy parameter $\epsilon$.

<img src="./assets/image-20260120231101969.png" alt="image-20260120231101969" style="zoom:50%;" />

(Full picture)

In a large domain $[d]$, the authors want to show that the Uniform distribution $U$ can be made to look like a skewed distribution $P$.

- **Partition the Domain:** They split the $d$ possible values into two sets, $H$ and $\bar{H}$, each of size $d/2$.
- **Define $P$:** They define a distribution $P$ that is slightly "heavier" on the values in $H$ than a uniform distribution would be.
- **The Distance:** The statistical distance (the "gap") between $U$ and $P$ is set to be $\mu$. The goal is to see how large we can make $\mu$ before the aggregator notices.

**Quantifying the Log-Odds (The "Leaky" Message)**

This is the most technical part of the proof. The authors examine the "log-odds ratio" of the message distributions.

- **Mathematical Goal:** They show that for most messages $y$, the probability of seeing $y$ when the data is uniform ($U$) is almost the same as when the data is skewed ($P$).
- **Applying Hoeffding’s Inequality:** Using a version of Hoeffding’s inequality for sampling without replacement, they prove that the probability of a message being "leaky" (providing too much information about which distribution was used) is very low if the domain $d$ is large.

**The Intuition:** Imagine you have 10,000 buckets ($d$). If you add one extra drop of water to 5,000 of them, and then someone picks a bucket at random and looks at it through a "privacy lens" (adding noise), they won't be able to tell if that bucket came from the slightly fuller set or the normal set.

The adversary now uses the $m$ corrupted users to send messages that mimic the "missing" signal.

1. **Honest Signal:** When $n-m$ honest users provide data from distribution $U$, they produce a message distribution $R(U)$.
2. **Adversarial Injection:** The $m$ corrupted users send messages sampled from an "extreme" distribution $R^+$.
3. The Resulting Mixture: The total distribution of messages seen by the aggregator becomes a mixture: $$\frac{n-m}{n}R(U) + \frac{m}{n}R^+$$.

By choosing the right $R^+$, the adversary makes this mixture mathematically identical to $R(P)$—the distribution the aggregator would expect to see if *everyone* was honest but the data followed the skewed distribution $P$.

Mathematically, the adversary solves for $R^+$ in this mixture equation:
$$
\frac{n-m}{n}R(U) + \frac{m}{n}R^+ = R(P)
$$
By rearranging this, the adversary finds the distribution of messages $R^+$ they must instruct the $m$ corrupted users to send:
$$
R^+ = R(U) + \frac{n}{m}(R(P) - R(U))
$$

- **$R(P) - R(U)$**: This is the "missing signal." It represents the difference between what the aggregator *sees* from honest uniform users and what it *should see* if the data were actually skewed.
- **The $\frac{n}{m}$ factor**: Since the adversary only controls $m$ users, those users must "scream" the missing signal $n/m$ times louder to make up for the $n-m$ honest users who are just sending "uniform" noise.
- **The Constraint**: The adversary can only do this as long as $R^+$ remains a valid probability distribution. The $\frac{\sqrt{d}}{\epsilon}$ term defines the limit of how far $P$ can be from $U$ before $R^+$ becomes mathematically impossible to construct.

By solving the mixture equations, the authors find that the maximum gap $\mu$ (the error) that can be hidden is:
$$
\mu = \Theta\left(\frac{m}{n} \cdot \frac{\sqrt{d}}{\epsilon}\right)
$$

- The $\mathbf{1/\epsilon}$ comes from the scaling required to extract signal from noise.
- The $\mathbf{\sqrt{d}}$ comes from the fact that in $d$ dimensions, the "leakage" of information per coordinate is diluted by a factor of $\sqrt{d}$.

**Why is this successful?**

The aggregator’s job is to take the incoming messages and output the distribution they represent.

1. **If the data is truly skewed ($P$) and everyone is honest:** The aggregator receives $R(P)$, and its output is $P$. This is a "correct" result.
2. **If the data is uniform ($U$) but an attack is happening:** The aggregator receives the mixture $\frac{n-m}{n}R(U) + \frac{m}{n}R^+$, which—thanks to the choice of $R^+$ above—is **mathematically identical** to $R(P)$.

Because the messages are identical in both cases, the aggregator **cannot distinguish** between a world where the data is actually skewed and a world where it is being attacked.

**Conclusion:** Because the aggregator cannot distinguish between "Uniform Data + Attack" and "Skewed Data + No Attack," it *must* fail to be accurate on at least one of those cases.

The vulnerability scales with the domain size $d$. For problems like frequency estimation, the authors show that manipulation can skew the distribution much further.

> Theorem I.2 (Informal):
>
> In a domain of size $d$, an adversary corrupting $m$ users can skew the estimated distribution $P$ away from the uniform distribution $U$ such that17:
>
> $$\|U - P\|_1 = \tilde{\Omega}\left(\frac{m \sqrt{d}}{\epsilon n}\right)$$

The paper introduces two metrics to calibrate attack effectiveness:

- **Breakdown Point:** The minimum fraction of corrupted users needed to make the protocol's accuracy trivial (error $\Omega(1)$). For frequency estimation, this is $m/n \approx \epsilon/\sqrt{d}$.
- **Significance Point:** The fraction of users needed to make the manipulation error as large as the inherent LDP error. For frequency estimation, this is $\sqrt{d/n}$.

| **Problem**                     | **Error (No Manipulation)**                             | **Manipulation Lower Bound (LB)**                            | **Breakdown Point**                              |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| **$l_1/l_1$ (Frequency)**       | $\Theta\left(\frac{d}{\epsilon \sqrt{n}}\right)$        | $\Omega\left(\frac{m \sqrt{d}}{n \epsilon \sqrt{\log n}}\right)$ | $O\left(\epsilon \sqrt{\frac{\log n}{d}}\right)$ |
| **$l_1/l_\infty$ (Histograms)** | $\Theta\left(\sqrt{\frac{\log d}{\epsilon^2 n}}\right)$ | $\Omega\left(\frac{m}{n\epsilon}\right)$                     | $O(\epsilon)$                                    |
| **$l_2/l_2$ (Gradients)**       | $\Theta\left(\frac{\sqrt{d}}{\epsilon \sqrt{n}}\right)$ | $\Omega\left(\frac{m}{n\epsilon}\right)$                     | $O(\epsilon)$                                    |

### Experiments

The authors conducted experiments on two versions of a frequency estimation protocol: **HST** (Hadamard Response/Histogram) and **NR-HST** (Non-Robust HST). These experiments validate that the vulnerability scales with both the domain size $d$ and the number of corrupted users $m$.

The authors fixed the population size and privacy level to observe how the attack scales with dimensionality:

- **Users ($n$):** $2 \times 10^5$
- **Privacy ($\epsilon$):** $1.0$.
- **Variable:** Dimension $d$ (ranging from 4 to 32) and the fraction of corrupted users $m/n$.
- **Metric:** Median $l_1$ error across many trials.

| **Feature**           | **HST (Robust)**                 | **NR-HST (Non-Robust)**                          |
| --------------------- | -------------------------------- | ------------------------------------------------ |
| **Randomness Source** | Publicly shared random string.   | Users sample their own randomness.               |
| **User Message**      | Sends a single bit.              | Sends a bit AND their chosen random vector.      |
| **Vulnerability**     | Adversary can only flip the bit. | Adversary can manipulate the vector and the bit. |

In **NR-HST**, because the corrupted users can choose their own "random" vectors, they have a much larger strategy space. This allows them to skew the results with significantly fewer users than what is required for the robust version.

The **Breakdown Point** is defined as the fraction of corrupted users needed to reach an $l_1$ error of $0.5$ (meaning the estimate is essentially useless). The experiments confirmed the $\sqrt{d}$ relationship: as the domain size $d$ grows, the number of attackers needed to break the system **drops**.

![image-20260120233151432](./assets/image-20260120233151432.png)

As shown above, for $d=32$, a tiny fraction of corrupted users (less than 1% for NR-HST) can completely destroy the protocol's utility.

The authors also found that if an attacker doesn't care about the *entire* distribution but wants to skew a **specific target query** (e.g., making one specific word appear more frequent), the system is even more fragile28. In NR-HST with $d=32$, the error on a target subset increased by a factor of 3 with only **250** corrupted users out of $50,000$ (just $0.5\%$).

### Defense: Public-Coin Protocols

A critical technical finding is that not all "optimal" LDP protocols are created equal.

- **Vulnerable:** Protocols where users choose their own randomization parameters (e.g., NR-HST).
- **Robust:** Protocols using **public-coin models**. By using a shared random string $S$ to compress reports to a single bit, these protocols restrict the adversary's "strategy space," matching the lower bounds for manipulation resistance.

The paper argues that LDP's inherent vulnerability to poisoning suggests caution in deployment. It reinforces the need for cryptographic alternatives like **Multiparty Computation (MPC)** or **Shuffling**, which can emulate centralized DP and limit adversaries to simple (and less destructive) input manipulation.