## (USENIX 2026) Verity: Verifiable Local Differential Privacy

This paper addresses a critical vulnerability in **Local Differential Privacy (LDP)** protocols: **poisoning attacks**. While LDP protects user privacy by adding noise to data before it leaves the user's device, this same noise mechanism allows malicious users to manipulate aggregate results significantly more than they could by simply lying about their inputs.

The authors propose two novel, non-interactive protocols to solve this:

1. **Verity-Auth:** Designed for scenarios where a trusted third party (authorizer) can verify the input (e.g., medical results). It defends

   - Input tampering
   - Randomizer tampering
     - Legendre PRF: split-key mechanism, ZKP-friendly
   - Unlinkability
     - EQS (Equivalence Class) blind signature (input integrity & unlinkability)

   While supporting verification with privacy with:

   - Pederson Commitments: binding secrets for later verification (without revealing secrets in ZKP)
   - ZKP (Bulletproofs and Schnorr’s protocols)
     - Proof of authorized input
     - Proof of the correct PRF (noise bits were generated with the correct split key)
     - Proof of the bit flipping (final response is correctly calculated correctly based on input and noise)

2. **Verity:** Designed for scenarios where users self-generate inputs (e.g., surveys) and **no** ground truth exists.

   - **Threshold HE, (2,2)-Threshold Scheme:** secret key is split between the User and the Server. Used for encrypted noise generation (PRF) and bit-flipping *inside* the encryption. The user blindly computes the response without knowing if their bit was actually flipped.
   - **Verification via Deterministic Re-execution**: instead of checking proof, just re-run homomorphic computation locally
   - ZKPs are now only used to prove that the **encrypted inputs** and **keys** are valid and well-formed (e.g., proving that an encrypted value is actually a 0 or 1, or proving correct partial decryption)

#### The Core Problem: Poisoning & LDP

In standard LDP, users randomize their input $x$ to produce a noisy response $y$. The server aggregates these responses to estimate statistics.

- **The Vulnerability:** Malicious users can attack the system in two ways:
  - **Input Tampering:** Falsifying the input data (possible in any system).
  - **Randomizer Tampering:** Skipping the noise generation and submitting a fabricated response $y$ directly. This is specific to LDP and far more damaging, allowing adversaries to amplify error by a factor of $1/\epsilon$ (where $\epsilon$ is the privacy parameter).
- **Drop-Out Attacks:** A malicious user might generate noise honestly but "drop out" (refuse to send data) if the noise flips their bit in a way they dislike. This paper is the first to explicitly analyze drop-out attacks in verifiable LDP.

#### Tools

**ZKP**:

- ZKP 101: **Schnorr's protocols**

  - You have a secret key $x$ and a public key $Y$, where $Y = g^x$ (in a group where discrete log is hard). You want to prove you know $x$ without revealing it.
  - The prover generates a random number $r$. They compute a "commitment" $T = g^r$ and send $T$ to the Verifier.
  - The Verifier picks a purely random number $c$ (the challenge) and sends it to the Prover.
  - The Prover computes $z = r + c \cdot x$ and sends $z$ to the Verifier.
    - This mixes the secret $x$ with the random $r$. Because $r$ is uniform random and unknown to the verifier, $z$ looks like a random number to the verifier (preserving Zero-Knowledge).
  - The Verifier checks if $g^z \stackrel{?}{=} T \cdot Y^c$.
    - *Math:* $g^z = g^{r + cx} = g^r \cdot (g^x)^c = T \cdot Y^c$.
    - *Result:* The equation balances only if the prover knew $x$. If they didn't know $x$, they couldn't calculate a $z$ that satisfies the equation after seeing the random $c$.

- Towards non-interactive ones: **The Fiat-Shamir Transform**

  - Instead of waiting for a verifier to send a random challenge $c$, the prover generates the challenge themselves by **hashing** their commitment.

    $$c = \text{Hash}(T, \text{Public Inputs})$$

  - Because cryptographic hash functions act like "random oracles," the output $c$ is unpredictable enough to serve as the challenge. This turns a 3-step conversation into a single package $(T, z)$ that anyone can verify later.

