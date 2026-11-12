# *De novo* design of an IL-2Rβ and γc selective cytokine mimetic
An open-source reimplementation of the design logic behind Neo-2/15 (Silva et al., Nature 2019): build a protein with a completely novel fold that presents interleukin-2's binding surface for the IL-2Rβ and γc receptors, while possessing no binding surface for the IL-2Rα receptor at all. 


The pipeline has produced a candidate that survived filters. The more useful result is that roughly a third of the metrics conventionally used to evaluate such molecules failed against an explicit control, where several of them by scoring a randomly shuffled sequence, as well as or better than a validated molecule. The casualties include the metric I had pre-registered as primary, and the one metric that most flattered my own designs.

### 1. The biological problem
Interleukin-2 is a small signalling protein that tells T cells to multiply. It does this by binding to receptors on the cell surface, and it has **two different receptor assemblies it can bind**:

| receptor assembly | affinity | expressed on | consequence of IL-2 signalling |
|---|---|---|---|
| IL-2Rα + IL-2Rβ + γc ("high affinity") | Kd ≈ 10 pM | constitutively high on regulatory T cells; induced on activated effector T cells and ILC2s | on Tregs, **suppressive**; the same receptor on an activated effector cell is activating |
| IL-2Rβ + γc ("intermediate affinity") | Kd ≈ 1 nM | NK cells, memory-phenotype CD8⁺ T cells | **activating** — the anti-tumour arm |

IL-2Rα (CD25) carries no signaling domain. It raises local affinity and presents IL-2 to the βγc pair, which transduces the signal. Because the trimer is roughly 100-fold tighter, administered IL-2 is captured preferentially by the CD25-high compartment at tolerable doses and is predominantly regulatory. Reaching the βγc-only cells requires a dose at which IL-2 is severely toxic.

The engineering goal is therefore a molecule that binds IL-2Rβ and γc, but not IL-2Rα. Neo-2/15 achieved this in 2019, not by mutating IL-2 but by designing an entirely new protein on an unrelated fold that reproduces IL-2's β/γc binding geometry and has no α site to remove. Sharing no sequence with IL-2 matters for a specific reason: an engineered IL-2 variant risks raising antibodies that cross-react with the patient's own IL-2, whereas a non-homologous mimetic cannot. Silva et al. reported undetectable immunogenicity in mice, and the clinical descendant NL-201, the first fully de novo protein taken into human trials. It demonstrated no significant immunogenicity in Phase 1 even after multiple cycles of therapy. Development was discontinued in November 2022 for strategic reasons rather than on safety or target engagement. This project reproduces the design approach using tools that are now freely available and do not require massive computational resources.

#### Why the premise had to be tested first
The whole approach only works if IL-2's two binding surfaces are physically separable: if the residues that grip β/γc are different residues from the ones that grip α. If they overlapped, there would be nothing to transplant. Testing this is Stage 1. For every residue of IL-2 in the crystal structure (PDB 2B5I), I measured how much of its surface is buried when each receptor binds: a standard quantity called ΔSASA, the change in solvent-accessible surface area. A residue that loses surface when β arrives is in contact with β.

| | IL-2 residues |
|---|---|
| visible in the crystal structure | 120 of 133 |
| contacting IL-2Rβ | 14 |
| contacting γc | 12 |
| contacting either β or γc (some contact both) | 24 |
| contacting IL-2Rα | 19 |
| **contacting both β/γc *and* α** | **0** |

**No IL-2 residue was detected doing both jobs**, and the two surfaces sit on different helices. That result, however, is conditional on an incomplete α structure: only 122 of IL-2Rα's 217 residues are resolved, including a 38-residue unresolved stretch whose flank sits 5.9 Å from IL-2. A proximity test bounds the exposure to a single motif residue (110), which was subsequently dropped for an unrelated reason, so the premise survives; it is not proven.

The premise holding, the motif was defined as three helical segments spanning 36 positions (A12–23, A84–95, A119–130), containing 21 measured β/γc contacts. Three further contacts (81, 110, 133) were isolated on loops with no helical neighbour and were dropped: freezing a single residue in isolation specifies a constraint a designed scaffold cannot reliably satisfy. That distinction between 36 transplanted positions and the 21 of them that actually touch a receptor drives the two design arms in Stage 3 and recurs throughout. Two deliberate choices here. The α threshold was set ten times stricter than the β/γc threshold (1 Å² against 10 Å²), because the two mistakes have different costs: missing a β/γc contact weakens the design, whereas missing an α contact silently builds the problem into the molecule. And only residues on well-formed helices were kept: a contact on a floppy loop carries geometric information that a new protein cannot reliably reproduce.

