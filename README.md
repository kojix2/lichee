
# LICHeE: Fast and scalable inference of multi-sample cancer lineages

## Changes in this fork

- Removed screen resolution parsing when generating DOT files, which required an X11 server.
- `release/lichee.jar` is treated as a generated artifact and is not tracked in git. Build it before running (see [How to Run](#how-to-run)).

### About

LICHeE is a combinatorial method for reconstructing multi-sample cell lineage trees and inferring the subclonal composition of tumor samples based on variant allele frequencies (VAFs) of deep-sequencing somatic single nucleotide variants (SSNVs). It accepts a list of SNVs with per-sample VAFs and outputs the inferred cell lineage tree(s) and sample subclone decomposition. An optional GUI allows interactive exploration of the results.

At a high level, LICHeE proceeds through the following steps:

1. SSNV calling across input samples
2. SSNV clustering by VAF (each group of SSNVs present in the same set of samples is clustered separately)
3. Construction of the evolutionary constraint network (nodes = clusters, edges = valid pairwise ancestry relationships)
4. Search for lineage trees satisfying all phylogenetic constraints
5. Output visualization

For more information about the algorithm please see the following publication:  
Popic V, Salari R, Hajirasouliha I, Kashef-Haghighi D, West RB, Batzoglou S.  
*Fast and scalable inference of multi-sample cancer lineages*. Genome Biology 2015, 16:91.

### Program Parameters

Default parameter values are set conservatively for noisy real data. For simulated or cleaner data, relaxing these thresholds can produce more granular results. For example:

- Lowering `-maxClusterDist` keeps nearby clusters separate, ordering additional SSNVs.
- Lowering `-minClusterSize` to 1 retains single-SSNV clusters in the network.

See [Parameter Tuning and Diagnostics](#parameter-tuning-and-diagnostics) for details.

##### Commands

| Option | Description |
|--------|-------------|
| `-build` | Reconstruct lineage trees |

##### Input/Output and Display Options

| Option | Description |
|--------|-------------|
| `-i <arg>` | Input file path (**required**) |
| `-o <arg>` | Output file path (default: input filename with `.trees.txt` suffix) |
| `-cp` | Input data represents cell prevalence (CP) values instead of VAF |
| `-sampleProfile` | Input file contains pre-computed SSNV presence-absence profile (disables default calling) |
| `-n, --normal <arg>` | Normal sample column index, 0-based (e.g. 0 for the first column) (**required**\*) |
| `-clustersFile <arg>` | Pre-computed SSNV clusters file path |
| `-s, --save <arg>` | Maximum number of output trees to save (default: 1) |
| `-showNetwork, --net` | Display the constraint network |
| `-showTree, --tree <arg>` | Number of top-ranking lineage trees to display (default: 0) |
| `-color` | Enable color mode in tree visualization |
| `-dot` | Export top-scoring tree as a DOT file (default path: input file with `.dot` suffix) |
| `-dotFile <arg>` | Custom DOT file path |

##### SSNV Filtering and Calling

| Option | Description |
|--------|-------------|
| `-maxVAFAbsent, --absent <arg>` | Max VAF to robustly call an SSNV absent from a sample (**required**\*) |
| `-minVAFPresent, --present <arg>` | Min VAF to robustly call an SSNV present in a sample (**required**\*) |
| `-maxVAFValid <arg>` | Maximum allowed VAF in a sample (default: 0.6) |
| `-minProfileSupport <arg>` | Min number of robust\*\* SSNVs for a group profile to be labeled robust; SSNVs from non-robust groups may be re-assigned (default: 2) |

\* Required unless `-sampleProfile` is specified.  
\*\* Robust SSNVs have VAF < `maxVAFAbsent` or > `minVAFPresent` across all samples.

##### Phylogenetic Network Construction and Tree Search

| Option | Description |
|--------|-------------|
| `-minClusterSize <arg>` | Min SSNVs per cluster (default: 2) |
| `-minPrivateClusterSize <arg>` | Min SSNVs for a private cluster (SSNVs in one sample only) (default: 1) |
| `-minRobustNodeSupport <arg>` | Min robust SSNVs for a node to be non-removable during tree search (default: 2) |
| `-maxClusterDist <arg>` | Max mean VAF difference per sample for collapsing two clusters (default: 0.2) |
| `-c, --completeNetwork` | Add all possible edges to the constraint network; by default, private nodes are connected only to the closest level parents, and only nodes with no other parents are descendants of root |
| `-e <arg>` | VAF error margin (default: 0.1) |
| `-nTreeQPCheck <arg>` | Number of top trees on which the QP consistency check is run; this check has not been observed to fail in practice (default: 0) |

##### Other

| Option | Description |
|--------|-------------|
| `-v, --verbose` | Verbose mode |
| `-h, --help` | Print usage information |

### How to Run

1. Build from source (from `LICHeE/`):

   ```sh
   make lichee.jar
   ```

2. Run from `LICHeE/release/`:

   ```sh
   ./lichee -build -i <input_file_path> -minVAFPresent <VAF1> -maxVAFAbsent <VAF2> -n <normal_sample_id> [other options]
   ```

### Examples

For additional command-line settings used on the ccRCC and HGSC datasets, see `data/README`.

Show the top-ranking tree:
```sh
./lichee -build -i ../data/ccRCC/RK26.txt -maxVAFAbsent 0.005 -minVAFPresent 0.005 -n 0 -showTree 1
```

Filter private clusters with fewer than 2 SSNVs, and save the top-ranking tree:
```sh
./lichee -build -i ../data/ccRCC/RMH008.txt -maxVAFAbsent 0.005 -minVAFPresent 0.005 -n 0 -minPrivateClusterSize 2 -showTree 1 -s 1
```

Tighten the VAF cluster centroid distance threshold:
```sh
./lichee -build -i ../data/hgsc/case6.txt -maxVAFAbsent 0.005 -minVAFPresent 0.01 -n 0 -maxClusterDist 0.1 -showTree 1
```

### Input File Types

LICHeE accepts three input formats.

#### Format 1: VAF/CP values per sample

One SSNV per line. Tab-separated header:
```
#chr position description <sample names separated by tabs> 
```

For example (the following file contains 5 samples and 3 SSNVs):
```
#chr    position    description    Normal    S1    S2    S3    S4                     
17      123456      A/T DUSP19     0.0       0.1   0.2   0.25  0.15                   
11      341567      C/G MUC16      0.0       0.4   0.09  0.38  0.24                   
9       787834      A/C OR2A14     0.0       0.35  0.14  0.17  0.48                   
```

#### Format 2: Pre-computed sample calls (with `-sampleProfile` flag)

Add a `profile` column before the frequency columns, specifying the binary presence–absence pattern across samples. For example, `01001` means the SSNV was called in samples 1 and 4 (0-based). Use the `-sampleProfile` flag to enable this mode and disable the default calling step.
```
#chr    position    description    profile        Normal    S1    S2    S3    S4      
1       184306474   A/G HMCN1      01111          0.0       0.1   0.2   0.25  0.15    
1       18534005    C/A IGSF21     01111          0.0       0.1   0.25  0.2   0.1     
1       110456920   G/A UBL4B      01111          0.0       0.4   0.4   0.45  0.45    
10      26503064    C/G MYO3A      01001          0.0       0.4   0.0   0.0   0.24    
```

#### Format 3: Pre-computed SSNV clusters (with `-clustersFile` flag)

Provide a separate file with one cluster per line. Fields are tab-separated and correspond to the primary SSNV input file:

```
profile   <cluster VAFs per sample separated by tabs> <comma-separated list of SSNVs> 
```

For example (the following file contains 3 clusters for the SSNV example file shown above; the SSNVs are specified as line numbers in the SSNV input file ignoring the header line, starting from 1):

```
01111     0.0   0.1   0.23  0.23  0.13    1,2                                         
01111     0.0   0.4   0.4   0.45  0.45    3                                           
01001     0.0   0.4   0.0   0.0   0.24    4                                           
```

### Output Visualization

Results can be saved to a text file using `-s <N>` (saves up to N top trees; it is recommended to review all trees with the best score). They can also be visualized in the interactive GUI (`-showTree`) or exported as a DOT file for Graphviz (`-dot` / `-dotFile`).

The GUI supports:
- Dynamically removing nodes or collapsing clusters of the same SSNV group.
- Clicking any node to inspect its contents (SSNV composition or subclone decomposition).
- The Snapshot button to export the current view as a PDF vector graphic (note: writing the file may take a moment).
- Dragging nodes to reposition them; zoom and pan with the trackpad.

Two display modes are available:
- Plain (default): Simple node-edge layout.
- Color (`-color`): Each cluster node gets a unique color, and sample nodes are decorated with the colors of their contributing clusters. Selecting a cluster node highlights its contribution to each sample in purple.

**Example 1: ccRCC patient RK26**

```sh
./lichee -build -i ../data/ccRCC/RK26.txt -maxVAFAbsent 0.005 -minVAFPresent 0.005 -n 0 -showTree 1 -color -dot
```

Render with Graphviz (must be installed separately):

```sh
dot -Tpdf ../data/ccRCC/RK26.txt.dot -O
```

<p align="center">
<img src="https://github.com/viq854/lichee/blob/master/img_demo/RK26.txt.dot.png" width="65%" height="65%" />
</p>

**Example 2: ccRCC patient RMH008**

```sh
./lichee -build -i ../data/ccRCC/RMH008.txt -maxVAFAbsent 0.005 -minVAFPresent 0.005 -n 0 -minPrivateClusterSize 2 -showTree 1 -color -dot
```

```sh
dot -Tpdf ../data/ccRCC/RMH008.txt.dot -O
```

<p align="center">
<img src="https://github.com/viq854/lichee/blob/master/img_demo/RMH008.txt.color.dot.png" width="65%" height="65%" />
</p>

Plain mode simple look (without ```-color``` flag):

<p align="center">
<img src="https://github.com/viq854/lichee/blob/master/img_demo/RMH008.txt.dot.png" width="65%" height="65%" />
</p>

GUI interaction examples:

Cluster node 10 is selected, sample contributions highlighted in purple.

<p align="center">
<img src="https://github.com/viq854/lichee/blob/master/img_demo/lichee_cluster_demo.png" width="65%" height="65%" />
</p>


Sample node R5 is selected, lineages highlighted in purple:

<p align="center">
<img src="https://github.com/viq854/lichee/blob/master/img_demo/lichee_sample_demo.png" width="65%" height="65%" />
</p>

### Parameter Tuning and Diagnostics

LICHeE may not find a valid lineage tree under every parameter setting, or may produce multiple alternative trees under different settings. Exploring the parameter space is recommended for each new dataset.

Tuning guidance:

- `-maxVAFAbsent` and `-minVAFPresent` control the presence/absence thresholds. Adjust these to match the expected noise level in your data, or supply pre-computed calls with `-sampleProfile`.
- Small clusters are more likely to represent mis-called patterns. Increase `-minClusterSize` and `-minPrivateClusterSize` to filter them out.
- Increase `-minRobustNodeSupport` to iteratively remove weakly supported nodes when no valid tree is found.
- Increase `-e` to tolerate more VAF noise (use sparingly).
- Lower the above thresholds on cleaner datasets to retain more clusters and produce more granular trees.

LICHeE prints a step-by-step log during execution. Use `-v` for more detail. The log covers SSNV calling, clustering, the constraint network structure, and tree scoring.

To trace why a specific SSNV was excluded, search the log for the `Filtered` keyword or the unique SSNV descriptor. When no valid trees are found, examining the cluster centroid VAFs and constraint network topology can reveal which phylogenetic constraint is violated.


### System Requirements

- Java 17 or later (Java 21 recommended)
- Graphviz (optional, for DOT file rendering)

### License

MIT License 

### Support

For help running the program or any questions/suggestions/bug reports, please contact viq@cs.stanford.edu
