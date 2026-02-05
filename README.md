# CMCC Truth Model Spreadsheet 
## The motivation, and how it works internally

---

## The Challenge

> If some truths are defined by convention, but others are constrained by reality itself — what does "source of truth" even mean?

This repository is an operational response to that question.

---

## My Prompt to ChatGPT 5.2:

> `How would you contend with consistent truths that don't come from construction. Like the amount of energy in an electron. We can claim different measurement systems for what to call the amount of energy, Joules, Foot-pound force, etc. But what isn't changing is the exact amount of energy in the electron. E=mc^2 would hold as long as the appropriate conversions are done. 0 Kelvin is the same temperature as ~-273 degrees Celsius. So what is the source of truth in this? To me, it must be reality. If we can't "change the ontology" what do you mean by source?`
> 
> To answer this questions, please Google the CMCC Conjecture - then build a CMCC Model for the rules described above:
> 
> Then - write a python script that can inject those rules into an excel spreadsheet with all of those moving parts.  A world model that I can download and play with here as an interactive model as a Spreadsheet.
>
> Please include enough mock data to demonstrate each element of the questions proposed. i.e. there should be true/false fields for the final questions being answered.

## The Response (thought for 28m)

The challenge draws a distinction between two kinds of truth:

1. **Constructed truths** — Unit systems, naming conventions, measurement standards. These are "true because we agreed." We could have defined the meter differently; we chose not to.

2. **Reality-constrained truths** — The electron's rest energy, the equivalence of 0 K and −273.15 °C, the relationship E=mc². These hold regardless of how we label them.

The key insight is that **representation changes, but underlying quantity does not**. When you convert electron-volts to Joules, you're not changing the energy — you're changing how you write it down. The invariant is the physical quantity; the variant is notation.

This spreadsheet makes that distinction operational:

- All values are converted to a **canonical SI layer** — a shared substrate where representations collapse to the same number.
- Physical laws like E=mc² are **computed and compared** against reference values.
- Claims are reduced to **explicit TRUE/FALSE tests**: does the invariant hold within tolerance?

The answer to "what is the source of truth?" becomes concrete:

- For **constructed truths**, the source is the definition table (`D_Units`).
- For **reality-constrained truths**, the source is **invariance under representation change** — the fact that multiple paths through different unit systems all converge to the same canonical value.

Reality reveals itself through consistency. You can't make 0 K ≠ −273.15 °C by relabeling. You can't break E=mc² by switching units. The spreadsheet demonstrates this by letting you change representations and watching the invariants hold.

---

## Technical Specification

This workbook implements a CMCC-style model using spreadsheet primitives:

* **S (Schema):** defined by sheet structures + named column meanings
* **D (Data):** the actual rows in the `D_*` sheets
* **L (Lookups):** cross-table joins via `XLOOKUP`/`VLOOKUP`-style formulas
* **F (Formulas):** calculated fields (canonicalization, unit conversion, invariants)
* **A (Aggregations/checks):** pass/fail claims and question truth values derived from multiple fields

It’s intentionally “database-like”: each sheet behaves like a table with keys, and other sheets reference those keys.

---

# Workbook layout

### 1) `S_Overview`

A human-readable map of the CMCC primitives and what each sheet represents.

---

### 2) `D_Units`  (Units table)

Defines unit systems and how to convert **to a canonical SI base**.

**Key columns**

* `UnitID` (primary key): e.g. `J`, `eV`, `K`, `C`
* `Quantity`: grouping label (Energy, Temperature)
* `ToSI_Mult`: multiplier to convert a value in this unit to SI (e.g. eV → J uses `1.602176634E-19`)
* `ToSI_Offset`: offset added after multiplying (used for affine transforms like °C → K)

**Canonicalization rule**
For any value `x` in some unit:

* `SI = x * ToSI_Mult + ToSI_Offset`

Examples:

* Joule (J): mult = 1, offset = 0
* electron-volt (eV): mult = 1.602176634e-19, offset = 0
* Kelvin (K): mult = 1, offset = 0
* Celsius (°C): mult = 1, offset = 273.15  (so `K = °C + 273.15`)

This directly models your point: **the representation changes, the underlying quantity (in canonical SI) does not**.

---

### 3) `D_Constants`  (Constants table)

Stores “reality-constrained” constants and also “constructed/defined” constants.

**Key columns**

* `ConstID` (primary key): e.g. `c`, `e`, `me`, `me_c2_ref`
* `Value`
* `UnitID` (FK into `D_Units`)
* `Notes` (what it is / why it matters)

**How it’s used**
Constants are not assumed to be in SI. They get canonicalized via lookups into `D_Units`:

* Pull `ToSI_Mult`, `ToSI_Offset` for the constant’s `UnitID`
* Compute a canonical SI value

This is how the sheet cleanly separates:

* “defined-by-convention” constants (like `e` and `c` in SI)
* empirically measured constants (like electron mass `me`)

---

### 4) `D_Instances`  (Scenario instances)

This sheet holds specific “cases” you can test.