### 2. How the pipeline works
| notebook | tool | what it does | in → out |
|---|---|---|---|
| `01_Binding_Motif_Determination` | ΔSASA analysis | find which residues to transplant | 1 structure → 36 positions |
| `02_Backbone_Generation` | **RFdiffusion** | invent new protein shapes built around those fixed positions | → 152 shapes, 29 survive |
| `03_Sequence_Design` | **ProteinMPNN** | given a shape, choose amino acids that would fold into it | → 290 sequences |
| `04_ESMFold_Validation` | **ESMFold** | predict what each sequence actually folds into | → 115 self-consistent, 12 selected |
| `05_Boltz_2` | **Boltz-2** | predict each candidate *bound to its receptors* | 18 predictions |
| `Boltz_2_AF_crosscheck` | **AlphaFold2-multimer** | re-predict with a different architecture | 15 predictions |
| `Refinement_counterscreens` | Boltz-2 | repeat with different random seeds; test self-sticking | 4 seeds × 7 designs |
| `Candidate_Independence` | TM-align | are the survivors genuinely different from each other? | 12 → 6 for testing |
| `Biophysics` | freesasa | measure interface size, chemistry, packing defects | — |
| `MSAandsimilarity` | MAFFT, pydssp | sequence alignment against references and random controls | — |
| `3Dstructure` | freesasa, Biopython, py3Dmol | backbone-geometry QC, core packing and surface hydrophobicity, interface decomposition, the three pre-wetlab gates, structure figures | — |
| `Master_Table_Candidate_Filters` | — | join everything, apply selection gates | 7 → **2** |

The core trick in Stage 2 is called motif scaffolding: the chosen residues are held at their exact original coordinates while a generative model invents everything around them from noise. The result has a shape unrelated to IL-2 but presents the same binding surface.

The validation in Stage 4: design a sequence for a shape, then predict that sequence's structure with a model that never saw the shape. If the prediction comes back to the intended shape, the sequence specifies it. ESMFold was chosen specifically, because it needs no evolutionary data: these molecules have no relatives, so any method relying on related sequences would be working outside its training range.

<img width="2354" height="1320" alt="attrition_funnel" src="https://github.com/user-attachments/assets/0d0eccd9-1a95-430d-bc4d-1444e0669b6a" />

### 3. Walkthrough the pipeline
#### Stage 2: backbone generation
152 backbones generated across four variants, crossing two amounts of padding around the motif with two length budgets.
The motif transplanted perfectly. All 152 reproduced the frozen geometry to within 0.25 Å against a 1.0 Å threshold, so the filter never did any work, and that is reported as a null result rather than a 100% success rate. The second filter rejected reality, and catching that saved the stage. The filter asks whether the new shape intrudes into the space IL-2Rα occupies. I pre-registered 3.5 Å as the minimum clearance. No design passed and neither did native IL-2, which is co-crystallised with IL-2Rα and therefore guaranteed to fit alongside it. Native IL-2's closest approach is 2.70 Å.So the filter was measuring the wrong thing: at 3.5 Å it detects molecular contact, not physical overlap, and a protein that binds α obviously touches it. Recalibrating to the largest tested value the real complex passes gave 29 survivors from 152.

Reviewing that control later found a second version of the same mistake. The threshold came from a native structure with all its side chains, but RFdiffusion outputs only the backbone and no side chains. Measuring native IL-2 on backbone atoms alone gives 3.02 Å, so the gate was too lenient by 0.32 Å, which costs 12 of the 29. Both numbers are reported, and everything downstream uses the 29. Distributional conclusions survive the recalibration. The two candidates that eventually pass every gate split on the stricter threshold: the lead candidate's backbone clears it at 3.25 Å, the co-leader's fails at 2.58 Å. That is a second criterion separating the two, measured on bare backbone geometry before any sequence existed, and it points the opposite way to the geometric criterion. Neither was used in the selection.

A direction this filter may have had backwards. The filter removes designs whose scaffold occupies the volume IL-2Rα fills in the crystal. If a design engages βγc in IL-2's geometry, scaffold mass sitting in α's volume would sterically exclude α, which is the selectivity the project is trying to achieve, so filtering those designs out removes the one α-exclusion mechanism guaranteed to work. The argument the other way, and the one that motivated the filter, is that exclusion by occlusion is a property of superposing the design into the crystallographic frame and predicts nothing about whether α binds the designed surface in isolation. The ambiguity is recorded rather than resolved. In practice the filter acted mainly as a length filter.

A second asymmetry: clearance was measured against IL-2Rα only. Nothing at this stage tested, whether the scaffold also grows into IL-2Rβ or γc: the chains it must bind, because RFdiffusion was run on the motif alone with no target chain present. That is tested implicitly at Stage 5, where the designs clearing the target-engagement gate bury 614–688 Å² against IL-2Rβ and 422–475 Å² against γc, so the target volume is not occluded in the candidates carried forward. Generating with βγc present as a fixed target chain is the correct construction and is the main structural change for a second pass.

Size, not padding, predicts clearance. Pass rate falls from 71% below 95 residues to 4% above 115 (p < 0.001); padding has no detectable effect (p = 1.00 and p = 0.58). Surviving designs average 102 residues; Neo-2/15 is about 100.

#### Stage 3: amino acid decoration of generated backbones
Two variants were run on every shape: freeze all 36 transplanted positions, or freeze only the 21 that actually touch a receptor and let the rest be redesigned. 

