# *De novo* design of an IL-2Rβ and γc selective cytokine mimetic
An open-source reimplementation of the design logic behind Neo-2/15 (Silva et al., Nature 2019): build a protein with a completely novel fold that presents interleukin-2's binding surface for the IL-2Rβ and γc receptors, while possessing no binding surface for the IL-2Rα receptor at all. 


The pipeline has produced 2 candidates that survived filters. The more useful result is that roughly a third of the metrics conventionally used to evaluate such molecules failed against an explicit control, where several of them by scoring a randomly shuffled sequence, as well as or better than a validated molecule. The casualties include the metric I had pre-registered as primary, and the one metric that most flattered my own designs.

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








