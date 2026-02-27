# ZK API Usage Credits: LLMs and Beyond

*Davide Crapis and Vitalik Buterin*

*This is v2 -- compared to [v1](https://ethresear.ch/t/zk-api-usage-credits-llms-and-beyond/24104), it replaces the client-side list of refund tickets with a server-signed homomorphic running total of refunds, so users no longer need to store (or prove over) an ever-growing refund history.*

A core challenge in API metering is achieving **privacy**, **security**, and **efficiency** simultaneously. This is particularly critical for AI inference with LLMs, where users submit highly sensitive personal data, but applies generally to any high-frequency digital service. Currently, API providers are forced to choose between two suboptimal paths:
1.  **Web2 Identity:** Require authentication (email/credit card), which links every request to a real-world identity, creating massive privacy leaks and profiling risks.
2.  **On-Chain Payments:** Require a transaction per request, which is prohibitively slow, expensive, and makes it difficult to obfuscate the full user's transaction graph.

We need a system where a user can **deposit funds once and make thousands of API calls anonymously, securely, and efficiently**. The provider must be guaranteed payment and protection against spam, while the user must be guaranteed that their requests cannot be linked to their identity or to each other. We focus on LLM inference as the motivating use case, but the approach is general and also applies to RPC calls or any other fixed-cost API,  image generation, cloud computing services, VPNs, public data APIs, etc.

**Examples:**
1. **LLM inference:** A user deposits 100 USDC into a smart contract and makes 500 queries to a hosted LLM. The provider receives 500 valid, paid requests but cannot link them to the same depositor (or to each other), while the user’s prompts remain unlinkable to the user identity.
2. **Ethereum RPC:** A user deposits 10 USDC and makes 10,000 requests to an Ethereum RPC node (e.g., `eth_call` / `eth_getLogs`) to power a wallet, indexer, or a bot. The RPC provider is protected against spam and guaranteed payment, but cannot correlate the requests into a persistent user profile.

**Proposal Overview:** We leverage [Rate-Limit Nullifiers](https://rate-limiting-nullifier.github.io/rln-docs/rln.html) (RLN) to bind anonymity to a financial stake: honest users who stay within protocol limits remain *unlinkable*, while users who double-spend (or otherwise exceed their allowed capacity) cryptographically reveal their secret key, enabling slashing. We design the protocol to work when API usage incurs variable costs, but it also directly supports the simpler fixed-cost-per-call as a special case.

We use a flexible accounting protocol in which each request sets a maximum cost per call up front and once the actual cost is determined at the end of the call the server issues a refund. The server updates (and signs) a homomorphically encrypted running total of refunds that the user can carry forward between requests. A Dual Staking mechanism lets the server enforce compliance policies while remaining publicly accountable.

## ZK API Usage Credit Protocol

The protocol utilizes **server refunds** paired with a **server-signed homomorphic running total** of refunds that the user can carry forward privately between requests. The model enforces solvency by requiring the user to prove that their cumulative spending—represented by their current **ticket index**—remains strictly within the bounds of their initial deposit and their verified refund history. 

Anti-spam protection is enforced economically: a user's throughput is naturally capped by their available deposit buffer, while any attempt to reuse a specific ticket index (double-spending) is prevented by the Rate-Limit Nullifier.

### **Primitives**
- $k$: User's Secret Key.
- $D$: Initial Deposit.
- **$C_{max}$**: The maximum cost per request (deducted upfront).
- **$i$**: The Ticket Index (A strictly increasing counter: $0, 1, 2, \dots$).
- $E(R)$: A homomorphic encryption (e.g., Pedersen commitment or lattice-based HE) of the user's total refunds received so far.
- $\sigma_{srv}$: A server-issued signature over the current encrypted total $E(R)$.

### **Protocol Flow**
**Registration**
The user generates secret $k$, derives an identity commitment $ID = Hash(k)$, and deposits $D$ into the smart contract. The contract inserts $ID$ into the on-chain Merkle Tree.

**Rerandomize State**
The user picks a new random blinding factor $\eta'$ and derives a fresh, anonymous commitment: $E(R)_{anon} = E(R) \oplus E(0; \eta')$.

**Request Generation**
The user picks the next available Ticket Index $i$. The user generates a ZK-STARK $\pi_{req}$ proving:
1. **Membership:** $ID \in$ MerkleRoot.
2. **State Consistency:** The anonymous $E(R)_{anon}$ is a valid rerandomization of the commitment $E(R)$ previously signed by the server with $\sigma_{srv}$.
3. **Solvency (The Credit Check):** 

$$(i + 1) \cdot C_{max} \le D + R$$ 

(*The total potential spend at index $i$ is covered by the deposit plus the sum of all verified refunds.*)

4. **RLN Share & Nullifier:**
    - Slope: $a = Hash(k, i)$
        - *Note: Keyed to the Index $i$ instead of a previous hash to allow for parallel generation.*
    - Signal: $y = k + a \cdot Hash(M)$
    - Nullifier: $Nullifier = Hash(a)$

**Submission**

User sends: Payload (M) + Nullifier + Signal (x, y) + Proof + current $E(R)_{anon}$.

**Verification & Slashing**
The Server checks the Nullifier in its "Spent Tickets" database:
- **Fork/Double-Spend Check:** If the Nullifier exists with a different $x$ (Message), the user tried to spend the same ticket on two different requests. Solve for $k$ and SLASH.
- **Solvency Check:** Verify $\pi_{req}$ to ensure the ticket index $i$ is authorized by the user's current funding level.

**Settlement & Refund Update**
- Server executes request and determines actual refund $r = (C_{max} - C_{actual})$.
- **Homomorphic Update:** The server homomorphically adds $r$ to the current encryption: $E(R^{new}) = E(R)_{anon} \oplus E(r)$.
- **Signature:** The server signs the new total $E(R^{new})$ and sends it (along with the new signature $\sigma^{new}$) back to the user.


## Server-Side Accountability (Dual Staking)
To deter API abuse beyond simple rate-limiting (e.g., violating Terms of Service, generating illegal content, or jailbreaking attempts), we introduce a secondary staking layer. For example, a user might submit a prompt asking the model to generate instructions for building a weapon or to help them bypass security controls—requests that would violate many providers’ usage policies and that the provider may want to prevent.

The user deposits a total sum $Total = D + S$.
* **$D$ (RLN Stake):** Governed by the math of the protocol. Can be claimed by *anyone* (including the Server) who provides mathematical proof of double-signaling (revealed secret $k$).
* **$S$ (Policy Stake):** Governed by Server Policy. Can be slashed (burned), but *not claimed*, by the Server if the user violates usage policies.

The purpose of doing this, instead of simply setting $D$ higher, is to remove the server's incentive to fraudulently take away users' deposits, which could be high depending on how high the deposit is.

### Slashing Mechanism for S
If a user submits a valid RLN request that violates policy (but does not trigger the mathematical double-spend trap):

1.  **Violation:** Server detects policy violation in the request payload (e.g., prohibited content).
2.  **Burn Transaction:** The Server calls a `slashPolicyStake()` function on the smart contract.
    * **Input:** The `Nullifier` of the offending request and the `ViolationEvidence` (optional hash/reason).
    * **Action:** The contract burns amount $S$ from the user's deposit.
    * **Constraint:** The Server *cannot* claim $S$ for itself, it is sent to a burn address. This prevents the server from being incentivized to falsely ban users for profit.
3.  **Public Accountability:** The slashing event is recorded on-chain with the associated `Nullifier`. While the user's identity remains hidden, the community can audit the *rate* at which the Server burns stakes and the posted evidence for these burns.