Freezing less was better. Better sequence-structure agreement (1.196 vs 1.291, lower is better) and it solved a problem the pipeline had created.
I had instructed the model never to use cysteine because the case is found in the extracellular space. In the "freeze only contacts" arm: zero cysteines. In the "freeze everything" arm: all 145 sequences had one. The exclusion only applies to positions the model is choosing; frozen positions keep whatever was transplanted. IL-2 has a lone cysteine at position 125, it sits inside one of the frozen segments, and it was simply carried along, despite burying no receptor surface at all.

Released, the model replaced it with leucine in 108 of 145 cases and something hydrophobic in 98%. I checked the position independently: it is in the 98th percentile of local packing density with zero solvent exposure: a buried core position, where hydrophobic is correct. The clinically used form of IL-2, aldesleukin, also removes that cysteine. Instead, it uses serine, which is a manufacturing compromise rather than a good core residue.

The two arms are not a clean one-variable comparison for this reason: they differ in the number of frozen positions and in the presence of a free cysteine, and the Stage 4 results below cannot separate the two effects.

A control: A score of 1.196 is uninterpretable alone, so I ran the identical procedure on native IL-2 and Neo-2/15, and then, the important part, froze the same positions on native IL-2 itself.

| | score (lower is better) |
|---|---|
| Neo-2/15, nothing frozen | 0.896 |
| native IL-2, 21 frozen | 1.031 |
| native IL-2, 36 frozen | 1.053 |
| native IL-2, nothing frozen | 1.069 |
| **my shapes, nothing frozen** | **1.078** |
| my shapes, 21 frozen | 1.196 |
| my shapes, 36 frozen | 1.291 |

The cost of the transplant changes sign between the two scaffolds: freezing those positions on native IL-2 slightly improves its score, while freezing the same positions on a novel shape makes it substantially worse. Without the native control, an elevated score could have meant either bad shapes or an expensive transplant, with no way to tell them apart.

What the 0.009 gap does and does not say. My generated shapes score within 0.009 of native IL-2 when redesigned unconstrained, but this is ProteinMPNN scoring agreement under its own model, and Stage 4 shows that score carries no within-arm information about whether a sequence folds (correlations of −0.17 and −0.02, null χ² tests). So the correct reading is narrow: the backbones are as easy for an inverse-folding model to encode as a real cytokine. It is not evidence that they fold, and not a claim that scaffold quality is unlimiting.

#### Stage 4: do sequences fold?
115 of 290 sequences fold back into their intended shape, from 20 of 29 backbones. This is the first evidence that the sequences specify their intended fold under an independent predictor. It is not evidence that they fold in reality: ProteinMPNN and ESMFold are both trained on the PDB and both favour idealised helical geometry, so self-consistency here is necessary and not sufficient.

The two arms fail differently, not at different rates. Pass rates of 65 vs 50 out of 145 (p = 0.093, not significant). But catastrophic failures, where the prediction comes back unrelated to the target shape, are 4 versus 20 (odds ratio 5.6, p < 0.001). Median agreement differs by 0.07 Å between arms and conceals this entirely. The informative part of a distribution running from 0.4 Å to 15.5 Å is the tail.

Two qualifications on that comparison. The arms differ in the free-cysteine covariate as well as in the number of frozen positions, so it does not isolate the mechanism. And the 290 sequences are 29 backbones × 5 sequences × 2 arms: clustered, and paired across arms, so a test treating them as independent reports an optimistic p-value.
The three-filter funnel turned out to be one filter. Of 118 designs clearing the confidence threshold, 115 clear everything. The other two filters remove three designs between them. One gate does 97% of the work, which is not what was expected when three were specified.

#### Candidate independence: are those 12 designs?
No. It is one fold, sampled twelve times.

TM-score measures structural similarity; above 0.5 means the same fold. All 66 pairs of candidates exceed it, and the median is 0.892 — near-identical structure, not merely the same topology. At the sequence level the median pairwise identity is 53.6% (range 42.9–69.5%), of which roughly 21 points is the frozen motif itself. No clustering was performed, so the number of distinct sequence groups is not quantified.

Running this before the complex predictions is what stopped twelve near-copies being reported as twelve independent designs, and saved GPU time predicting duplicates. It also bounds every later claim: this project describes one topology, not a survey of solutions. The diversity selection that followed confirms it: picking the six most dissimilar candidates lowered maximum pairwise TM from 0.979 to 0.921 and left the median unchanged (0.891 against 0.892).

One incidental finding: the designs are structurally closer to Neo-2/15 (0.686) than Neo-2/15 is to IL-2 (0.602), despite never having been compared to Neo-2/15 during design. Two explanations are available and this project does not separate them. The first is shared idealisation: generative models and Rosetta both produce regular, idealised helical bundles, while evolved proteins are irregular. The second is connectivity: my designs run H1 → H3 → H4 with the intervening helix absent, and Neo-2/15's published topology is H1-H3-H2'-H4, so both place the three binding helices in an order native IL-2 does not. The second explanation is the more parsimonious one and was not tested.

