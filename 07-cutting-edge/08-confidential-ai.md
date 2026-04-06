# 🔒 Confidential AI & Trusted Execution Environments

> **Phase 7 · Article 8** | ⏱️ 20 min read | 🏷️ `#cutting-edge` `#tee` `#confidential-computing` `#hardware-security`

---

## TL;DR

- **Confidential computing** uses hardware-based **Trusted Execution Environments (TEEs)** to protect data even from the cloud provider hosting it.
- For AI agents, this means: model weights, prompts, and reasoning can be protected *even if the server is compromised* — a fundamentally different security guarantee than software controls alone.
- This is an emerging area with real deployments in 2024-2025 at companies handling extremely sensitive data (healthcare, legal, financial).

---

## The Problem: Who Can See Your AI's Reasoning?

```
CURRENT CLOUD AI DEPLOYMENT:
─────────────────────────────────────────────────────────────────

Your data flows through:
  Your App → [NETWORK] → Cloud Provider → [LLM] → Response

At the cloud provider, your data is accessible to:
  ✅ You (obviously)
  ✅ The cloud provider's infrastructure (server admins)
  ❓ The LLM provider (if using a hosted model)
  ❓ Law enforcement (with subpoena)
  ❓ An attacker who compromises the cloud provider's systems

For most applications: this is acceptable.
For some: it is absolutely not.
  - Healthcare AI (patient data)
  - Legal AI (attorney-client privilege)
  - Financial AI (material non-public information)
  - Government AI (classified information)
  - Competitive intelligence (your business strategies)
```

---

## What Is a TEE?

A Trusted Execution Environment is a secure area of a processor that:

```
TEE PROPERTIES:
─────────────────────────────────────────────────────────────────

ISOLATION:
  Code and data inside the TEE are isolated from the rest of the system.
  Even the OS, hypervisor, and hardware management software cannot
  read the contents of a running TEE.

ATTESTATION:
  The TEE can produce a cryptographic proof ("attestation report")
  that verifies:
  - What code is running inside the TEE
  - That the code has not been tampered with
  - Which hardware platform is being used
  - The security configuration of the TEE

SEALING:
  Data can be "sealed" to a specific TEE instance.
  Sealed data can only be decrypted by that specific TEE running
  that specific code. Even if the storage medium is stolen,
  the data cannot be decrypted.

IN PRACTICE:
  Cloud VM admin: "Let me dump the memory of this AI server"
  TEE: "No. The memory is encrypted with a key only I have."

  Attacker (root access): "Let me read the model weights"
  TEE: "No. The weights are loaded and decrypted only inside the enclave."
```

---

## TEE Technologies

### Intel TDX (Trust Domain Extensions)

```
INTEL TDX (2023+):
─────────────────────────────────────────────────────────────────
WHAT IT IS:
  Intel's VM-level confidential computing technology.
  Protects entire virtual machines, not just applications.
  Available on: 4th Gen Xeon (Sapphire Rapids) and newer.

HOW IT WORKS:
  The VM runs inside an encrypted "Trust Domain."
  Memory is encrypted with a hardware key.
  The VMM (hypervisor) cannot read the VM's memory.

FOR AI:
  Run your entire AI agent in a TDX VM.
  Weights, prompts, reasoning, outputs — all encrypted in memory.
  The cloud provider cannot inspect any of it.

ATTESTATION:
  Before sending sensitive data, the client can verify:
  "Is this really a TDX VM running my expected code?"
  Intel provides attestation verification services.

PROVIDERS:
  Microsoft Azure (DCadsv5 series)
  Google Cloud (C3 confidential VMs)
  Alibaba Cloud
```

### AMD SEV-SNP (Secure Encrypted Virtualization)

```
AMD SEV-SNP:
─────────────────────────────────────────────────────────────────
Similar to Intel TDX but AMD's implementation.
Available on: EPYC 7003 (Milan) and newer.

STRONGER THAN TDX FOR SOME THREATS:
  SEV-SNP includes additional "Reverse Map Table" protection
  that prevents certain memory aliasing attacks.

PROVIDERS:
  Microsoft Azure (DCasv5 series)
  Google Cloud (N2D confidential VMs)
  AWS (currently limited preview)
```

### Intel SGX (Software Guard Extensions)

