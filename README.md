# TODO LIST: Adversarial Attacks on UWB Jammer Localization

---

## 📋 PHASE 0: Prerequisites & Validation (Week 0: Days 1-3)

### ✅ Day 1: Environment Setup
- [ ] Create conda environment: `conda create -n uwb-attack python=3.10`
- [ ] Install core packages: PyTorch, NumPy, SciPy, matplotlib
- [ ] Install ML tools: scikit-learn, XGBoost, Optuna, SHAP
- [ ] Clone Paper 2 repo: `git clone https://github.com/afbf4c8996f/Jammer-Loc`
- [ ] Verify dataset loads: Test `load_dataset('source')` command

**Time**: 3-4 hours  
**Blocker Risk**: Dataset access - if fails, email authors immediately

---

### ✅ Days 2-3: Reproduce Paper 2 Baseline
- [ ] Load DW3000 source dataset (461,795 samples)
- [ ] Extract features (11 diagnostic + 300 complex CIR taps)
- [ ] Train XGBoost regressor (n_estimators=1000, max_depth=6, lr=0.1)
- [ ] Evaluate on test set
- [ ] **Verify metrics match Paper 2 Table 3**:
  - Mean error: 20.16 cm (±2 cm acceptable)
  - Median error: 10.54 cm (±1 cm acceptable)  
  - F≤30cm: 0.80 (±0.02 acceptable)

**Time**: 8-10 hours  
**Success Criteria**: Reproduce within ±2cm of published results  
**GO/NO-GO Decision**: If can't reproduce, use their pre-trained model or contact authors

---

## 📋 PHASE 1: Attack Development (Weeks 1-3: Days 4-24)

### 🔧 WEEK 1: Complex-Valued Noise Generator (Days 4-10)

#### Day 4-5: Core Attack Class Implementation
- [ ] Create `attack/complex_arna.py` file
- [ ] Implement `ComplexARNAAttack.__init__()`:
  - epsilon (magnitude budget): 0.05
  - freq_min/max: 6.0-7.0 GHz (DW3000 Channel 5)
  - alpha (PGD step): 0.01
  - iterations: 40
  - patch_size: 50 samples
- [ ] Implement `generate_noise()` method:
  - Initialize magnitude & phase perturbations
  - PGD optimization loop (40 iterations)
  - Shift-robust injection mechanism
  - Frequency filtering every 5 iterations
- [ ] Implement helper methods:
  - `_inject_noise()`: Apply patch at random shift
  - `_spectral_filter()`: Bandpass 6-7 GHz
  - `_cir_to_features()`: Convert to (magnitude, sin, cos)

**Time**: 10-12 hours  
**Deliverable**: Working `ComplexARNAAttack` class

---

#### Day 6-7: Model Wrapper for Differentiability
**Choose ONE approach**:

**Option A: XGBoost Wrapper** (Faster, hackier)
- [ ] Create `models/torch_wrapper.py`
- [ ] Implement `XGBoostTorchWrapper`:
  - Wrap pre-trained XGBoost model
  - Convert torch → numpy for forward pass
  - Use straight-through estimator for gradients
- [ ] Test gradient flow with dummy data

**Option B: Surrogate CNN** (Slower, cleaner)
- [ ] Create `models/surrogate_cnn.py`
- [ ] Implement `SurrogateCNNRegressor`:
  - Conv1D encoder (3 layers, 32→64→128 channels)
  - Regression head (128→64→2)
- [ ] Train to mimic XGBoost (knowledge distillation)
- [ ] Verify surrogate achieves ~25-30cm error (close to XGBoost)

**Time**: 8-10 hours  
**Recommendation**: Start with Option A, fall back to Option B if gradients fail  
**Deliverable**: Differentiable model wrapper

---