#### Stage 5: Predicting the complexes
Each candidate predicted twice: once with IL-2Rβ and γc, once with IL-2Rα. Three controls run identically: native IL-2, Neo-2/15, and a randomly shuffled version of one of my own designs, a sequence that cannot fold into anything. This is where the pre-registered primary metric died. The replacement metric: interface error decomposed per chain pair - separates the controls correctly, and five of six designs are predicted to engage the target receptors more confidently than Neo-2/15. The results explain why that should be read with suspicion rather than satisfaction, and records that the floor this gate stands on is weaker than it looks.

An internal check worth noting: IL-2Rβ and γc touch each other in every prediction whatever is bound to them, so their mutual interface error should be constant. It is: 7.98 ± 0.51 across nine predictions, confirming the decomposition measures what it claims to.

#### Interfaces, replication, developability
A single-residue fix, found by comparing siblings. One candidate had three buried polar atoms with no hydrogen-bonding partner which is a packing defect. Comparing the five sequences ProteinMPNN produced for that same shape showed they differ at position 22: three have tyrosine, one leucine, one arginine. The arginine variant was my candidate, and its arginine was the defect. Taking the tyrosine sibling instead reduced the count from three to one on the ESMFold models (three to two when measured on the predicted complexes: the two structure sources do not agree exactly, and both are reported). Same geometry, one residue different: a controlled comparison.

Replication showed the α counter-screen is worse than noisy. Repeating the α prediction across four random seeds: confidence varied enormously (one design ranged 0.24 to 0.77 — wider than the entire gap between the positive and negative controls). But the geometric measurement is more damning. Buried interface area ranged from 769 to 1202 Å², and the identity of the contacting residues overlapped by only 42% between seeds. Two predictions of the same complex share fewer than half their interface residues. The model is not uncertain about how good the interface is and it is building a different interface each time.

The self-sticking screen fails its own control. Each sequence was predicted against a copy of itself, since a protein that prefers its own surface to the receptor's is unusable. Native IL-2, which is a monomer in solution, scores as the most strongly self-associating molecule in the panel (0.869), a false positive on the one molecule whose answer is known. The likely cause is an MSA asymmetry: native IL-2 was the only ligand given a server alignment, and alignment-driven prediction of cytokine homodimers is a known failure mode. Neo-2/15, predicted the same way as the designs, gives the correct answer (0.336) and is used as the reference instead. The threshold derived that way is the only one in the project not anchored on a control, and is labelled post-hoc wherever it appears.

Interface composition, including the control. Designs bury 495–688 Å² against IL-2Rβ and 422–475 Å² against γc, against native IL-2's 665 and 455 and Neo-2/15's 742 and 489. The shuffled control buries 773 Å² against IL-2Rβ, more than every design and more than native IL-2, at a comparable nonpolar fraction of 0.600. A sequence that cannot fold is given the largest β interface in the panel, which is a fifth metric failing its negative control and the clearest single indication that buried area carries no binding information here. Nonpolar fraction at that interface is 0.59–0.61 for designs, 0.58 for native IL-2 and 0.56 for Neo-2/15, so the designed interfaces are marginally greasier than either reference, which is the idealisation pattern again.

Developability does not discriminate. Native IL-2 carries three lone cysteines and three unsatisfied buried polar atoms. Neo-2/15's computed instability index is worse than five of seven designs. Both molecules work. These metrics describe whether something can be manufactured, not whether it functions, and are reported without gating.

#### Cross-check
Boltz-2 and AlphaFold2-multimer are built differently with different internal architectures, different handling of evolutionary data, different training vintages — so they can fail differently. Three complexes were re-predicted with AlphaFold. Both models rank the lead candidate above both references on target-interface confidence, and under AlphaFold the lead candidate reproduces the native receptor arrangement more closely than Neo-2/15 does: 1.04 Å against 1.77 Å, where AlphaFold disagrees with itself about native IL-2 by 0.73 Å.

The first of those is not independent corroboration. Native IL-2 < Neo-2/15 < my design is exactly the idealisation ordering documented in §4, so two tools agreeing is at least as consistent with a bias they share as with better target engagement. It is reported as an observation.

The second does not survive the cross-check either, and that is the point of running one. The same measurement under Boltz-2 inverts the ordering — Neo-2/15 1.50 Å, design 37 2.07 Å — against a gate pre-registered at 2.0 Å, which the lead candidate therefore fails and the co-leader (1.48 Å) passes. The AlphaFold values are five non-independent networks, the Boltz value is a single seed. Neither carries an error model that resolves the disagreement, so the geometry claim is tool-dependent rather than established, and it is not carried into the selection in Results.

This is a partial check in two further ways: both models train on the same database of experimental structures and share blind spots wherever those are scarce: de novo complexes being exactly such a gap and they disagree with each other by 1.5–2.0 Å about a receptor pair both have almost certainly seen in training, which is notable rather than reassuring.

#### Alignment nulls: is the resemblance to Neo-2/15 meaningful?
Aligning the designs against Neo-2/15 with the transplanted motif masked out — so only freely designed positions are compared — and against two unrelated human helical proteins as controls:

