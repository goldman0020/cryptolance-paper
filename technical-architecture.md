CryptoLance Technical Architecture

Overview

CryptoLance is a decentralized project-based work and payment platform built on the TRON network.

Its core financial infrastructure combines:

- TRON smart contracts
- TRC20 USDT
- wallet-based cryptographic authorization
- smart-contract escrow
- programmable project states
- automated settlement
- dispute handling
- on-chain platform accounting

The purpose of this architecture is to allow project payments to be governed by programmable blockchain rules rather than relying entirely on an off-chain payment intermediary.

The smart contract is responsible for enforcing the financial lifecycle of a project.

---

Architecture at a Glance

                    ┌─────────────────────┐
                    │      Client         │
                    │   TRON Wallet       │
                    └──────────┬──────────┘
                               │
                         Authorization
                               │
                               ▼
                    ┌─────────────────────┐
                    │  CryptoLance       │
                    │  Smart Contract     │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
                    ▼                     ▼
             TRC20 USDT Escrow      Project State
                    │                     │
                    │                     │
                    └──────────┬──────────┘
                               │
                     ┌─────────┴─────────┐
                     │                   │
                     ▼                   ▼
               Normal Release        Dispute
                     │                   │
                     │                   ▼
                     │            Resolution Rules
                     │                   │
                     └─────────┬─────────┘
                               ▼
                       Final Settlement

---

1. Smart Contract Layer

The core financial logic is implemented in a Solidity smart contract deployed for execution on TRON.

The contract maintains the state of individual projects through structured on-chain records.

Each project contains information including:

- project identifier
- client address
- freelancer address
- project amount
- client and freelancer fees
- freelancer performance deposit
- dispute fees
- escrow status
- completion status
- dispute status
- dispute resolution information

The contract therefore functions as both the escrow layer and the programmable settlement layer.

---

2. Wallet-Based Authorization

CryptoLance uses cryptographic signatures to obtain participant authorization.

Before funds are frozen, the relevant project parameters are assembled into a deterministic message.

The message includes the critical project information required to bind the authorization to the specific project, including:

- project ID
- client
- freelancer
- project amount
- platform fees
- freelancer deposit
- dispute fee

The resulting message is hashed and signed by the relevant wallet.

The smart contract subsequently verifies the signatures against the expected participant addresses.

This creates a cryptographic link between:

Wallet
   ↓
Signature
   ↓
Project Parameters
   ↓
Smart Contract Validation

The contract therefore does not rely solely on an off-chain statement that a participant approved the project.

---

3. Signature Verification

The current implementation uses Solidity's "ecrecover" mechanism for signature verification.

The contract receives the signed message and signature data and recovers the signing address.

The recovered address must match the participant address associated with the project.

Conceptually:

Signed Project Data
        ↓
     Hashing
        ↓
TRON signed-message format
        ↓
   ECDSA Signature
        ↓
    ecrecover()
        ↓
Recovered Address
        ↓
Compare with expected signer

This verification occurs inside the smart contract.

---

4. Escrow Initialization

The escrow lifecycle begins with "freezeFunds()".

Before accepting the operation, the contract validates the project and authorization state.

The process includes:

1. Checking that the project has not already been frozen.
2. Validating the participant addresses.
3. Validating the project amount.
4. Calculating the applicable platform fees.
5. Reconstructing the authorization message.
6. Verifying the client signature.
7. Verifying the freelancer signature.
8. Checking the required USDT balances.
9. Checking the required token allowances.
10. Executing the required TRC20 transfers.
11. Recording the project state on-chain.

The funds are transferred into the smart contract through TRC20 "transferFrom()" operations.

This is the point at which the project moves from an agreed state into an actual blockchain-controlled escrow state.

---

5. TRC20 USDT Settlement

CryptoLance uses TRC20 USDT as the settlement asset.

The smart contract interacts with the token through the standard ERC20-compatible interface implemented for TRC20 tokens.

The contract performs operations including:

- balance verification
- allowance verification
- "transferFrom()"
- "transfer()"

The system therefore does not require a proprietary settlement token for the core project lifecycle.

The escrowed value remains represented by TRC20 USDT throughout the settlement process.

---

6. Project State Machine

The contract maintains explicit state for each project.

A simplified representation is:

                 ┌─────────────┐
                 │   Created   │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │    Frozen   │
                 └──────┬──────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
       Normal Release          Dispute
              │                   │
              │                   ▼
              │             Resolution
              │                   │
              └─────────┬─────────┘
                        ▼
                 ┌─────────────┐
                 │  Completed  │
                 └─────────────┘

State checks prevent incompatible operations from being executed.

For example, a normal release cannot proceed while an active dispute exists, and a completed project cannot be completed again.

---

7. Normal Release

For a project without an active dispute, "releaseFunds()" performs the normal settlement process.

The contract calculates the applicable amounts for:

- freelancer payment
- returned dispute fees
- platform revenue

The project is marked as completed and the relevant TRC20 transfers are executed.

The settlement is therefore calculated and performed by the contract rather than requiring a separate off-chain payment calculation.

---

8. Dispute Initiation

CryptoLance also implements a dedicated dispute state.

"initiateDispute()" first verifies that:

- the project has frozen funds
- the project has not been completed
- a dispute has not already been initiated

The contract reconstructs the project authorization data and verifies the participant signatures.

At least one authorized participant must approve the dispute action.

Once accepted, the project is marked as having an active dispute.

The normal release path is then blocked.

---

9. Dispute Resolution

After a dispute has been initiated, the contract provides a separate settlement path through "releaseFundsAfterDispute()".

The resolution contains:

- ruling type
- client percentage
- freelancer percentage

The contract verifies that:

Client Percentage + Freelancer Percentage = 100%

It then calculates the distributable amount according to the predefined project rules.

The resulting amounts are transferred to the appropriate parties.

The resolution is also recorded as part of the project's on-chain state.

This creates an auditable relationship between:

Project
   ↓
Dispute
   ↓
Resolution
   ↓
Distribution

---

10. Platform Fee Accounting

Platform fees are calculated by the smart contract.

Rather than relying exclusively on an external accounting database, the contract maintains platform balances associated with the owner address.

When a project is settled, the applicable platform revenue is added to the platform balance.

The contract also contains an automatic withdrawal mechanism based on a configured threshold.

This allows the platform's settlement accounting to remain connected to the same contract that executes project payments.

---

11. Reentrancy Protection

State-changing financial operations are protected by a reentrancy guard.

The contract maintains an execution status and prevents a protected function from being entered again before the previous execution has completed.

The guard is applied to the main financial operations, including:

- escrow initialization
- dispute initiation
- normal release
- dispute settlement
- platform withdrawal

This provides an additional protection layer around the contract's token-transfer operations and state transitions.

---

12. Ownership and Administrative Control

The contract contains an ownership mechanism for administrative operations.

The owner is used for functions such as:

- executing platform-level operations
- updating the platform fee within the configured limits
- withdrawing platform balances
- transferring contract ownership

The financial lifecycle of individual projects remains governed by the contract's state checks and validation logic.

Administrative authority and participant authorization are therefore represented as separate concepts within the contract.

---

13. On-Chain Data Integrity

Project state is stored directly in the smart contract.

The contract records information necessary to reconstruct the financial state of a project, including its participants, amounts, fees, escrow status, completion state, and dispute resolution.

This creates a persistent on-chain representation of the project lifecycle.

The platform interface can read this state, but the underlying financial state is maintained by the blockchain execution layer.

---

14. Execution Flow

A simplified normal project lifecycle is:

1. Project terms prepared
          ↓
2. Client authorization
          ↓
3. Freelancer authorization
          ↓
4. Signatures submitted
          ↓
5. Smart contract verifies signatures
          ↓
6. Balances and allowances verified
          ↓
7. TRC20 USDT transferred to escrow
          ↓
8. Project marked as frozen
          ↓
9. Project completed
          ↓
10. Release transaction
          ↓
11. Contract calculates settlement
          ↓
12. TRC20 USDT distributed
          ↓
13. Project marked completed

The dispute path branches from the frozen state:

Frozen Project
      ↓
Dispute Initiated
      ↓
Resolution
      ↓
Distribution
      ↓
Completed

---

15. Resource Considerations on TRON

Smart-contract execution on TRON consumes network resources, particularly Energy for contract execution and Bandwidth for transaction data.

CryptoLance's financial operations involve several resource-consuming components, including:

- cryptographic verification
- persistent storage
- project-state updates
- TRC20 contract calls
- token transfers
- event emission

The escrow initialization path is particularly substantial because it combines authorization verification, token validation, multiple token interactions, and project-state initialization in one operation.

Consequently, runtime optimization should be based on measured Energy consumption rather than assumptions about a single instruction or operation.

---

16. Engineering Principle

The architecture follows a simple principle:

«The blockchain should enforce the financial rules that matter.»

The frontend can provide the user experience.

The backend can coordinate application-level operations.

But the critical financial state and settlement rules are implemented at the smart-contract layer.

This creates a separation between:

User Interface
      ↓
Application Coordination
      ↓
Blockchain Enforcement
      ↓
TRC20 Settlement

The blockchain layer remains responsible for the final financial state transitions.

---

17. Current Implementation

The core architecture described above has been implemented and tested on the TRON test environment.

The implementation includes:

- Solidity smart-contract logic
- TRC20 USDT integration
- wallet signature verification
- escrow initialization
- project state management
- normal settlement
- dispute initiation
- dispute resolution
- platform fee accounting
- automated platform withdrawal logic
- reentrancy protection
- ownership management

The result is a working implementation of the core CryptoLance financial lifecycle rather than a purely conceptual architecture.

---

Conclusion

CryptoLance implements a complete programmable project-settlement lifecycle on TRON.

At its core, the system connects:

cryptographic authorization

→ TRC20 USDT

→ smart-contract escrow

→ on-chain project state

→ normal or disputed settlement

→ programmable distribution

The architecture is designed so that the smart contract is not simply a payment endpoint.

It is the layer that validates authorization, controls escrowed funds, enforces project state transitions, calculates settlement amounts, and records the resulting financial state.

This provides the technical foundation for a cross-border project-based work platform in which payment and settlement rules can be executed directly by blockchain infrastructure.