```
INTEL SGX (2015+, most mature):
─────────────────────────────────────────────────────────────────
WHAT IT IS:
  Application-level enclave (smaller than TDX's VM-level).
  Small, isolated "enclave" of memory within an application.

TRADE-OFF:
  + More mature technology, smaller attack surface
  + Well-tested attestation infrastructure
  - Limited memory (traditionally ~256MB, now larger with EDMM)
  - Requires application rewrite to use enclaves

FOR AI:
  Limited practical use for large LLMs (memory constraints).
  More suited for: key management, token validation,
  specific sensitive operations rather than full inference.

USE CASE: Seal API keys / OAuth tokens inside SGX enclave.
  Even if the application server is compromised, the attacker
  cannot extract the API keys — they're sealed in hardware.
```

---

## Confidential AI in Practice: Architecture Patterns

### Pattern 1: Confidential Inference

```
┌─────────────────────────────────────────────────────────────────────┐
│               CONFIDENTIAL INFERENCE ARCHITECTURE                   │
│                                                                     │
│  USER SIDE:                                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  1. Client attests the TEE (verifies code + hardware)        │  │
│  │  2. Establishes encrypted channel to the TEE                 │  │
│  │  3. Sends encrypted prompt (only TEE can decrypt)            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                     Encrypted channel                               │
│                              │                                      │
│  CLOUD PROVIDER SIDE:                                               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  TEE (e.g., TDX VM):                                         │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  4. Decrypts prompt (inside TEE only)                   │  │  │
│  │  │  5. Runs inference (model weights also in TEE)          │  │  │
│  │  │  6. Encrypts response                                   │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                                                              │  │
│  │  Cloud admin: cannot read prompt, model, or response        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                     Encrypted response                              │
│                              │                                      │
│  USER SIDE:                                                         │
│  7. Decrypts response                                               │
│  Only the user ever sees the plaintext                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Confidential Fine-Tuning

```
PROBLEM: You want to fine-tune a foundation model on proprietary data.
         But the model provider (or the cloud running the training)
         shouldn't be able to see your proprietary training data.

CONFIDENTIAL FINE-TUNING:
  1. Upload encrypted training data to cloud storage
  2. Provision TEE for training job
  3. TEE attests (client verifies code and hardware)
  4. Client sends decryption keys directly to TEE
     (Keys are sealed — only this specific TEE can use them)
  5. TEE decrypts training data inside the enclave
  6. Fine-tuning runs entirely within TEE
  7. Fine-tuned weights are encrypted before leaving TEE
  8. Cloud provider sees only encrypted blobs throughout

REAL DEPLOYMENTS:
  Microsoft Azure Confidential Computing for ML
  Google Cloud Confidential GKE nodes
```

### Pattern 3: Multi-Party Confidential AI

```
USE CASE: Hospital A and Hospital B both want to train a medical AI
          but can't share patient data due to HIPAA.

FEDERATED LEARNING IN TEEs:
  Each hospital:
  1. Runs model training in a TEE (using their private patient data)
  2. Computes gradient updates inside the TEE
  3. Encrypts gradient updates (not raw data) for aggregation

  Central aggregator:
  4. Aggregates encrypted gradients (in a TEE)
  5. Returns aggregated model update to all hospitals

  No hospital ever sees another hospital's data.
  The aggregator never sees raw gradients.
  All processing is verified via attestation.

COMMERCIAL IMPLEMENTATION:
  This is roughly how federated learning with secure aggregation
  works in systems like Google's Cross-Device FL.
```

---

## Attestation: The Trust Mechanism

Attestation is what makes TEEs trustworthy — it's cryptographic proof that the expected code is running on genuine hardware.

```python
# Simplified attestation flow for a confidential AI deployment

class ConfidentialAIClient:
    def __init__(self, tee_service_url: str, expected_code_hash: str):
        self.service_url = tee_service_url
        self.expected_hash = expected_code_hash

    def establish_trusted_session(self) -> bool:
        """
        Before sending any sensitive data, verify the TEE is genuine
        and running the expected code.
        """
        # 1. Request attestation report from the TEE
        response = requests.get(f"{self.service_url}/attestation")
        attestation_report = response.json()

        # 2. Verify with hardware attestation service
        # (Intel IAS for SGX, Azure Attestation, Google Cloud Attestation)
        verified = self.verify_attestation(attestation_report)

        if not verified:
            raise SecurityError("TEE attestation failed — not a genuine TEE")

        # 3. Verify the code hash matches what we expect
        code_hash = attestation_report["enclave_measurement"]
        if code_hash != self.expected_hash:
            raise SecurityError(
                f"Code hash mismatch. Expected: {self.expected_hash[:16]}... "
                f"Got: {code_hash[:16]}..."
                "\nPossible attack: someone modified the AI code!"
            )

        # 4. Get the TEE's ephemeral public key for this session
        self.session_key = attestation_report["ephemeral_public_key"]

        print("✅ TEE attestation verified — secure session established")
        return True

    def send_confidential_prompt(self, prompt: str) -> str:
        """Send a prompt that only the TEE can decrypt."""
        # Encrypt using the TEE's session key
        encrypted_prompt = encrypt_for_tee(prompt, self.session_key)

        response = requests.post(
            f"{self.service_url}/inference",
            json={"encrypted_prompt": encrypted_prompt}
        )

        # Decrypt the response (only we have the client's decryption key)
        return decrypt_from_tee(response.json()["encrypted_response"])