| reference | observed identity | shuffled null | z |
|---|---|---|---|
| Neo-2/15 | 18.3% | 9.7 ± 3.8% | +2.2 |
| saposin C | 14.9% | 8.3 ± 4.4% | +1.5 |
| apolipoprotein E, N-terminal domain | 19.8% | 9.1 ± 3.3% | **+3.2** |

The designs score no closer to the molecule they were modelled on than to an unrelated human helical bundle. Whatever similarity exists is generic helical-bundle character, not functional convergence. (The three z-scores are not separable from one another at n = 6 with overlapping nulls; the claim is the absence of a Neo-2/15-specific signal, not a ranking.)

### 4. The metric problem 

Twelve metrics were discarded, including the one pre-registered as primary. Ten of them failed against an explicit control, never because they gave an inconvenient answer about a design. The remaining two: pDockQ and ipSAE, are reported as untested rather than failed: ColabFold emits them as direction-dependent per-chain-pair values and they were never reduced to comparable scalars, so no control was run on them.

#### How controls were designed
Every selectivity metric was evaluated against three reference molecules at once:
1. Native IL-2 — positive control. It binds both receptor assemblies, so a valid selectivity metric must score it as non-selective.
2. Neo-2/15 — the reference for what a genuinely selective molecule looks like.
3. A composition-matched shuffled sequence — the floor. If a metric cannot separate a real protein from its own amino acids in random order, it cannot rank designs.

Plus an internal invariant (a quantity that should not change between conditions, as a check on the measurement itself), seed replication to measure randomness within one model, and a second architecture to measure bias across models.

#### What controls caught 
The pre-registered primary metric. ipTM is a standard confidence score for predicted complexes. My metric was its difference between the two panels — high confidence on the target, low on the counter-target, should mean a selective molecule.

| | confidence, βγc | confidence, α | **difference** |
|---|---|---|---|
| native IL-2 | 0.908 | 0.884 | 0.024 |
| Neo-2/15 | 0.891 | 0.761 | 0.130 |
| **shuffled sequence** | **0.870** | 0.706 | **0.164** |

A sequence that cannot fold is predicted against the receptor assembly at 0.870, against real IL-2's 0.908, and scores as more selective than the validated reference drug candidate. The metric reads the positive control correctly and fails on the negative one. Discarded.

The replacement metric's difference form fails the same way. Interface error gave Neo-2/15 7.66 and shuffled 7.57 are indistinguishable. Only the raw value on the target interface worked (IL-2 6.69, Neo-2/15 6.65, shuffled 8.97). A selectivity metric built as the difference between two confidence scores inherits the uninformativeness of both, because both are high for almost any two chains a model can pack together.

Buried surface area, same failure. The shuffled control buries 773 Å² against IL-2Rβ against native IL-2's 665 (§3). Interface size does not separate a protein from a sequence that cannot fold, so it is reported descriptively and gates nothing.

Confirmed independently in a second architecture. IL-2Rβ and γc touch each other in every complex, whatever is bound. Across 15 AlphaFold predictions spanning three unrelated molecules, the confidence for that receptor pair is 0.8601 ± 0.0052: a variation of 0.6%. The score is blind to what is bound to it. This is the most robust number in the project and it is a negative result about a metric.

Most of the published confidence panel cannot discriminate at all. Of five such metrics tested, one accounts for 90% of the between-molecule variance and four do not. One of them, actifptm, returns 0.981, 0.980 and 0.980 for a natural cytokine, a clinical-stage design and an untested design.

**A sequence-only metric cannot beat shuffled sequence.** ESM-2 pseudo-log-likelihood scores how "natural" a sequence looks. Against a shuffled floor of −2.68 ± 0.13: Neo-2/15 scores −2.76, below the mean, and its structure is gap-free so the score is clean. Native human IL-2 scores −2.83 from the crystal sequence and −2.57 from the full UniProt entry — the crystal sequence has unresolved loops spliced out, which depresses the score, so the splicing artefact explains part of the deficit but not the result. (The UniProt entry is the 153-residue precursor including its signal peptide, so that value is an upper bound rather than a like-for-like comparison.) At best IL-2 sits 0.9 SD above a floor made of shuffled sequence. My untested designs score −2.07 to −2.24, three to five standard deviations above it. The mechanism is visible in the references: the large, well-studied receptor chains score well (γc −1.50) while IL-2, a short cytokine with few close relatives, scores badly. The metric measures how typical a sequence is of its training data, not whether it folds. Designed sequences are more generic than real ones, so it rewards them.

Two caveats on that comparison, in the same spirit as the rest of this section: the shuffled floor is composition-matched to my designs and not to IL-2 or Neo-2/15, and the reference values for the receptor chains are full-length entries including transmembrane and cytoplasmic regions. Both weaken the quantitative anchors without changing the direction.

A correlation that looked like validation was circular. That same metric does correlate with ESMFold's confidence (Spearman +0.45, p = 10⁻¹⁵). But ESMFold is built on ESM-2: its sequence encoder is an ESM-2 model. Two metrics from the same model family agreeing is not a cross-check. This is why the AlphaFold comparison was worth doing: to get an independent check you have to leave the family.