- Towards complexity: **Arithmetic Circuits**

  - Schnorr proves you know a simple key. But in Verity-Auth, the user must prove: *"I ran a PRF, generated bits, and flipped my input based on them"*.
  - To prove code execution, we translate the code into an Arithmetic Circuit.
    1. **Flattening:** We break the computation down into addition and multiplication gates.
       - Example from paper: Proving a bit $b$ is binary ($0$ or $1$) becomes the equation $b(1-b) = 0$.
       - Example from paper: Proving the PRF evaluation becomes $w^2 - ((1-b)n+b)(K+j) = 0$.
    2. **R1CS (Rank-1 Constraint System):** We organize these equations into a system of vectors and matrices.
    3. **Polynomials:** We interpolate these vectors into polynomials. If the prover performed the computation correctly, the polynomials will be divisible by a specific "target polynomial."

- The paper specifies using **Bulletproofs**. This is a specific modern ZKP system chosen for two reasons critical to the paper's context:

  1. **No Trusted Setup:** Older systems (like SNARKs) required a "Trust Ceremony" to generate keys. If that ceremony was compromised, adversaries could forge proofs. Bulletproofs do not require this, making them safer for ad-hoc deployment.
  2. **Range Proofs:** Bulletproofs are extremely efficient at proving a value is within a certain range (e.g., $[0, 2^{64}]$). While Verity-Auth focuses on binary checks, the underlying library logic relies on the inner-product arguments that Bulletproofs pioneered to keep proof sizes small (logarithmic size).

HE:

- The user needs to calculate the noise and flip their input bit, but **must not see the result** (otherwise they could strategically lie). HE allows the user to run this calculation "blindly."

- The paper doesn't use generic HE; it uses a specific setup called a **(2,2)-Threshold Leveled HE scheme** based on **BGV**.

  **Why "Threshold"?**

  In standard HE, the user holds the secret key. If Verity used standard HE, the user could simply decrypt the noise, see that it will flip their bit, and change their input to compensate.

  - **The Fix:** The secret key is split into two shares: $sk_{i,U}^{HE}$ (User's share) and $sk_{i,S}^{HE}$ (Server's share).
  - **The Rule:** Decryption requires **both** shares. The user can perform the math on the encrypted data, but they cannot peek at the result because they lack the Server's half of the key.

  **Why "BGV"?**

  The **BGV** scheme is chosen because it supports integer arithmetic (needed for the PRF math) and **SIMD (Single Instruction, Multiple Data)** operations.

  - **Efficiency:** SIMD allows the user to pack thousands of inputs into a single ciphertext and process them all at once (batching), making the heavy HE math scalable.

- Steps:

  **Step 1: Encrypted Setup**

  Before the protocol starts, the user and server exchange encrypted shares of the PRF key.

  - The User has a PRF secret $sk_{i,U}^\mathcal{F}$.
  - The Server has a PRF secret $sk_{i,S}^\mathcal{F}$.
  - The effective PRF key is the sum: $K = sk_{i,U}^\mathcal{F} + sk_{i,S}^\mathcal{F} + \text{counter}$
  - Crucially, this summation happens **inside** the encryption. The User never sees the Server's share, and vice versa.

  **Step 2: Homomorphic PRF Evaluation**

  The user must calculate the **Legendre PRF** on the encrypted key. The formula for the Legendre symbol involves modular exponentiation:

  $$\mathcal{F}(x, K) = 2^{-1} (((K+x)^{(p-1)/2} \pmod p) + 1) \pmod p$$

  - **The Challenge:** HE is very good at addition and multiplication, but terrible at complex operations like exponentiation because each multiplication adds "noise" to the ciphertext. If the noise gets too loud, the data is corrupted.
  - **The Solution:** The BGV parameters (specifically the "ring degree" and "modulus") are carefully tuned to allow just enough "depth" to compute this specific exponentiation formula without the ciphertext collapsing.

  **Step 3: The "Blind" Bit Flip**

  Once the noise bit $b$ is generated (still encrypted), the user computes the Randomized Response flip:

  $$y = (1-x) \cdot b + x \cdot (1-b)$$

  This is done using homomorphic addition and multiplication gates. The result is an encrypted ciphertext $ct_{y_i}$ containing the noisy response. The user holds this box but cannot open it.

