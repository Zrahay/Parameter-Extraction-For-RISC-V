# Parameter-Extraction-For-RISC-V

# AI-Assisted Extraction of Architectural Parameters from RISC-V Specifications

## Submission Requirements

Please submit the following information:

1. Details about the LLM(s) used (name, version, context length etc.)
2. Prompts: Especially how you developed and refined the prompt, dealt with model hallucinations etc.
3. Results formatted as a YAML file with fields for name, description, type and constraints etc.

---

# 1. LLM Details

**Model Used:** GPT-5.2 (OpenAI)
**Context Window:** Up to 400,000 tokens
**Max Output Tokens:** Up to 128,000 tokens
**Temperature:** 0 (deterministic extraction)
**Total Tokens Processed for this Extraction:** 4,693

---

# 2. Prompt Engineering Methodology

The prompt design process followed an iterative refinement approach.
Three distinct prompt versions were developed to progressively improve extraction accuracy, structural consistency, and hallucination control.

The evolution of prompts is summarized below.

---

## 2.1 Prompt v1 — Baseline Parameter Extraction

The first prompt focused on direct architectural parameter extraction without enforcing hierarchy.

**Capabilities:**

* Extracted sizes, limits, and widths
* Detected encoding bit ranges
* Identified accessibility and alignment properties
* Returned parameters in flat YAML format

**Strengths:**

* High parameter detection rate
* Accurate encoding extraction
* Good identification of structural properties

**Shortcomings:**

* No architectural grouping
* Inconsistent parameter naming
* Missed scope constraints (e.g., uniformity qualifiers)
* Limited interpretation of normative specification language

This version established baseline extraction feasibility but lacked structural organization.

---

## 2.2 Prompt v2 — Hierarchical Schema Integration

To address structural limitations, a hierarchical schema was introduced:

```
isa_cpu
addressing
cache
csr
privilege
memory
```

The goal was to group parameters under architectural subsystems.

**Enhancements Introduced:**

* Category-based organization
* Improved readability of outputs
* Subsystem-level parameter mapping

**Observed Improvements:**

* CSR encodings grouped correctly
* Addressing and accessibility fields better structured

**Limitations Observed:**

Schema text alone did not fully anchor hierarchy.

The model sometimes inferred real-world architectural containment, producing outputs such as:

```
isa_cpu:
  cache:
```

This indicated ontology drift despite schema presence.

---

## 2.3 Prompt v3 — Few-Shot + Constraint-Aware Hierarchical Prompt

The final prompt iteration incorporated multiple refinements to resolve structural and semantic shortcomings.

### Enhancements Introduced

**1. Specification Language Interpretation Layer**

Normative ISA keywords were mapped to parameter semantics:

* implementation-specific → configurable parameter
* optional → feature capability
* shall / must → mandatory constraint
* uniform → system-wide constraint
* power-of-two → size/alignment constraint
* naturally aligned → address constraint

This significantly improved constraint extraction fidelity.

---

**2. Few-Shot Structural Anchoring**

Few-shot examples were added to stabilize category placement:

* CSR encoding example → anchored encoding hierarchy
* Cache parameter example → prevented ISA nesting

This eliminated cross-category drift.

---

**3. Category Independence Rules**

Explicit instructions ensured:

* All architecture categories remain top-level
* No category nesting allowed

---

**4. Anti-Hallucination Controls**

Rules were introduced to suppress speculative outputs:

* Do not infer absent parameters
* Do not generate placeholders
* Omit unsupported categories entirely

---

## 2.4 Evolution Summary

| Capability            | Prompt v1 | Prompt v2    | Prompt v3 |
| --------------------- | --------- | ------------ | --------- |
| Parameter detection   | High      | High         | High      |
| Constraint extraction | Moderate  | High         | Very High |
| Structural grouping   | None      | Partial      | Stable    |
| Hierarchy correctness | —         | Inconsistent | Correct   |
| Hallucination control | Low       | Medium       | Strong    |

---

The final prompt (v3) produced structurally consistent, constraint-complete YAML outputs aligned with ISA specification semantics, making it suitable for automated architectural knowledge extraction.

---

# 3. Extraction Pipeline

### Workflow

1. Load hierarchical prompt template
2. Load specification snippet from external file
3. Send both via API call
4. Generate structured YAML output
5. Store results in `/outputs` directory

### Design Choice

Snippets were not embedded in prompts to ensure:

* Reusability
* Scalability
* Dataset modularity

---

# 4. Results

Parameter extraction was tested on RISC-V privileged specification snippets.

---

## 4.1 Cache Parameters — Extracted

```yaml
cache:
  cache_block_range_properties:
    description: Cache blocks represent a contiguous, naturally aligned power-of-two (NAPOT) range of memory locations.
    type: addressing
    constraints:
      contiguous: true
      naturally_aligned: true
      power_of_two_or_napot: true

  cache_block_identification_by_physical_address:
    description: A cache block is identified by any of the physical addresses corresponding to the underlying memory locations.
    type: addressing
    constraints:
      address_space: physical
      any_address_within_block_identifies_block: true

  cache_capacity:
    description: Capacity of a cache.
    type: size
    constraints:
      implementation_specific: true

  cache_organization:
    description: Organization of a cache.
    type: organization
    constraints:
      implementation_specific: true

  cache_block_size:
    description: Size of a cache block.
    type: size
    constraints:
      implementation_specific: true
      uniform_throughout_system_in_initial_cmo_extensions: true

  cache_discovery_mechanism:
    description: Execution environment provides software a means to discover information about caches and cache blocks in a system.
    type: feature
    constraints:
      provided_by_execution_environment: true
```

---

## 4.2 CSR Parameters — Extracted

```yaml
csr:
  csr_address_width:
    description: Width of the CSR address encoding space.
    type: encoding
    constraints:
      bit_range: "[11:0]"
      width_bits: 12

  csr_count_limit:
    description: Maximum number of CSRs addressable in the CSR encoding space.
    type: limit
    constraints:
      maximum_csrs: 4096

  csr_accessibility_field:
    description: Upper CSR address bits used by convention to encode CSR read/write accessibility and privilege accessibility.
    type: encoding
    constraints:
      bit_range: "[11:8]"
      width_bits: 4
      function: read_write_and_privilege_accessibility

  csr_read_write_indicator_bits:
    description: CSR address bits indicating whether a CSR is read/write or read-only.
    type: encoding
    constraints:
      bit_range: "[11:10]"
      width_bits: 2
      encodings:
        read_write: ["00", "01", "10"]
        read_only: ["11"]

  csr_lowest_privilege_access_bits:
    description: CSR address bits encoding the lowest privilege level that can access the CSR.
    type: encoding
    constraints:
      bit_range: "[9:8]"
      width_bits: 2
      function: lowest_privilege_level_accessible
```

---