A geometric filter rejected the crystal structure. Covered in §3: the pre-registered α clearance threshold excluded the real IL-2/IL-2Rα complex.

A measurement whose answer changes between runs. Covered in §3: the α interface moves between seeds, with 42% residue overlap.

A screen that fails on a known monomer. Covered in §3: native IL-2 scores as the strongest self-associator in the panel.

The same inversion appeared on unrelated metrics from different model families:
| metric | ordering |
|---|---|
| ProteinMPNN sequence-structure agreement | Neo-2/15 better than native IL-2 |
| ESMFold confidence (pLDDT) | IL-2 < Neo-2/15 < designs |
| Boltz-2 interface error | designs better than Neo-2/15 and IL-2 |
| AlphaFold target-interface confidence | IL-2 < Neo-2/15 < my design |

Native protein worst, published de novo design in the middle, my untested design best — every time. That is the order of increasing structural idealisation, not of demonstrated function. Generative models produce regular, textbook-perfect geometry, evolved proteins are irregular, and prediction models are more confident about the regular ones.

Every confidence-based result in this project therefore carries a bias in the direction that flatters the designs, including the favourable ones. And the inference follows: when a metric ranks an untested design above a clinical-stage molecule, the correct conclusion is that the metric is invalid for that comparison, not that the design is better.

### 5. What the candidates are, and are not
Two designs pass all three selection gates: pad0_len100_37_fixContacts_2 and pad0_len100_2_fixContacts_3. No single winner is defensible, which is consistent with the ranking having changed at every one of five evaluation layers.

<img width="3453" height="1317" alt="candidate_comparison" src="https://github.com/user-attachments/assets/36a560cb-0edc-4d40-9e2c-0653d53c6ea3" />

The three gates are not of equal standing:

1. Target engagement must beat the shuffled sequence. Anchored on a control, but on a control that does not fold, so it is a protein-ness floor rather than a binding floor.
2. α rejection must beat Neo-2/15's values across at least three of four seeds. Anchored on a control, but applied to a measurement §3 shows is not reproducible between runs, and against a Neo-2/15 reference value that is itself single-seed while the designs are judged on four-seed distributions. This is triage, not evidence of the same quality as the first gate.
3. Self-association is post-hoc, for the reason given in §3, and labelled as such everywhere.