#### Solution I: Verity-Auth (With Authorized Input)

This protocol is for settings where a third party holds "ground truth" (e.g., a hospital knows a user's COVID status, or an advertiser knows a conversion occurred).

- **How it Works:**
  1. **Authorization:** The user proves to an Authorizer that their input matches the ground truth. The Authorizer issues a **blind signature**, ensuring the Authorizer cannot link the specific user to the final report sent to the server (preserving unlinkability).
  2. **Noise Sampling:** The user generates noise using a **Pseudorandom Function (PRF)**. The key for this PRF is derived jointly by the user and the server, preventing the user from "cherry-picking" favorable noise.
  3. **Verification:** The user submits the noisy response along with a **Zero-Knowledge Proof (ZKP)**. This proof confirms that the response was generated by correctly running the randomized response mechanism on the signed, authorized input.
- **Security Guarantee:** Prevents both input tampering and randomizer tampering. The only remaining error is the statistical noise inherent to LDP.
- **Efficiency:** Highly efficient. It takes the user roughly **8.37ms** to generate a proof and the server **3.54ms** to verify it.

#### Solution II: Verity (Without Authorized Input)

This protocol is for settings where users generate their own data (e.g., opinion polls), meaning input tampering cannot be prevented. The goal is to prevent **randomizer tampering**.

- **The Challenge:** If a malicious user can see the noise (randomness) in the clear, they can adjust their input to force a specific output. Therefore, the randomness must be hidden even from the user.
- **How it Works:**
  1. **Encrypted Randomness:** The protocol keeps the randomness encrypted. Users evaluate the PRF under **Homomorphic Encryption (HE)**.
  2. **Deterministic Verification:** Instead of using heavy ZKPs for the encryption steps (which is inefficient), Verity leverages the deterministic nature of HE ciphertexts. The server re-runs the homomorphic operations on the encrypted inputs and checks if the resulting ciphertexts match the user's submission.
     1. **Submission:** The User sends their encrypted inputs ($ct_{x_i}$) and the final encrypted output ($ct_{y_i}$) to the Server.
     2. **Re-execution:** The Server takes the encrypted inputs and **re-runs** the exact same homomorphic code locally.
     3. **The Check:** Because the encryption and evaluation operations are deterministic, the Server's result ($ct'_{y_i}$) must match the User's result ($ct_{y_i}$) **bit-for-bit**.
        - If they match: The User followed the protocol.
        - If they differ: The User tried to tamper with the math (e.g., by injecting their own value).
  3. Once the Server verifies the computation was honest:
     1. **Partial Decryption (User):** The User uses their key share ($sk_{i,U}^{HE}$) to partially unlock the result. They add "flooding noise" to this partial decryption to ensure the Server can't mathematically extract their key share from the result.
     2. **Partial Decryption (Server):** The Server uses their key share ($sk_{i,S}^{HE}$) to provide the second half of the unlock.
     3. **Combine:** The two partial decryptions are combined to reveal the final, noisy plaintext $y_i$.
- **Security Guarantee:** Limits the adversary to input tampering only. They cannot exploit the randomness to bias results.
- **Efficiency:** Due to Homomorphic Encryption, this is computationally heavier. It takes the user approximately **1424 seconds** (online time) to generate a verifiable bit, though batching is possible.

------

#### Key Technical Innovations