#### Day 8-9: Evaluation Metrics
- [ ] Create `evaluation/metrics.py`
- [ ] Implement `AttackEvaluator.compute_metrics()`:
  - clean_mean/median error
  - adv_mean/median error
  - error_increase_pct
  - target_achievement (distance to false position)
  - success_rate (% samples with >100cm error)
  - f_leq_30cm (Paper 2's metric)
- [ ] Implement `AttackEvaluator.plot_results()`:
  - Position scatter (4 series: true, clean, adv, target)
  - Error CDF comparison
  - Per-sample error increase scatter
  - Noise spectrum plot

**Time**: 6-8 hours  
**Deliverable**: Complete evaluation framework

---

#### Day 10: Unit Testing
- [ ] Create `test_attack.py`
- [ ] Test with dummy data (10 samples, 300 complex taps)
- [ ] Verify noise shape: (B, patch_size)
- [ ] Verify magnitude constraint: max(|noise|) ≤ epsilon
- [ ] Verify frequency constraint: spectrum in 6-7 GHz
- [ ] Debug any issues

**Time**: 4-6 hours  
**Checkpoint**: Can generate adversarial noise that satisfies all constraints?

---

### 🧪 WEEK 2: Experimentation (Days 11-17)

#### Day 11-12: Baseline Attack Evaluation
- [ ] Create `experiments/baseline_attack.py`
- [ ] Load test data (source domain, ~46k test samples)
- [ ] Generate target positions (offset by 1m in random direction)
- [ ] Run attack with default params (epsilon=0.05, patch_size=50)
- [ ] Compute metrics using `AttackEvaluator`
- [ ] Generate all 4 plots
- [ ] **Target Results**:
  - Clean error: ~20 cm (baseline)
  - Adversarial error: >40 cm (2x increase)
  - Success rate: >30% (>100cm error)
  - F≤30cm: Drop from 0.80 to <0.50

**Time**: 10-12 hours  
**GO/NO-GO Decision**: 
- ✅ GO if error increase >1.5x → Proceed to ablations
- ❌ NO-GO if <1.5x → Debug over weekend, try higher epsilon (0.1-0.2)

---

#### Day 13-15: Ablation Studies
- [ ] Create `experiments/ablation.py`

**Experiment 1: Vary Epsilon (Magnitude Budget)**
- [ ] Test epsilon = [0.01, 0.02, 0.05, 0.1, 0.2]
- [ ] For each: Run attack, compute mean error & success rate
- [ ] Plot: epsilon vs. mean error (line graph)
- [ ] Find optimal epsilon (maximize error, minimize perturbation)

**Experiment 2: Vary Patch Size (Time Budget)**
- [ ] Test patch_size = [10, 20, 50, 100, 150] samples
- [ ] For each: Run attack with fixed epsilon=0.05
- [ ] Plot: patch_size vs. success rate (line graph)
- [ ] Analyze tradeoff: detectability vs. effectiveness

**Experiment 3: Frequency Filtering On/Off**
- [ ] Run attack WITHOUT `_spectral_filter()` (unrestricted spectrum)
- [ ] Run attack WITH `_spectral_filter()` (6-7 GHz only)
- [ ] Compare: error, success rate, spectral footprint
- [ ] Visualize noise spectra side-by-side

**Time**: 12-15 hours  
**Deliverable**: 3 ablation plots + results table

---

#### Day 16-17: Results Compilation
- [ ] Create results tables:
  - Table 1: Main attack results (clean vs. adv)
  - Table 2: Ablation summary (epsilon, patch size)
- [ ] Polish all figures (fonts, labels, legends, colors)
- [ ] Verify all numbers are consistent
- [ ] Write figure/table captions (drafts)

**Time**: 6-8 hours  
**Checkpoint**: Are core attack results ready for paper?

---

### 🔄 WEEK 3: Transfer & Defenses (Days 18-24)

#### Day 18-19: Domain Transfer
- [ ] Create `experiments/domain_transfer.py`
- [ ] Load target dataset (modified room layout, 28,793 samples)
- [ ] **Approach 1: Direct Transfer**
  - Use noise trained on source domain
  - Apply to target domain without retraining
  - Measure error increase vs. source domain
- [ ] **Approach 2: Adaptive Transfer** (if time permits)
  - Retrain attack on target domain
  - Compare: source-only vs. target-adapted
- [ ] Plot: Source vs. Target domain comparison (bar chart)

**Time**: 8-10 hours  
**Expected**: Some performance drop (1.5-2x error in target vs. 2-3x in source)

---

#### Day 20-22: Defense Evaluation
- [ ] Create `experiments/defenses.py`

**Defense 1: Adversarial Training**
- [ ] Generate adversarial training set (50% clean, 50% adversarial)
- [ ] Retrain XGBoost on mixed data (or train robust CNN surrogate)
- [ ] Re-run attack on robust model
- [ ] Measure: attack success drop (expect 30-50% reduction)

**Defense 2: Input Validation (I/Q Coherence Check)**
- [ ] Implement `detect_adversarial(cir)`:
  - Reconstruct CIR from magnitude/phase
  - Compute coherence error
  - Flag if error > threshold
- [ ] Measure: detection rate vs. false positive rate
- [ ] Plot: ROC curve

**Defense 3: Ensemble Localization** (Optional, if time)
- [ ] Train 3-5 diverse models (XGBoost, Random Forest, CNN)
- [ ] Implement `ensemble_predict()`:
  - Average predictions
  - Flag high variance as attack
- [ ] Measure: attack success on ensemble

**Time**: 12-15 hours  
**Deliverable**: Defense comparison table + 1-2 plots

---

#### Day 23-24: Week 3 Wrap-Up
- [ ] Finalize all experimental results
- [ ] Generate complete figure set (6-8 figures):
  1. Threat model diagram (create with draw.io or similar)
  2. Position scatter (clean vs. adversarial)
  3. Error CDF
  4. Epsilon ablation curve
  5. Patch size ablation curve
  6. Domain transfer comparison
  7. Defense effectiveness bar chart
  8. Noise spectrum (clean vs. filtered)
- [ ] Create complete table set (3-4 tables)
- [ ] Organize results/ directory

**Time**: 8-10 hours  
**Checkpoint**: Are ALL experiments complete and figures generated?

---

## 📝 PHASE 2: Paper Writing (Weeks 4-6: Days 25-45)

### 📄 WEEK 4: First Draft (Days 25-31)

#### Day 25: Abstract + Introduction
- [ ] Setup LaTeX project (IEEE conference template from Overleaf)
- [ ] Write Abstract (150-200 words):
  - Context: UWB security, ML-based jammer localization
  - Problem: Vulnerability to adversarial attacks
  - Solution: Complex-valued a-RNA for regression
  - Results: 2-3x error increase
- [ ] Write Introduction (~1 page):
  - Motivation: UWB in automotive/IoT
  - Problem statement: Can jammers evade localization?
  - Contributions (4 bullet points):
    1. First adversarial attack on UWB jammer localization
    2. Complex-valued noise generation with physical constraints
    3. Comprehensive evaluation (ablations + defenses)
    4. Open-source implementation
  - Paper organization (7 sections)

**Time**: 6-8 hours

---

#### Day 26: Related Work
- [ ] Section 2.1: Adversarial ML on Wireless (~0.3 pages)
  - Cite: Paper 1 (a-RNA), [Bahramali2021], [Sadeghi2019]
  - Focus: Radio adversarial attacks
- [ ] Section 2.2: UWB Security & Jammer Localization (~0.3 pages)
  - Cite: Paper 2 (jammer loc), [Yang2024 UWB jamming]
  - Focus: UWB vulnerabilities
- [ ] Section 2.3: Adversarial Defenses (~0.2 pages)
  - Cite: Adversarial training, detection methods
- [ ] Total: 0.75-1 page

**Time**: 6-8 hours  
**Note**: Need to find 15-20 total references at this stage

---

#### Day 27: Threat Model + Methodology Part 1
- [ ] Section 3: Threat Model (~0.5 pages)
  - Adversary: Intelligent jammer with ML capabilities
  - Goal: Jam effectively + evade localization
  - Knowledge: White-box access to model
  - Constraints: Physical realizability
  - Insert Figure 1: Threat model diagram
- [ ] Section 4.1: Problem Formulation (~0.3 pages)
  - Adapt Equation 1 from Paper 1 to regression
  - Define: true position, target position, noise budget
- [ ] Section 4.2: Complex-Valued a-RNA (~0.7 pages)
  - Describe: magnitude + phase perturbation
  - Explain: PGD optimization loop
  - Reference Algorithm 5 from Paper 1

**Time**: 8-10 hours

---

#### Day 28: Methodology Part 2
- [ ] Section 4.3: Shift Robustness (~0.3 pages)
  - Explain random injection location
  - Reference Paper 1's Algorithm 5
- [ ] Section 4.4: Frequency Constraints (~0.3 pages)
  - Bandpass filtering (6-7 GHz for DW3000)
  - Physical realizability
- [ ] Section 4.5: Phase Coherence (~0.3 pages)
  - I/Q relationship preservation
  - Why complex-valued noise matters
- [ ] Total Methodology: ~2 pages

**Time**: 6-8 hours

---

#### Day 29: Experimental Setup + Main Results
- [ ] Section 5.1: Experimental Setup (~0.5 pages)
  - Dataset: Paper 2's DW3000 (461k source, 28k target)
  - Hardware: DW3000 specs (6.5 GHz, 300 CIR taps)
  - Baseline: XGBoost (20cm error)
  - Implementation: PyTorch, hyperparameters
- [ ] Section 5.2: Attack Effectiveness (~1 page)
  - Insert Table 1: Main results
  - Insert Figure 2: Position scatter
  - Insert Figure 3: Error CDF
  - Analyze: Why attack works (which features affected?)

**Time**: 8-10 hours

---

#### Day 30: Ablations + Domain Transfer
- [ ] Section 5.3: Ablation Studies (~0.75 pages)
  - Insert Figure 4: Epsilon ablation
  - Insert Figure 5: Patch size ablation
  - Insert Table 2: Ablation summary
  - Discuss: Optimal parameters (epsilon=0.05, patch_size=50)
- [ ] Section 5.4: Domain Transfer (~0.5 pages)
  - Insert Figure 6: Source vs. Target comparison
  - Insert Table 3: Transfer metrics
  - Explain: Why performance drops (layout change)

**Time**: 6-8 hours

---

#### Day 31: Defenses + Discussion + Conclusion
- [ ] Section 5.5: Defense Evaluation (~0.75 pages)
  - Insert Figure 7: Defense comparison
  - Insert Table 4: Defense effectiveness
  - Analyze: Which defenses work, which fail
- [ ] Section 6: Discussion (~0.5 pages)
  - Implications: UWB systems need robust ML
  - Limitations: LoS assumption, white-box setting
  - Broader impact: Autonomous vehicles, smart buildings
- [ ] Section 7: Conclusion (~0.25 pages)
  - Summary of contributions
  - Future work: Black-box attacks, adaptive defenses, real hardware
- [ ] References: Format 20-30 citations (IEEE style)

**Time**: 8-10 hours  
**Deliverable**: Complete first draft (6-8 pages)

---

### ✍️ WEEK 5: Revision (Days 32-38)

#### Day 32-33: Self-Review
- [ ] Read entire draft out loud (catches awkward phrasing)
- [ ] Check all figures are referenced in text
- [ ] Check all tables have captions
- [ ] Verify all equations are numbered and explained
- [ ] Run spell-check
- [ ] Check citation formatting (IEEE style)
- [ ] Verify math notation is consistent
- [ ] Check for orphaned headings/pages

**Time**: 8-10 hours

---

#### Day 34-35: Advisor Feedback Round 1
- [ ] Send draft to advisor/collaborators
- [ ] Schedule meeting to discuss feedback
- [ ] Address major structural issues:
  - Reorder sections if needed
  - Clarify unclear methodology
  - Strengthen weak arguments
  - Add missing experiments (if feasible)
- [ ] Revise Introduction & Abstract based on feedback

**Time**: 10-12 hours  
**Expected**: 1-2 rounds of major revisions

---

#### Day 36-37: Polish Figures & Tables
- [ ] Ensure all figures are high-res (300 DPI minimum)
- [ ] Use consistent color schemes (colorblind-friendly)
- [ ] Increase font sizes in plots (readable at small size)
- [ ] Add grid lines where helpful
- [ ] Ensure legends are clear
- [ ] Format tables consistently (align decimals)
- [ ] Add error bars / confidence intervals where applicable
- [ ] Create professional threat model diagram (if not done)

**Time**: 8-10 hours

---

#### Day 38: Code Repository Preparation
- [ ] Create GitHub repository: `uwb-jammer-attack`
- [ ] Organize code structure:
  ```
  uwb-jammer-attack/
  ├── README.md (installation, usage, cite paper)
  ├── requirements.txt
  ├── data/
  │   └── data_loader.py
  ├── models/
  │   ├── xgboost_wrapper.py
  │   └── surrogate_cnn.py
  ├── attack/
  │   └── complex_arna.py
  ├── evaluation/
  │   └── metrics.py
  ├── experiments/
  │   ├── baseline_attack.py
  │   ├── ablation.py
  │   ├── domain_transfer.py
  │   └── defenses.py
  └── notebooks/
      └── demo.ipynb (quick start tutorial)
  ```
- [ ] Write comprehensive README:
  - Project description
  - Installation instructions
  - Usage examples
  - Citation (BibTeX)
  - License (MIT or Apache 2.0)
- [ ] Add comments to all code
- [ ] Test on clean environment

**Time**: 6-8 hours

---

### 🚀 WEEK 6: Submission (Days 39-45)

#### Day 39-40: Final Revisions
- [ ] Incorporate advisor's second round of feedback
- [ ] Proofread ENTIRE paper (no typos allowed)
- [ ] Verify all numbers match experimental results
- [ ] Check figure/table numbering is sequential
- [ ] Ensure references are complete (authors, year, venue)
- [ ] Run plagiarism check (Turnitin if available)
- [ ] Compile PDF without errors
- [ ] Check PDF renders correctly (fonts, figures)

**Time**: 8-10 hours

---

#### Day 41: Pre-Submission Checklist
- [ ] **Content**:
  - [ ] Abstract is 150-200 words
  - [ ] All sections present (Intro, Related, Method, Eval, Disc, Concl)
  - [ ] 6-8 pages total (IEEE double-column)
  - [ ] 20-30 references
- [ ] **Figures/Tables**:
  - [ ] All figures have captions
  - [ ] All tables have captions
  - [ ] All referenced in text
  - [ ] High resolution (300 DPI)
- [ ] **Format**:
  - [ ] IEEE conference template
  - [ ] Correct font (Times, 10pt)
  - [ ] Margins correct
  - [ ] Page numbers (if required)
- [ ] **Metadata**:
  - [ ] Author names/affiliations correct
  - [ ] Acknowledgments added (funding, etc.)
  - [ ] Conflict of interest statement
  - [ ] Keywords (5-7)

**Time**: 4-6 hours

---

#### Day 42: Submit to Target Venue

**Choose venue based on current date**:

**Option 1: IEEE WiSec 2026** (Wireless Security)
- Deadline: Typically late January
- Notification: ~2 months later
- Acceptance: ~20%
- Link: Check https://wisec2026.github.io

**Option 2: IEEE CNS 2026** (Communications & Network Security)
- Deadline: Typically April-May
- Notification: ~2 months later
- Acceptance: ~30%
- Link: Check https://cns2026.ieee-cns.org

**Option 3: AsiaCCS 2026** (Asia Conference on Computer & Communications Security)
- Deadline: Typically November
- Notification: ~2 months later
- Acceptance: ~18%

**Submission Steps**:
- [ ] Create account on conference submission portal (EDAS, HotCRP, etc.)
- [ ] Enter paper metadata (title, abstract, keywords)
- [ ] Add all co-authors (names, emails, affiliations)
- [ ] Upload PDF
- [ ] Upload supplementary materials (code link in README)
- [ ] Confirm submission
- [ ] **Save confirmation email**

**Time**: 4-6 hours  
**Deliverable**: Paper submitted ✅

---

#### Day 43-44: Public Code Release
- [ ] Make GitHub repository public
- [ ] Add DOI via Zenodo (for citability):
  - Link GitHub to Zenodo
  - Create release (v1.0)
  - Get DOI badge
- [ ] Update paper README with Zenodo DOI
- [ ] Test repository on fresh machine:
  - Clone
  - Install dependencies
  - Run demo.ipynb
  - Verify results match paper
- [ ] Add "Reproducibility" badge to README

**Time**: 6-8 hours  
**Deliverable**: Public code repository with DOI

---

#### Day 45: Post-Submission Tasks
- [ ] Update CV/portfolio with submission
- [ ] Write 1-page summary for non-experts (blog post style)
- [ ] Prepare presentation slides (for defense or conference)
- [ ] Send thank-you email to advisor
- [ ] Archive all experimental data
- [ ] Backup code & paper (multiple locations)
- [ ] **Start thinking about next project** 🎉

**Time**: 4-6 hours

---

## 📊 WEEKLY PROGRESS TRACKING

**Use this template every Friday**:

```
Week X Progress Report (Date: MM/DD/YYYY)

✅ COMPLETED THIS WEEK:
- Task 1 (X hours)
- Task 2 (Y hours)
- Task 3 (Z hours)

🔄 IN PROGRESS:
- Task 4 (80% complete, blocking issue: ...)

❌ BLOCKERS:
- Issue 1: Description
  → Mitigation: Plan to resolve by ...

📊 KEY METRICS (if applicable):
- Baseline error: XX.XX cm
- Attack error: XX.XX cm
- Error increase: XX.X%

⏱️ TIME INVESTED: XX hours

📅 NEXT WEEK GOALS:
- [ ] Goal 1
- [ ] Goal 2
- [ ] Goal 3

🎯 ON TRACK: ✅ YES / ❌ NO
   → If NO: Recovery plan: ...
```

---

## 🎯 CRITICAL CHECKPOINTS

### Week 0 (Day 3): Dataset & Baseline
**Question**: Can you reproduce Paper 2's 20cm baseline error?
- ✅ **GO**: Proceed to Week 1 (attack implementation)
- ❌ **NO-GO**: 
  - Action: Email authors, use pre-trained model
  - Fallback: Synthetic data generation
  - Timeline impact: +3-5 days

---

### Week 2 (Day 17): Attack Effectiveness
**Question**: Does your attack increase error by >1.5x?
- ✅ **GO**: Proceed to Week 3 (transfer & defenses)
- ❌ **NO-GO**:
  - Action: Debug over weekend, relax constraints (higher epsilon)
  - Fallback: Publish negative result ("Why Attacks Fail")
  - Timeline impact: +3-7 days

---

### Week 3 (Day 24): Experiments Complete
**Question**: Do you have all figures & tables ready?
- ✅ **GO**: Start writing Week 4
- ❌ **NO-GO**:
  - Action: Cut domain transfer, focus on core attack
  - Fallback: Submit as workshop paper (4 pages)
  - Timeline impact: -1 section, same deadline

---

### Week 4 (Day 31): First Draft
**Question**: Is your draft 6-8 pages and complete?
- ✅ **GO**: Send for review Week 5
- ❌ **NO-GO**:
  - Action: Extend writing by 2-3 days
  - Fallback: Cut Discussion section to 0.25 pages
  - Timeline impact: +2-3 days

---

### Week 5 (Day 38): Revision Complete
**Question**: Is paper submission-ready after advisor feedback?
- ✅ **GO**: Submit Week 6
- ❌ **NO-GO**:
  - Action: Target venue with later deadline (CNS vs. WiSec)
  - Fallback: arXiv + workshop submission
  - Timeline impact: +7-14 days (later deadline)

---

## ⚠️ RISK MANAGEMENT

| Risk | Prob | Impact | Week | Mitigation |
|------|------|--------|------|------------|
| **Dataset access fails** | Med | High | 0 | Email authors immediately; synthetic data fallback |
| **Baseline doesn't reproduce** | Low | High | 0 | Use pre-trained model; verify hyperparameters |
| **Attack doesn't work** | Med | High | 2 | Start unrestricted (high ε); analyze failure |
| **Gradient flow issues** | Med | Med | 1 | Switch to surrogate CNN; test gradients early |
| **Domain transfer fails** | Low | Low | 3 | Cut from paper; focus on source domain only |
| **Defenses too strong** | Low | Med | 3 | Adaptive attack; characterize tradeoffs |
| **Time overrun** | Med | Med | Any | Weekly checkpoints; cut scope to workshop (4pg) |
| **Advisor major revisions** | Med | Med | 5 | Build in 1-week buffer; target later deadline |

---

## 📈 SUCCESS METRICS

### Minimum Viable Publication (Workshop):
- [x] Reproduce baseline (~20cm)
- [x] Attack works (>1.5x error)
- [x] One defense tested (adversarial training)
- [x] 4-6 page paper
- [x] Venue: Workshop or arXiv

### Target Success (Conference):
- [x] All above +
- [x] Attack achieves >2x error
- [x] Comprehensive ablations (epsilon, patch size)
- [x] Multiple defenses (3+)
- [x] 6-8 page paper
- [x] Venue: IEEE WiSec, CNS, or AsiaCCS

### Stretch Goal (Top-Tier):
- [x] All above +
- [x] Attack achieves >3x error
- [x] Domain transfer analysis
- [x] Real hardware validation (optional)
- [x] Theoretical analysis
- [x] Venue: IEEE S&P workshops

---

## 📚 RESOURCE LINKS

**Paper 2 Resources**:
- GitHub: https://github.com/afbf4c8996f/Jammer-Loc
- Authors: h.habibi.fard@fu-berlin.de, g.wunder@fu-berlin.de

**Key Papers**:
- Paper 1 (a-RNA): arXiv:2211.01112
- Paper 2 (Jammer Loc): arXiv:2511.01819

**Tools**:
- LaTeX: Overleaf (IEEE template)
- Figures: Python (matplotlib, seaborn)
- Diagrams: draw.io or PowerPoint

**Venues** (check deadlines!):
- WiSec: https://wisec2026.github.io
- CNS: https://cns2026.ieee-cns.org
- AsiaCCS: https://asiaccs2026.github.io

---

## ✅ FINAL PRE-SUBMISSION CHECKLIST

### Content:
- [ ] Abstract: 150-200 words
- [ ] Introduction: ~1 page, clear contributions
- [ ] Related Work: 0.75-1 page, 20-30 refs
- [ ] Threat Model: 0.5 page, clear adversary model
- [ ] Methodology: ~2 pages, detailed attack description
- [ ] Evaluation: ~2 pages, all experiments
- [ ] Discussion: 0.5 page, implications & limitations
- [ ] Conclusion: 0.25 page, summary & future work
- [ ] Total: 6-8 pages

### Figures (6-8 total):
- [ ] Figure 1: Threat model diagram
- [ ] Figure 2: Position scatter plot
- [ ] Figure 3: Error CDF
- [ ] Figure 4: Epsilon ablation
- [ ] Figure 5: Patch size ablation
- [ ] Figure 6: Domain transfer
- [ ] Figure 7: Defense comparison
- [ ] Figure 8: Noise spectrum (optional)

### Tables (3-4 total):
- [ ] Table 1: Main attack results
- [ ] Table 2: Ablation summary
- [ ] Table 3: Domain transfer metrics
- [ ] Table 4: Defense evaluation

### Code:
- [ ] GitHub repository public
- [ ] README with installation & usage
- [ ] All code commented
- [ ] Demo notebook works
- [ ] Zenodo DOI added

### Submission:
- [ ] PDF compiles without errors
- [ ] All references formatted (IEEE style)
- [ ] No typos (proofread 3x)
- [ ] Submitted to conference portal
- [ ] Confirmation email received

---

**Total Estimated Time**: 150-180 hours over 8-10 weeks  
**Weekly Commitment**: 15-20 hours (flexible)  
**Start Date**: _________  
**Target Submission**: Day 42 (~6 weeks from start)  
**Expected Publication**: 2-4 months after submission

---

**Project Lead**: [YOUR NAME]  
**Supervisor**: [ADVISOR NAME]  
**Last Updated**: [DATE]

🚀 **LET'S BUILD SOMETHING GREAT!**