Two further criteria exist, were computed, and are not in the gate count, because they point in opposite directions. The corrected backbone-only α clearance from Stage 2 favours 37_fixContacts_2 (3.25 Å against the co-leader's 2.58 Å, measured before any sequence existed). The pre-registered receptor-orientation gate from the pre-wetlab checks favours 2_fixContacts_3 (1.48 Å against 2.07 Å, against a 2.0 Å threshold the lead therefore fails), and that measurement is single-seed and inverted by the other predictor (§3). Each is a single unreplicated number, they disagree, and neither is strong enough to break the tie. They are reported here rather than used, and they are the clearest statement of why the tie should not be broken on this evidence.

On the ungated developability measures, 37_fixContacts_2 is the stronger of the two: one unsatisfied buried polar atom against three, the lowest computed instability index in the panel including both references, no lone cysteine, and a net charge well away from the value at which proteins aggregate. That is why it is treated as the lead candidate. It is a weaker basis than a gate for two reasons: no control-anchored threshold exists for any developability metric, and the unsatisfied-polar counts are computed on predicted side-chain positions at a 3.5 Å cutoff: single-digit integers that shift by up to three when the cutoff moves to 3.2 Å.

#### What they are
A 111-residue protein of novel fold, carrying IL-2's β/γc motif across two measured hops: the generated backbone reproduces the native motif geometry to within 0.25 Å, and the independent structure prediction reproduces that backbone's motif to 0.48 Å, which folds into its intended shape under a predictor that never saw the backbone, and which two architecturally distinct models place confidently against the target receptors. The end-to-end motif RMSD against the crystal structure was not computed, the two hops are reported separately rather than composed.

Two measurements made on the receptor side rather than the design side support the same reading. The lead candidate buries 87.5% of the IL-2Rβ residues and 76.9% of the γc residues that native IL-2 buries in the equivalent prediction (Neo-2/15: 93.8% and 100%), so it engages the receptors at the epitope IL-2 uses rather than somewhere else on the same chains. And its bound conformation differs from its free one by 0.53 Å, against a field convention of 1.5 Å, so it does not have to rearrange to bind. Both are computed against a predicted native complex rather than the crystal, and both are single-seed.

#### What they are not: better than Neo-2/15
The metrics favouring them are confidence scores subject to the bias in §4, and the ordering that favours them is that bias's signature. Under AlphaFold my candidate deviates less from the native receptor geometry than Neo-2/15 does, under Boltz-2 the reverse. Either way Neo-2/15 controls tumours in mice, so geometric fidelity demonstrably is not what predicts efficacy. Neo-2/15 has experimental data and these have none.

#### What they are not: selective
Five of six designs contact IL-2Rα more than native IL-2 does, and the lead candidate contacts it most in the panel: 46 contacts against Neo-2/15's 14.

Why that is not conclusive is itself a result. Comparing each molecule's β/γc-contacting residues against its α-contacting residues:

| | shared residues | overlap (Jaccard) | % of α surface shared with βγc surface |
|---|---|---|---|
| native IL-2 | **0** | **0%** | **0%** |
| Neo-2/15 | 19 | 54.3% | 79.2% |
| shuffled sequence | 21 | 58.3% | 91.3% |
| designs (n = 7) | 16–21 | 34.8–58.3% | 44.4–77.8% |

Native IL-2's two receptor surfaces are entirely separate: zero shared residues, independently reproducing the Stage 1 premise from two separate predicted complexes rather than from the crystal. Every de novo molecule has essentially one interface-like surface instead, including Neo-2/15, which has the property more strongly than six of my seven designs and is nonetheless experimentally selective.

Three conclusions. This is a further metric that fails its negative control: it cannot separate a validated selective molecule (54.3%) from a sequence that cannot fold (58.3%). The α counter-screen is answering the wrong question: for native IL-2, "does it bind α" concerns a spatially distinct site; for a de novo mimetic, α and the target receptors compete for the same surface. And the reason is structural rather than incidental: the separability the design objective assumes is a property of IL-2's 133-residue irregular fold, and a ~100-residue idealised bundle does not have the surface to keep two epitopes apart. Grafting β/γc contacts while excluding α contacts cannot deliver α-selectivity by construction at this scaffold size. Selectivity there is an affinity competition on one surface, which no confidence score can evaluate. It requires a measured binding constant.

Two further notes. The design with the lowest overlap has the worst target engagement in the panel, worse than the shuffled control, so low overlap tracks prediction quality rather than selectivity. And that same design (15_fixAll_4) is the one piece of evidence that the target-engagement gate is not purely a foldedness detector: it clears self-consistency at pLDDT 92.5 and scRMSD 0.62, so it folds, and still scores 11.10 against the shuffled control's 8.97. That is n = 1 and not a substitute for the control described in §6.

#### What they are not: characterised energetically
Every metric here is a predictive confidence or a buried surface area. None is a binding free energy. No affinity, no selectivity ratio, no melting temperature.

#### What this project establishes, suggests, and cannot address
| | |
|---|---|
| **Establishes** | The geometric premise holds for native IL-2. The pipeline runs end to end and produces self-consistent molecules of novel fold. Ten conventional evaluation metrics, including the pre-registered primary, fail against controls that are shown. Confidence metrics are systematically biased toward idealised geometry. The α counter-screen is ill-posed for de novo molecules. |
| **Suggests** | Scaffold length constrains α clearance. Freezing only measured contacts outperforms freezing whole segments. Two candidates are worth carrying forward over the other five. |
| **Cannot address** | Whether any design binds anything. Whether any is selective. Stability, expression, solubility, immunogenicity. All require experiment. |

### 6. Limitations

Some limitations were discovered by testing and are reported above as results: the metric failures in §4, the non-reproducible α interface, the self-association control failure, the one-topology finding. Some are inherited from the method and the compute budget and bound the claims without invalidating them. Some are fatal to a specific claim, and each is stated with the claim it rules out. Nothing below was found by a reviewer: all of it came out of the controls, which is the point.

**No folded negative control:** The target-engagement gate is anchored on a composition-matched shuffled sequence, which does not fold. It therefore differs from a design in two ways at once: no epitope and no structure, so clearing it establishes that a design docks better than a random coil, not that it binds. The control needed is a folded chain with no binding surface: design 37 with the 21 motif contacts mutated out, verified to still fold. This matters more than it would elsewhere because structure predictors are trained on the PDB, which contains essentially no examples of proteins failing to interact, so their prior for any two chains is that an interface exists. The shuffled control nonetheless burying 773 Å² against IL-2Rβ is consistent with exactly that. Kills the claim that target engagement is specific.

**The candidate set descends from a mis-calibrated filter:** The corrected backbone only α clearance threshold (3.02 Å, §3) excludes 12 of the 29 survivors, and everything downstream uses the 29. Distributional conclusions survive: the length-dependence finding is measured through that same gate, and of the two candidates that eventually pass every gate, one clears the stricter threshold (3.25 Å) and one does not (2.58 Å). Kills the claim that the 29 are "the α-compatible backbones" rather than 29 backbones.

**The α clearance filter may have the wrong sign:** removing designs that occupy α's crystallographic volume removes the designs most likely to exclude α sterically. The filter was never justified in the direction it was applied, and the βγc equivalent: the clash that would actually matter and it was never computed at this stage at all. Kills the claim that the filter contributed selectivity rather than a size constraint.

**The α gate thresholds a non-reproducible measurement against an unreplicated reference.** Requiring three of four seeds quantifies the noise documented rather than removing it, and the Neo-2/15 anchor is single-seed while the designs are four-seed. Kills any ranking of designs below the top two.

**The shapes cannot be regenerated:** No random seed was set in Stage 2, so the 152 backbones cannot be reproduced from the code. The files are in this repository, so the work is verifiable but not reproducible: different claims, and the more consequential limitation here. Distributional findings would replicate on a re-run; no statement about a specific named design would.

**One topology, not six candidates:** Median pairwise structural similarity of 0.892.

**The two design arms are confounded:** They differ in the number of frozen positions and in the presence of a free cysteine simultaneously, so the five-fold difference in catastrophic folding failures cannot be attributed to either alone.

**Target engagement is single-seed** only the α screen was replicated. So the unfavourable measurement has four seeds and the favourable one has one. This asymmetry runs in the project's favour and has no justification beyond compute budget. The pre-wetlab geometry gates inherit it.

**Only the design that already scored best was cross-checked**, after the Boltz results were known, so the agreement reported there is a confirmation rather than a discrimination test. Cross-checking a design the panel rejected would have been the stronger experiment.

**The complex prediction used the variant shown not to work:** Bennett et al. 2023 benchmarked in-silico filters against thousands of experimentally tested designs and found that ordinary complex prediction has little power to separate binders from non-binders, while AlphaFold2 with "initial guess" does, while seeding the model with the designed coordinates and asking whether it keeps that pose. Everything in the complex-prediction and cross-check stages uses the ordinary variant. My own shuffled control behaved exactly as that paper predicts it would, which I take as the strongest evidence that the diagnosis in §4 is correct.

**The α structure used to define the motif is incomplete:** 122 of IL-2Rα's 217 residues are resolved, including a 38-residue unresolved stretch near the interface.

**No energy function:** No binding energy, no shape complementarity score, no count of unsatisfied buried polar atoms computed on relaxed coordinates. PyRosetta would have supplied all three and was not used.

**No structures were energy-minimised before measuring contacts:** The contact analysis finds clashes in every design and none in the references, which is an artefact of unrelaxed predicted coordinates. Diagnosed rather than removed. The buried-unsatisfied-polar counts used in 5 inherit this: they are computed on predicted side chains and shift with the hydrogen-bond cutoff.

**Sequences were designed on the isolated protein:** ProteinMPNN accepts the target as context and the field standard for binders is to use it, so that positions at the edge of the interface know what they bind. Between 55 and 94 freely designed positions per design here did not.

**The cross-check is partial:** Both models train on the same structural database, the α panel was never cross-checked at all, so the half of the claim that works has a second architecture behind it and the half that doesn't has none.

**Per-shape designability was never measured.** Five sequences per shape at low sampling temperature are expected to be close variants: pairwise identity within a backbone group was not computed, and one was carried forward. Standard practice uses 8–48 and treats the fraction that fold correctly as the signal.

### 7. Environment and provenance
L4 and A100 GPUs. Every stage records its own environment.

Backbone generation pins the RFdiffusion commit and verifies the model weights by SHA256. What that does and does not guarantee, stated plainly: the digest was recorded on first download, so it guarantees every later run gets the same bytes, and cannot attest that the first download was authentic: no independently published digest exists for that file. It was fetched over HTTPS with an explicitly flagged plaintext fallback, and the transport actually used is recorded.

Complex prediction could not use the package's own dependency versions, because Boltz 2.2.1 declares a Python ceiling that Colab exceeds: a limit arising from its pinned compiled dependencies, not from Boltz itself. Dependencies were installed unversioned and Boltz with --no-deps, so "Boltz 2.2.1" does not specify this run. pip_freeze.txt does.

Boltz-2's weights file embeds its training configuration as a Python object, which recent versions of PyTorch refuse to load because such files can execute arbitrary code. This was resolved by allowing only the specific configuration classes, not by disabling the check, which would have permitted arbitrary code execution from a downloaded file.

The third-party notebook used for the AlphaFold cross-check was not granted Google Drive access; results were downloaded manually and extracted locally, so third-party code never ran against this project's files.

### 8. References
1. Silva, Yu, Ulge et al. De novo design of potent and selective mimics of IL-2 and IL-15. Nature 565, 186–191 (2019) — PDB 6DG5, 6DG6
2. Wang, Rickert, Garcia. Structure of the quaternary complex of interleukin-2 with its α, β, and γc receptors. Science 310, 1159–1163 (2005) — PDB 2B5I
3. Bennett et al. Improving de novo protein binder design with deep learning. Nat Commun 14, 2625 (2023)
4. Watson et al. De novo design of protein structure and function with RFdiffusion. Nature 620, 1089–1100 (2023)
5. Dauparas et al. Robust deep learning-based protein sequence design using ProteinMPNN. Science 378, 49–56 (2022)
6. Lin et al. Evolutionary-scale prediction of atomic-level protein structure with a language model. Science 379, 1123–1130 (2023) — ESMFold, ESM-2
7. Wohlwend et al. Boltz-1 / Boltz-2. MIT (2024, 2025)
8. Mirdita et al. ColabFold: making protein folding accessible to all. Nat Methods 19, 679–682 (2022)
