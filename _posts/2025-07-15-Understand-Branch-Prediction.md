---
---

# Background  
Modern CPUs typically feature pipelines exceeding 10 stages. When executing branch instructions, the jump target address is resolved only at the **Result Buffer** stage (due to indirect jumps). This creates a pipeline "bubble" between **Fetch** and **Result Buffer**. CPUs utilize this period to maximize throughput via **Dynamic Branch Prediction**.  

<img src="image.png" alt="CPU Pipeline Stages" style="width:50%; height:auto;"/>

The prediction mechanism requires:  
1. **Direction prediction** (taken/not-taken)  
2. **Target address prediction**  
This relies on historical execution records stored in hardware tables. Correct predictions improve throughput; mispredictions incur penalties:  
- Pipeline flush (~10 cycles)  
- Correct instruction stream reload  

### Why Prediction Works: Locality Principles  
- **Temporal Locality**: Loops exhibit consistent behavior (e.g., 99% taken in 100-iteration loops).  
- **Spatial Locality**: Correlated branches (e.g., height/weight checks in normally distributed data).  

### Performance Impact  
- Branches constitute 8-16% of dynamic instructions (SPEC92).  
- With 20% branch density and 80% prediction accuracy:  
  - 4-issue OoO CPU suffers ~4.8 wasted instructions per branch  
  - **Throughput reduction ≥50%** in worst-case scenarios  

---

## Branch Instruction Characteristics (RISC)  
| Instruction | Direction Resolved        | Target Resolved       |  
|-------------|---------------------------|-----------------------|  
| `J`         | After Instruction Decode  | After Instruction Decode |  
| `JR`        | After Instruction Decode  | After Register Fetch  |  
| `BEQZ/BNEZ` | After Execute/Reg Fetch*  | After Instruction Decode |  
*\*Assuming zero-detection during register read*  

---

## Reducing Control Flow Penalty  
### Software Solutions  
- **Branch Elimination**: Loop unrolling → increases run length  
- **Early Condition Resolution**:  
  ```c
  // Compute condition early
  bool tmp = (a > b);
  z = x * y;  // Independent operation
  if (tmp) goto L1;  // Reduced stall
  ```  

### Hardware Solutions  
- **Delay Slots**: Fill bubbles with useful work (requires compiler cooperation)  
- **Speculative Execution**: Execute beyond branches using prediction  

---

# Branch Prediction  
## Static Branch Prediction  
- **Heuristic-based**:  
  - Backward branches (loops): Predict *taken*  
  - Forward branches: Predict *not-taken*  
- **ISA-Directed**: Architect-defined hints (e.g., PA-RISC, IA-64)  
  → ~80% accuracy, still used in modern CPUs (e.g., Arm Neoverse)  

## Dynamic Branch Prediction  
### Core Mechanism  
<img src="image-2.png" alt="Prediction Abstract Model" style="width:50%; height:auto;"/>  

### Predictor Primitives (Emer & Gloy, 1997)  
<img src="image-1.png" alt="Predictor State Machine" style="width:50%; height:auto;"/>  

#### Evolution:  
1. **1-bit Predictor**  
   - Fails on alternating patterns (e.g., TNTNTN...)  
   <img src="image-5.png" alt="1-bit Failure Case" style="width:50%; height:auto;"/>  

2. **2-bit Saturating Counter**  
   - Resolves oscillation via hysteresis  
   <img src="image-4.png" alt="2-bit State Machine" style="width:50%; height:auto;"/>  
   <img src="image-3.png" alt="2-bit Implementation" style="width:50%; height:auto;"/>  

### Exploiting Locality  
#### Global History (GHR)  
- Single shift register tracking recent branch outcomes  
- Indexing: `PC XOR GHR`  
  <img src="image-6.png" alt="Global History Predictor" style="width:50%; height:auto;"/>  

#### Local History (LHR)  
- Per-PC history table + private counter array  
  <img src="image-7.png" alt="Local History Predictor" style="width:50%; height:auto;"/>  

### Hybrid Predictors  
#### Combined Indexing Schemes  
<img src="image-9.png" alt="Per-PC GHR Selection" style="width:50%; height:auto;"/>  
<img src="image-10.png" alt="PC-GHR Concatenation" style="width:50%; height:auto;"/>  

#### GShare  
- Index = `PC XOR GHR`  
  <img src="image-11.png" alt="GShare Predictor" style="width:50%; height:auto;"/>  

#### Adaptive Combiner  
- Dynamically selects between global/local predictors  
  <img src="image-12.png" alt="Adaptive Combiner" style="width:50%; height:auto;"/>  

### TAGE Predictor  
- Multi-tiered global history lengths (e.g., 2/4/8/16/32)  
- Indexing: `hash(PC, GHRₙ)`, Tag: `hash(PC, GHRₙ)`  
- Selects longest matching history entry  
  <img src="image-13.png" alt="TAGE Predictor" style="width:50%; height:auto;"/>  

---

## Branch Target Buffer (BTB)  
- Stores `PC → Target` mappings for predicted jumps  
  <img src="image-14.png" alt="BTB Structure" style="width:50%; height:auto;"/>  

### Unified Entry Example  
```cpp
struct BPEntry {
    uint64_t tag;       // High bits of PC
    uint64_t target;    // Predicted jump target
    uint8_t  counter;   // 2-bit saturating counter (direction)
    bool     valid;     // Entry validity
};
```

### Indirect Jump Handling  
- Challenged by multiple targets (e.g., virtual functions)  
- Modern solutions: Dedicated predictors + **Return Address Stack (RAS)**  

## Return Address Stack (RAS)  
- Predicts `call/ret` via hardware stack:  
  - `call`: Push `PC+4`  
  - `ret`: Pop → predicted target  

---

# Modern Predictor Overview  
<img src="image-15.png" alt="Full Prediction Pipeline" style="width:50%; height:auto;"/>  

### Key Components in Commercial CPUs (e.g., Arm Neoverse):  
1. Dynamic direction predictor  
2. Static fallback predictor  
3. Indirect jump predictor  
4. RAS  

--- 

> **Note**: All predictor schematics trade off aliasing conflicts (limited hardware entries) versus coverage. Optimal designs balance history length, table size, and indexing entropy.

# Reference 
https://www.cs.cmu.edu/afs/cs/academic/class/15740-f18/www/lectures/16-branch-prediction.pdf

https://www.cs.sfu.ca/~ashriram/Courses/CS7ARCH/assets/lectures/02_BPred_PreciseInt.pdf

Arm Neoverse CPU TRM document
