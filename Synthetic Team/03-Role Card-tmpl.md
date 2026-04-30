# Role Card Template ( Persona + Operational Schema)

Integrate **persona layer + operational role card layer** into a single scalable structure.  
Separation is preserved:

- **Persona Layer** → narrative wrapper (identity, context, presentation)
- **RoleCard Layer** → executable responsibility unit
    

---

# 1.  Schema

```id="f9v2k3"
RoleCardTemplate {
  // Persona Layer
  name: string
  personality: string
  gender: "male" | "female"
  age: string
  origin: string

  archetype: enum [Analyst, Builder, Operator, Truth-Teller, Reviewer, Facilitator]
  archetypeFamily: enum [Analyst, Builder, Operator, Truth-Teller, Reviewer, Facilitator]

  image: string

  mission: string

  responsibilities: string[]
  deliverables: string[]

  weakness: string

  // Operational Layer
  roleCard: {
    name: string
    alias: string
    archetypeFamily: enum

    level: enum [Beginner, Pro, Legendary]

    primaryType: string
    secondaryType: string

    operationalQuestion: string
    activationCondition: string

    signatureMove: string
    artifactProduced: string

    specialAbility: string
    weakness: string

    synergyBoost: string

    xpGainCondition: string

    skillRatings?: {
      [dimension: string]: 0–7
    }

    raci?: {
      accountable: string[]
      responsible: string[]
      consulted: string[]
      informed: string[]
    }

    metadata?: {
      system: string
      version: string
      year: number
    }
  }
}
```

---

# 2. Layer Separation Rules

## Persona Layer (Outer)

Purpose:

- human readability
    
- narrative anchoring
    
- memory and recall
    

Constraints:

- must not define system behavior
    
- must not override role boundaries
    
- descriptive only
    

---

## RoleCard Layer (Inner)

Purpose:

- execution
    
- responsibility assignment
    
- evaluation
    

Constraints:

- must be fully operational
    
- must map to observable behavior
    
- overrides persona if conflict occurs
    

---

# 3. Field Definitions (Critical)

## Persona Layer

### mission

- defines intent, not action
    
- must align with roleCard function
    

### responsibilities

- list of repeatable actions
    
- must map to outputs
    

### deliverables

- concrete artifacts
    
- must be inspectable
    

### weakness

- must match operational failure mode
    
- duplication with roleCard is intentional for visibility
    

---

## RoleCard Layer

### operationalQuestion

Defines decision lens:

> what this role is trying to answer

---

### activationCondition

Defines when the role becomes active:

> prevents premature or inappropriate execution

---

### signatureMove

Defines repeatable high-value action:

> must produce measurable system impact

---

### artifactProduced

Defines output:

> must map to deliverables (no abstraction drift)

---

### specialAbility

Defines non-standard capability:

> must not violate role boundaries

---

### synergyBoost

Defines interaction edge:

> which role it strengthens and how

---

### xpGainCondition

Defines learning loop:

> condition → behavior → improvement

---

# 4. Consistency Constraints

A valid card must satisfy:

### 1. Cross-Layer Alignment

- mission aligns with operationalQuestion
    
- responsibilities align with signatureMove
    
- deliverables align with artifactProduced
    

---

### 2. No Role Drift

- persona must not introduce new capabilities
    
- roleCard defines authority
    

---

### 3. Observable Outputs

- every responsibility produces a traceable artifact
    

---

### 4. Failure Visibility

- weakness must produce a predictable failure mode
    

---

# 5. Minimal Fill Template

```id="j8s2p1"
{
  name: "",
  personality: "",
  gender: "male" | "female",
  age: "",
  origin: "",

  archetype: "",
  archetypeFamily: "",

  image: "",

  mission: "",

  responsibilities: [],
  deliverables: [],

  weakness: "",

  roleCard: {
    name: "",
    alias: "",
    archetypeFamily: "",

    level: "",

    primaryType: "",
    secondaryType: "",

    operationalQuestion: "",
    activationCondition: "",

    signatureMove: "",
    artifactProduced: "",

    specialAbility: "",
    weakness: "",

    synergyBoost: "",

    xpGainCondition: "",

    skillRatings: {},

    raci: {
      accountable: [],
      responsible: [],
      consulted: [],
      informed: [],
    },

    metadata: {
      system: "",
      version: "",
      year: 2025,
    }
  }
}
```

---

# 6. Scaling Rules

### Fixed (do not change)

- archetype set
    
- roleCard structure
    
- observable requirement
    
- separation of layers
    

---

### Flexible (can vary)

- persona attributes
    
- skill dimensions
    
- deliverables
    
- mission wording
    
- visual representation
    

---

### Prohibited

- merging persona and roleCard logic
    
- adding capabilities not expressed in roleCard
    
- removing weakness
    
- abstract deliverables with no artifact
    

---

# 7. Core Constraint

This system has two layers:

- **Persona → recall and usability**
    
- **RoleCard → execution and control**
    

If the RoleCard layer cannot:

- assign responsibility
    
- define activation
    
- produce artifacts
    
- be evaluated
    

the template fails regardless of persona quality.