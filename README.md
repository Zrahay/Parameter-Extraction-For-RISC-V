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
**Total Tokens Processed for this Extraction: 4,693**

# 2. Prompt Engineering Methodology

The prompt underwent multiple refinement stages to improve:

* Structural consistency
* Parameter coverage
* Constraint extraction
* Hallucination control

## 2.1 Baseline Prompt (Zero-Shot)

The initial prompt instructed the model to:

* Extract architectural parameters
* Return YAML output
* Avoid hallucinations

### Observations

**Strengths:**

* High parameter detection
* Good encoding extraction

**Shortcomings:**

* Structural inconsistency
* Category nesting errors
* Naming variance
* Partial constraint capture

Example issue:

```
isa_cpu:
  cache:
```

Cache parameters were incorrectly nested under ISA CPU.

---

## 2.2 Hierarchical Schema Integration

A category schema was introduced:

```
isa_cpu
addressing
cache
csr
privilege
memory
```

### Outcome

Partial improvement, but schema alone was insufficient.
The model still inferred real-world hierarchy (CPU → Cache).

---

## 2.3 Specification Language Interpretation Layer

To improve constraint extraction, normative ISA keywords were mapped to parameter semantics.

### Signals Incorporated

| Spec Term               | Interpretation            |
| ----------------------- | ------------------------- |
| implementation-specific | Configurable parameter    |
| optional                | Feature parameter         |
| may                     | Conditional capability    |
| shall / must            | Mandatory constraint      |
| uniform                 | System-wide constraint    |
| power-of-two            | Alignment/size constraint |
| naturally aligned       | Address constraint        |
| encoding bits           | Structural encoding       |

### Impact

Improved detection of:

* Uniformity constraints
* Alignment properties
* Implementation variability
* Accessibility semantics

## 2.4 Few-Shot Structural Anchoring

Few-shot prompting was introduced to stabilize hierarchy.

### Example 1 — CSR Encoding

Demonstrated:

* Bit ranges
* Width extraction
* Accessibility encoding

This successfully anchored CSR parameters at the top level.

---

## 2.5 Multi-Category Few-Shot Expansion

A second example was added for cache parameters.

### Cache Example Anchored

* Implementation-specific size
* Uniformity constraint

This eliminated incorrect nesting under ISA CPU.

---

## 2.6 Anti-Hallucination Controls

Explicit rules were introduced:

* Do not infer absent parameters
* Do not output placeholders
* Omit unsupported categories entirely

### Result

Eliminated outputs such as:

```
memory:
  page_size: not specified
```

Ensured extraction remained text-grounded.

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