- **Non-Interactivity:** Both protocols preserve the "one-shot" nature of Randomized Response. Users prepare and submit contributions without needing multiple rounds of online interaction with the server, which is crucial for mobile devices with limited connectivity.
- **Legendre PRF:** The authors use the Legendre PRF for generating randomness because it is efficient and "ZKP-friendly" (easy to prove in arithmetic circuits).
  - Standard PRFs like SHA-256 or AES are "ZKP-hostile" because they rely on bit-shifts and XORs.
  - Verity uses the **Legendre PRF**, which relies on modular exponentiation. This is algebraic and extremely cheap to prove in ZK.
- **ZKP systems:**
  - This paper: **Bulletproofs** (Range Proofs / Arithmetic Circuits).
    - *Why:* It allows for transparent setup (no trusted setup ceremony required) and produces small proofs (logarithmic size).
    - Instead of allowing *any* probability like 0.73 or 0.75, they restrict the probability to powers of $1/2$ (e.g., $1/2, 1/4, 1/8$).
      - See why the alternative is costly below.
    - **No Comparison Needed:** To get a probability of $1/8$, they simply take 3 random bits ($b_1, b_2, b_3$) and multiply them: $b_{final} = b_1 \times b_2 \times b_3$.
    - Since multiplication is just **one gate** in a ZK circuit, this is massively more efficient than the "Large Random + Comparison" method.
  - In Bontekoe et al., PoPETs 2025: **Groth16 (zk-SNARK)**.
    - *Why:* It has **constant proof size** (very small) and **constant verification time**, regardless of how complex the circuit is. This is crucial because their circuit is massive.
    - *Trade-off:* It requires a **Trusted Setup**. If the setup key leaks, proofs can be forged. (The authors argue the server can generate this, assuming a semi-honest server model).
    - For PRF evaluation: They use **Blake2s**, a standard cryptographic hash.
      - *Cost:* Blake2s uses bitwise XORs and rotations. In a ZK circuit (which works over prime fields), these operations must be "emulated" by breaking numbers into bits. This consumes tens of thousands of constraints (~20,000+ constraints per hash).
    - To support *any* LDP mechanism (not just RR), they generate a large random integer from the PRF and compare it to a threshold.
      - For example, to simulate that 0.75 probability using only uniform bits, Bontekoe et al. use the following method:
        - Generate a long string of $\ell$ random bits (e.g., 64 bits). When you interpret these bits as a single number, you get a large integer $R$ uniformly chosen between $0$ and $2^{64}-1$ (the maximum value).
        - Calculate a **threshold** $T$ based on the target probability. If the target is $75\%$ (0.75), the threshold is $75\%$ of the maximum value.
        - **If $R \le T$**: Output **1** (Heads).
        - **If $R > T$**: Output **0** (Tails).
      - To perform a "Less Than" ($<$) comparison in a ZK circuit, you cannot just compare two numbers directly. You must break the "Large Random" number down into its individual binary bits (e.g., all 64 bits) to prove that no "borrow" occurred during the subtraction $R - T$. This adds thousands of constraints to the circuit.
      - You must also strictly prove that the number $R$ is indeed a valid integer within the correct range $[0, 2^{\ell}-1]$, adding further overhead.
    - *Cost:* This requires range checks and greater-than/less-than comparators, which are expensive (requiring bit-decomposition gadgets).
- **Blind Signatures:** Verity-Auth uses a modified EQS-based blind signature to ensure that even if the Authorizer and Server collude, they cannot de-anonymize the user.

#### Performance Comparison

The authors benchmarked their work against concurrent solutions (e.g., Bontekoe et al., PoPETs 2025).

- **Verity-Auth** is approximately **100x more efficient** than the concurrent work, which requires 1.31 seconds per user compared to Verity-Auth's milliseconds execution.
- **Verity** (the encrypted version) is computationally expensive due to HE but is the first to provide robustness by keeping randomness encrypted in a non-interactive setting.