```

---

## Current Limitations of TEEs for AI

```
CURRENT LIMITATIONS (2025):
─────────────────────────────────────────────────────────────────

PERFORMANCE:
  Memory encryption adds 5-15% overhead.
  SGX enclave memory limits constrain large model sizes.
  TDX overhead has improved significantly but still present.

LARGE MODEL SUPPORT:
  LLaMA-70B, GPT-4-scale models require terabytes of memory.
  Current TEE implementations may struggle with models this large.
  Active research area: paging encrypted model weights through TEE.

SIDE CHANNEL ATTACKS:
  TEEs don't eliminate all attack surface.
  Cache timing attacks, power analysis, and speculative execution
  vulnerabilities (Spectre/Meltdown) can potentially leak data.
  Intel and AMD continually patch these — stay updated.

TRUSTED CODE STILL MATTERS:
  The TEE is only as trustworthy as the code inside it.
  If the inference code has a bug that leaks data,
  the TEE won't prevent it.
  Attestation verifies the code hash — not that the code is bug-free.

SUPPLY CHAIN:
  The root of trust is the hardware manufacturer.
  You're trusting Intel or AMD at the hardware level.
  For nation-state adversaries, this may not be sufficient.
```

---

## Who Needs Confidential AI?

```
HIGH-NEED USE CASES:
─────────────────────────────────────────────────────────────────

HEALTHCARE:
  Patient diagnostic AI processing PHI.
  Mental health chat agents with sensitive disclosures.
  Drug discovery AI using proprietary compound data.

LEGAL:
  Contract analysis AI on privileged legal documents.
  Litigation support AI on case strategy.

FINANCIAL:
  Trade strategy AI on MNPI (Material Non-Public Information).
  Risk modeling AI on confidential portfolio data.

GOVERNMENT:
  Classified document analysis.
  Intelligence gathering agents.

COMPETITIVE INTELLIGENCE:
  Product roadmap analysis.
  M&A strategy documents.

LOWER-NEED USE CASES (software controls sufficient):
  Customer service agents (data is business-operational, not ultra-sensitive)
  Public-facing content generation
  Internal productivity agents (lower sensitivity)
```

---

## Getting Started with Confidential AI

```
PRACTICAL FIRST STEPS:
─────────────────────────────────────────────────────────────────

STEP 1: Assess your sensitivity requirements
  → What data does your agent process?
  → Who (beyond you) could currently see it?
  → What are the regulatory/contractual requirements?

STEP 2: Start with existing confidential cloud offerings
  → Azure Confidential Computing (TDX VMs for AI inference)
  → Google Confidential Space (attestation + VM-level TEE)
  → These handle most of the complexity

STEP 3: Understand what you're protecting and from whom
  → Cloud provider: TEE protects from them ✅
  → Insider threats at your own org: TEE doesn't help
  → Nation-state: TEE helps but has limits
  → Bugs in your code: TEE doesn't help

STEP 4: Implement attestation in your client
  → Verify the TEE before sending sensitive data
  → Pin expected code hashes in your client
  → Alert if attestation fails

STEP 5: Monitor the field
  → NVIDIA H100 "Hopper" architecture includes TEE for GPUs
  → AMD SEV-SNP on MI300X (future)
  → Hardware-level ML security is evolving rapidly
```

---

## Further Reading

- [Intel TDX Documentation](https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html)
- [AMD SEV-SNP Architecture Guide](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/programmer-references/55766_SEV-KM_API_Specification.pdf)
- [Microsoft Azure Confidential Computing](https://azure.microsoft.com/en-us/solutions/confidential-compute/)
- [Google Confidential Space](https://cloud.google.com/blog/products/identity-security/announcing-confidential-space)
- [NVIDIA Confidential Computing on H100](https://www.nvidia.com/en-us/data-center/solutions/confidential-computing/)
- [Gramine Project: Run unmodified apps in SGX](https://gramine.readthedocs.io/)

---

*← [Prev: Agent Security Benchmarks](./07-agent-benchmarks.md) | [Next: Zero Trust for Agents →](./09-zero-trust-agents.md)*