Think: “the same electron rest energy expressed in two different units”, or “absolute zero expressed in K and °C”.

**Key columns**

* `InstanceID` (primary key)
* `Topic` (Energy or Temperature)
* `ObservedValue`
* `ObservedUnitID` (FK to `D_Units`)
* `Canonical_SI` (calculated)
* `DerivedValue` / `DerivedUnitID` (optional: for computed comparisons)

**Internal mechanic**
Each instance row computes:

* `Canonical_SI = ObservedValue * Unit.ToSI_Mult + Unit.ToSI_Offset`

So if you enter:

* `0` in `K` → canonical is `0 K`
* `-273.15` in `C` → canonical is `0 K`
  Those two become *equal at the canonical layer*.

---

### 5) `D_Calculations`  (Derived invariants like E = m c²)

This sheet computes invariants from constants and compares to reference values.

Typical derived rows:

* `E_calc = me * c^2` (computed in Joules)
* Compare against `me_c2_ref` (CODATA electron rest energy) in Joules

**Key columns**

* `CalcID` (primary key)
* `FormulaType` (e.g., `E=mc^2`)
* `Result_SI` (calculated)
* `Reference_SI` (looked up)
* `AbsError` and `WithinTolerance` (TRUE/FALSE)

**Internal mechanic**

* Look up `me` and `c` from `D_Constants`
* Canonicalize both to SI
* Compute `E_calc_SI = me_SI * (c_SI^2)`
* Look up `me_c2_ref` and canonicalize
* Compute error and tolerance pass/fail

This is the “reality” layer: even if you change which units you *display*, the canonical SI computation should still agree.

---

### 6) `D_Claims`  (Atomic TRUE/FALSE checks)

This is the heart of the workbook for “final questions.”

Each claim is a testable proposition with a computed boolean.

**Key columns**

* `ClaimID` (primary key)
* `ClaimText` (plain English)
* `TestType` (e.g., `SI_EQUALITY`, `DERIVED_MATCH`, `UNIT_CONVERSION_CONSISTENT`)
* `LeftValue_SI`, `RightValue_SI` (calculated numbers)
* `Tolerance`
* `Pass` (TRUE/FALSE)

**How claims are computed**
Claims use the canonical SI values from `D_Instances` and/or `D_Calculations`.

Examples of claim logic:

* **Invariance across representation:**
  `ABS(InstanceA.Canonical_SI - InstanceB.Canonical_SI) <= Tolerance`
* **Derived-law agreement:**
  `ABS(Calc.Result_SI - Calc.Reference_SI) <= Tolerance`

So every “truth” you want to argue about becomes:

* a specific comparison
* a specific tolerance
* a TRUE/FALSE outcome

---

### 7) `D_Questions`  (Higher-level “final questions”)

This sheet answers your conceptual questions by referencing one or more claims.

**Key columns**

* `QuestionID`
* `QuestionText`
* `DependsOnClaimIDs` (comma-separated)
* `Answer` (TRUE/FALSE)

**Internal mechanic**
`Answer` is computed as an AND across the referenced claims:

* TRUE only if *all* dependent claims are TRUE

So the “final questions” are explicitly grounded in checkable claims, not vibes.

---

# How this maps to your philosophical point

### Constructed truth (coordination)

* Units, conversion factors, labels, and *representations* live in `D_Units`.
* They’re “true because defined” in the sense that the mapping is chosen/standardized.

### Reality-constrained invariants

* Canonical SI values provide a shared substrate:

  * Convert representation → canonical
  * Compare canonicals
* Physical equivalences (0 K = −273.15 °C) and invariant energies (mₑc²) show up as **stable equalities at the canonical layer**, even when surface forms differ.

### “What is the source of truth?”

In this model:

* The “source” for *constructed truths* is `D_Units` (definitions).
* The “source” for *invariant truths* is `Canonical_SI` agreement + derived-law agreement (`D_Calculations` + `D_Claims`).

---

# How to play with it

### Change a representation (should NOT break invariant claims)

* Change an instance’s `ObservedUnitID` and adjust `ObservedValue` accordingly (e.g., replace eV with J using conversion).
* The canonical SI should stay the same and the invariance claims should remain TRUE.

### Change a constant (may break derived-law claims)

* Modify electron mass `me` or reference `me_c2_ref` in `D_Constants`.
* You should see the derived E=mc² comparison flip depending on tolerance.

### Tighten tolerance (can flip TRUE→FALSE)

* `Tolerance` fields in `D_Claims` are deliberate “engineering range” knobs.
* Reduce tolerance and watch borderline claims fail.

---

# Implementation notes (for transparency)

* All cross-sheet relationships are done by **key lookup** (`UnitID`, `ConstID`, `InstanceID`, etc.).
* All “truth” outputs reduce to **explicit boolean cells** (`Pass`, `WithinTolerance`, `Answer`).
* Canonicalization uses the affine transform: `SI = x*mult + offset` so it can represent both:

  * pure scaling (J↔eV)
  * offset + scaling (°C↔K)
