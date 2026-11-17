# NovoCRF

NovoCRF performs **de novo peptide sequencing** from tandem mass spectra: for each MS/MS spectrum it predicts
the most likely peptide sequence, with no protein database. It reads:

- **Thermo `.raw`** files and **Bruker timsTOF `.d`** folders directly (with the vendor reader libraries you
  download once, see [Reading Thermo and Bruker raw files](#reading-thermo-and-bruker-raw-files)), and
- **MGF** peak lists.

**Run NovoCRF on the raw files. It needs the MS1 scans, to correct each spectrum's precursor, and the collision
energy recorded in the raw file, which an MGF does not carry. Support for MGF input is legacy and may be
deprecated.**

Everything NovoCRF needs, including the trained model, is inside the program. There is nothing to install and no
model file to manage. It uses an NVIDIA GPU when the computer has a suitable one and runs on the CPU otherwise
(see [GPU](#gpu)).

---

## Files in this release

```
README.md
LICENSE.md                <- licence terms for NovoCRF
THIRD-PARTY-NOTICES.txt   <- licences of the bundled third-party components and of the model's training data
SHA256SUMS.txt            <- checksums of the programs below, to check your download
vendor_reader/            <- where you put the downloaded vendor libraries (starts empty)
windows/
  novocrf.exe             <- the program (Windows x64)
  vcomp140.dll            <- the OpenMP runtime novocrf.exe uses (Microsoft; keep it next to novocrf.exe)
  DotnetBridge.exe        <- used to read Thermo .raw
linux/
  novocrf                 <- the program (Linux x64)
  DotnetBridge            <- used to read Thermo .raw
```

Use the `windows/` or `linux/` folder for your platform. Keep the whole release folder together: each program
finds the shared `vendor_reader/` folder automatically, and reads Thermo `.raw` through the `DotnetBridge` program
sitting next to it. Nothing has to be installed: `DotnetBridge` carries its own .NET runtime, and `novocrf` carries
the parts of CUDA it needs.

### Checking the download

`SHA256SUMS.txt` lists the SHA-256 checksum of every program file. To check them, run in the release folder:

```
# Linux
sha256sum -c SHA256SUMS.txt

# Windows (PowerShell)
Get-Content SHA256SUMS.txt | ForEach-Object { $h, $f = $_ -split '\s+', 2; if ((Get-FileHash $f).Hash -eq $h) { "OK      $f" } else { "FAILED  $f" } }
```

Every line should say OK. In addition, `novocrf` checks its built-in model every time it starts and stops with a
message if the program file is damaged.

---

## System requirements

- **Operating system:** Windows 10 or 11 (x64), or Linux x64 with glibc 2.34 or newer (Ubuntu 22.04+, Debian 12+,
  RHEL / Rocky / AlmaLinux 9+).
- **CPU:** x86-64 with AVX2 (Intel Haswell (2013) or later, AMD Zen or later).
- **Memory:** 16 GB of RAM or more is recommended. Runs of 6,000 to 208,000 spectra used at most 8.4 GB (Linux,
  16 threads, with a GPU).
- **GPU (optional):** an NVIDIA GPU with 12 GB of memory or more, and an NVIDIA driver for CUDA 12.0 or newer (driver
  version 525 or newer; CUDA 13 drivers work too). The program contains code for every NVIDIA architecture from
  Volta (V100) through Turing, Ampere (A100, RTX 30), Ada (L40, RTX 40), Hopper (H100) and Blackwell (B200, RTX 50),
  plus a portable form that newer GPUs compile when the program starts. No CUDA toolkit is needed: the driver is
  enough. Without a suitable GPU everything runs on the CPU.

---

## Quick start

Open the `windows/` or `linux/` folder for your platform and run NovoCRF from there.

```
# a Thermo raw file (after installing the Thermo reader, see below)
novocrf sample.raw -o results.csv

# a Bruker timsTOF .d folder (after installing the Bruker reader)
novocrf sample.d -o results.csv

# an MGF file (legacy input: no MS1 precursor correction, no recorded collision energy)
novocrf spectra.mgf -o results.csv

# several runs: every .raw in a folder, each run decoded on its own, one CSV per run in results/
novocrf "rawdata/*.raw" -o results/
```

(On Linux, write `./novocrf` instead of `novocrf`.)

**These runs search only two variable modifications: M oxidation and N-terminal acetylation** (with the fixed
carbamidomethyl C). Phospho, the other N-terminal modifications and unmodified cysteine are searched only when you
switch them on with `--ptms`, see [Modifications](#modifications).

---

## What a run does

By default NovoCRF decodes each run **twice**:

1. **Round 1:** every spectrum is decoded with the built-in model.
2. **Between the rounds,** the model learns from the run's own confident answers (at least 1,000 are needed): part
   of the model is fine-tuned on them, the precursor and fragment m/z are calibrated, the precursor mass is
   re-checked from the fragment ions, and a retention-time term is fitted (raw input, or MGF with `RTINSECONDS`).
3. **Round 2:** every spectrum is decoded again. These are the final predictions.

Round 2 takes about as long as round 1. `--no-otf` stops after round 1 (half the time, somewhat lower recall). A
run with fewer than 1,000 confident answers keeps its round-1 predictions.

Next to the output CSV NovoCRF writes:

- `NAME.round1.csv`: the round-1 predictions (what `--no-otf` gives), as soon as round 1 is done.
- `NAME.log`: everything the program prints: the computer and the devices used, warnings, and at the end the time
  of each part (`timing: reading, round 1, between the rounds, round 2, writing; total`).

---

## Output

A CSV file with one row per spectrum, in input order. The columns follow the
[denovo_benchmarks](https://github.com/PominovaMS/denovo_benchmarks) common format, plus extras:

| Column | Meaning |
|--------|---------|
| `spectrum_id` | `<run>:<index>`: the run's file name without extension, and the 0-based index of the spectrum in the run |
| `sequence` | the predicted peptide in ProForma with UNIMOD codes, e.g. `[UNIMOD:1]-PEPC[UNIMOD:4]M[UNIMOD:35]K`; empty if no peptide fits the precursor mass |
| `score` | confidence from 0 to 1: the calibrated probability that the whole peptide is correct |
| `aa_scores` | per-residue scores (the peptide score repeated; an N-terminal modification counts as a residue) |
| `novocrf_sequence` | the same peptide with mass deltas, e.g. `C+57.021`, `M+15.995`, `+42.011` (N-terminal) |
| `isotope` | the precursor isotope of the answer: 0 = the measured (monoisotopic) precursor; with `--m1` or `--isotopes`, the offset chosen (1 = the +1 isotope) |
| `charge`, `precursor_mz` | the precursor used (raw input: corrected from the MS1 data, see below) |
| `title` | raw input: `<run>.<scan>.<scan>.<charge>` (timsTOF: the precursor id in place of the scan), so you can map each row back to its scan; MGF input: the spectrum's `TITLE` |
| `q_value` | with `--fdr` only: the q-value from decoy spectra, computed within the run |

- **I and L** are both reported as **L**; **deamidated N/Q** as **D/E** (identical masses).
- **Fixed carbamidomethyl C** is `C[UNIMOD:4]` (`C+57.021`).
- **User modifications** (`--extra-ptms`) are written with their mass, e.g. `C[+119.004]`.

Example:

```
spectrum_id,sequence,score,aa_scores,novocrf_sequence,isotope,charge,precursor_mz,title
C1_1_03:994,GSC[UNIMOD:4]FHR,0.99691,"0.99691,0.99691,0.99691,0.99691,0.99691,0.99691",GSC+57.021FHR,0,2,382.168152,C1_1_03.2583.2583.2
C1_1_03:1864,[UNIMOD:1]-LVSETNDTK,0.96636,"0.96636,0.96636,0.96636,0.96636,0.96636,0.96636,0.96636,0.96636,0.96636,0.96636",+42.011LVSETNDTK,0,2,524.758484,C1_1_03.3474.3474.2
C1_1_03:2917,LYDSM[UNIMOD:35]K,0.96384,"0.96384,0.96384,0.96384,0.96384,0.96384,0.96384",LYDSM+15.995K,0,2,386.680725,C1_1_03.4554.4554.2
```

---

## GPU

NovoCRF checks the computer when it starts. It uses the first NVIDIA GPU that qualifies:

- the NVIDIA driver supports CUDA 12.0 or newer,
- the GPU has 12 GB of memory or more,
- the program has code for its architecture (every architecture listed under System requirements).

It then runs the neural-network parts (the CRF's scorer, the three rerankers, the fragment-intensity predictor and
the fine-tuning between the rounds) on the GPU, and the rest (spectrum preprocessing, the exact candidate search) on
all CPU cores. It uses about 9 GB of GPU memory at most, whatever the GPU and the input.

The log says what it found and what runs where, for example:

```
host: CPU AMD EPYC 7513 32-Core Processor, 128 hardware threads
host: GPU 0 NVIDIA A100-SXM4-80GB [PCI 0000:01:00], 79.3 GiB, compute capability 8.0 (qualifies)
host: NVIDIA driver supports CUDA 13.4; this build: CUDA 12.8
compute: CPU (16 threads) + GPU 0 (NVIDIA A100-SXM4-80GB [PCI 0000:01:00])
  on the GPU: CRF potentials (features + scorer MLP), first-stage reranker, second-stage reranker, intensity predictor, last-stage reranker, on-the-fly fine-tuning (CRF training, predictor trunk)
  on the CPU: spectrum preprocessing, k-best search, reranker evidence features, CRF edge scores + Viterbi, predictor head training (fine-tuning)
```

Without a qualifying GPU the `compute:` line says why, e.g. `compute: CPU only (32 threads): no NVIDIA driver
found`, and everything runs on the CPU.

- `--no-gpu` runs on the CPU even when a GPU qualifies. `--device N` picks GPU N (CUDA numbering; an error if it
  cannot be used). `CUDA_VISIBLE_DEVICES` works as usual.
- **GPU and CPU results are not bit-identical.** The GPU sums in a different order and fine-tunes in single
  precision, so near-ties can be decided differently, mostly in round 2: on one test set 93% of the final answers
  were identical, with recall 84.46 on the GPU and 84.41 on the CPU. Runs on the same computer give identical
  results; another GPU model or operating system can decide a few near-ties differently.
- On a shared computer, the first qualifying GPU is used even if another job is using it: choose one with
  `--device` or `CUDA_VISIBLE_DEVICES`. `NOVOCRF_GPU_MEM=6` limits the working memory to 6 GB.

---

## Reading Thermo and Bruker raw files

NovoCRF does not include the vendor reader libraries: you download them once from the vendor and accept the
vendor's licence. Put the downloaded files anywhere inside the `vendor_reader/` folder of this release; the exact
sub-folder and version do not matter, NovoCRF searches that folder by file name.

### Thermo `.raw`

1. Download **RawFileReader** from the official Thermo repository:
   <https://github.com/thermofisherlsms/RawFileReader>
2. In the repository, open `Libs/NetCore/Net8` (the cross-platform build, the same files for Windows and Linux).
   Each `.nupkg` file is a ZIP archive; from the two packages `thermofisher.commoncore.rawfilereader` and
   `thermofisher.commoncore.data`, extract the DLLs `lib/net8.0/ThermoFisher.CommonCore.RawFileReader.dll` and
   `lib/net8.0/ThermoFisher.CommonCore.Data.dll` (rename a `.nupkg` to `.zip` if your extractor needs it).
3. Put those two `.dll` files anywhere inside `vendor_reader/`, for example `vendor_reader/thermo/`.

Use `Libs/NetCore/Net8`, not `Libs/Net48`: the Net48 DLLs have the same names but are for the older .NET
Framework, and NovoCRF skips them (it tells you if those are all it finds). `DotnetBridge` provides the rest.

### Bruker timsTOF `.d`

1. Download the **TDF SDK** from the official Bruker download page:
   <https://www.bruker.com/service/support-upgrades/software-downloads/mass-spectrometry.html>
2. From the SDK, take the shared library for your platform:
   - Windows: `timsdata.dll` (also install the *Microsoft Visual C++ 2015–2022 Redistributable (x64)*)
   - Linux: `libtimsdata.so`
3. Put that file anywhere inside `vendor_reader/`, for example `vendor_reader/bruker/`.

If a vendor library is missing when you open a `.raw` or `.d` file, NovoCRF prints a message telling you which
file it could not find. To keep the libraries elsewhere, point `NOVOCRF_THERMO` (the folder with the two Thermo
DLLs) or `NOVOCRF_TIMSDATA` (the `timsdata` library file) at them.

### Precursors

NovoCRF searches only peptides that fit each spectrum's precursor m/z and charge. When it reads a raw file it
corrects every precursor from the MS1 data (the isolated peak and its isotope envelope), which fixes the many
precursors that instruments and converters report one or more isotopes off. **MGF input is used as it is**: if
you can, give NovoCRF the raw files rather than MGFs converted from them.

---

## Modifications

**By default NovoCRF searches only two variable modifications: oxidation of M (`M+15.995`) and N-terminal
acetylation (`+42.011`)**, plus the fixed carbamidomethyl cysteine (`C+57.021`). **Every other modification below
is off until you name it with `--ptms`.**

| Name | Modification | Default |
|------|--------------|---------|
| `oxidation` | oxidation of M (`M+15.995`) | **on** |
| `acetylation` | N-terminal acetylation (`+42.011`) | **on** |
| `phospho` | phosphorylation of S/T/Y (`+79.966`) | off |
| `carbamylation` | N-terminal carbamylation (`+43.006`) | off |
| `pyroglu` | pyroglutamate from N-terminal Q (`-17.027`) | off |
| `unmodified_c` | allow an **unmodified cysteine** (a bare `C`) in addition to `C+57.021` | off |
| `deamidation` | deamidation of N and Q (`+0.984`) | always reported as D/E (see below) |

- **`--ptms` replaces the default list.** Give every modification you want, `oxidation,acetylation` included if
  you still want them. For a phospho-enriched sample: `--ptms oxidation,acetylation,phospho`.
- `--ptms default` is `oxidation,acetylation`; `--ptms all` switches on every name in the table; `--ptms none`
  searches no variable modification.
- **Deamidation is not a switch.** A deamidated N or Q has exactly the mass of D or E, so NovoCRF always
  considers it and always reports it as D or E, whatever `--ptms` says. The name `deamidation` is accepted in
  `--ptms` but changes nothing.
- Releases before v0.0.2 searched every name but `phospho` by default. For those searches, give
  `--ptms oxidation,acetylation,carbamylation,pyroglu,unmodified_c`.
- `--max-ptms N` allows at most N variable modifications per peptide (default: no limit).

### Cysteine

Carbamidomethyl cysteine (`C+57.021`, from iodoacetamide / chloroacetamide alkylation) is searched by default.

- **Alkylated samples:** the default reports every cysteine as `C+57.021`. If the alkylation may be incomplete,
  add `unmodified_c` to also allow a bare `C`: `--ptms oxidation,acetylation,unmodified_c`.
- **Non-alkylated samples:** `--no-cam --ptms oxidation,acetylation,unmodified_c` searches bare `C` only. (With
  `--no-cam` and without `unmodified_c`, no cysteine is searched at all, and NovoCRF warns.)
- **A different alkylation,** e.g. NEM (`C+125.048`): add it with `--extra-ptms C+125.048`, together with
  `--no-cam` if no cysteine carries `C+57.021`.

### Other modifications

`--extra-ptms` adds modifications the model was not trained on, each as a residue letter and a signed mass delta
(comma-separated for several):

```
novocrf spectra.mgf --extra-ptms C+119.004,P+15.995 -o results.csv
```

An entry at the mass of a built-in modification that is switched on is skipped with a warning; one at the mass of a
built-in modification that is switched off (e.g. `S+79.966` without `phospho`) is searched as a user modification.
`--extra-prior X` (default 0) makes all of them more (X > 0) or less (X < 0) likely to be called.

---

## Several runs

Give a list of runs, a folder, or a quoted wildcard pattern such as `"rawdata/*.raw"` (NovoCRF expands `*`, `?`
and `[...]` itself; a folder or a pattern takes only its `.raw`, `.d` and `.mgf` entries).

- **Each run on its own (the default):** the runs are decoded one by one, and each is fine-tuned on its own
  confident answers: every run gets exactly the predictions it gets when given alone. The model is loaded and the
  GPU set up once.
- **`--pooled-otf`:** one fine-tuning on the confident answers pooled over all runs (round 1 of every run, the
  fine-tuning, then round 2 of every run). This helps small runs of one sample type (fewer than 1,000 confident
  answers each); runs that differ (treatments, instruments) can lose.
- **Output:** `-o results/` (any name not ending in `.csv`) writes `results/<run>.csv` and
  `results/<run>.round1.csv` per run, with `results/novocrf.log`; `-o all.csv` writes one CSV of every run's
  predictions, joined in input order. Without `-o`: `novocrf_results/`.
- A run that cannot be read is skipped and named at the end; the others are decoded.

---

## Options

```
usage: novocrf INPUT... [-o PATH] [--weights FILE] [--ptms LIST] [--extra-ptms LIST] [--fdr] [--threads N]
               [more options below]

Input and output
  INPUT        Thermo .raw files, Bruker timsTOF .d folders, MGF files, folders of these, or wildcard patterns
               (quoted, e.g. "rawdata/*.raw": the program expands *, ? and [...] itself; a folder or pattern
               takes only its .raw, .d and .mgf entries; .gz files must be decompressed). Raw runs are read
               directly, with their MS1-corrected precursors: Thermo .raw through DotnetBridge (next to the
               program) and Thermo's RawFileReader DLLs, Bruker .d through Bruker's timsdata library, both
               put by you in vendor_reader/ (see the README; MGF input needs neither)
               Several runs: decoded one by one, each fine-tuned on its own confident answers, with the same
               predictions as each run given alone (--pooled-otf: one fine-tuning on all runs' answers)
  -o PATH      a PATH ending in .csv: one CSV (several runs: every run's predictions, joined); any other
               PATH: a folder of <run>.csv. Default: predictions.csv for one run, novocrf_results/ for
               several. An earlier run's outputs of the same names are replaced. Next to it: NAME.log (all
               that is printed here, with the time of each round; several runs in a folder: novocrf.log)
               and, when the fine-tuning runs (the default), NAME.round1.csv: the first round's predictions
               (what --no-otf gives), written as soon as that round is done
  --weights F  a model file instead of the model built into the program
  --           what follows are inputs, even a name that starts with -
  --version    the program's version

Modifications and precursor
  --ptms L     enabled variable PTMs: comma-separated subset of
               oxidation,unmodified_c,deamidation,acetylation,carbamylation,pyroglu,phospho;
               default: oxidation,acetylation (M+15.995, N-terminal +42.011); every other PTM is
               searched only when listed; 'all': every one; 'none': no variable PTM. deamidation
               changes nothing: deamidated N/Q have the masses of D/E and are always reported as D/E
  --extra-ptms L  PTMs the model was not trained on, as residue + mass delta, e.g. C+119.004,P+15.995
               (one at the mass of a built-in edge that is switched on is skipped with a warning;
               --allow-duplicate-ptms keeps it; at the mass of one switched off, e.g. S+79.966 without phospho,
               it is searched as a user PTM)
  --extra-prior X  added to the --extra-ptms residues' prior (default 0; lower it for fewer calls)
  --max-ptms N at most N variable PTMs per peptide: oxidation, phospho, N-terminal mods and --extra-ptms
               (deamidated N/Q read as D/E and are not counted); default -1 = no limit
  --no-cam     no carbamidomethyl C (C+57.021) edge: for non-alkylated samples (with --ptms ...,unmodified_c)
  --m1         also consider that the precursor m/z is the +1 isotope (M1) peak
  --isotopes L precursor isotope offsets to consider (-3 to 5, the first 0), e.g. 0,1,2 or 0,1,-1
               (overrides --m1)

Search
  --peaks N    read the N most intense peaks of every spectrum (default: the model file's; the bundled
               model reads 400 at every charge)
  --nbest N    CRF candidates per spectrum for the rerankers (default 1000)
  --nbest-hi N candidates for precursors heavier than 1800 Da (default 2500; 0, or not above --nbest:
               --nbest for all)
  --max-pops N k-best search budget per spectrum (default 120000)
  --no-rerank  CRF top-1 only (faster, lower recall)

Learnt on each run (from its own confident answers; each step can be switched off)
  --otf        on-the-fly fine-tuning, the default: after a first pass, part of the CRF and the head of the
               rerankers' fragment-intensity predictor are trained on the run's confident answers (the 30%
               most confident with >= 6 residues; >= 1000 needed), then every spectrum is decoded again
  --reranker-otf  fine-tune the rerankers' intensity predictor only, not the CRF: less gain, faster (only the
               last stage runs again; with --pooled-otf every run is decoded again in full)
  --no-otf     no fine-tuning: one round of prediction (the retention-time term below still applies)
  --pooled-otf several runs of one experiment: one fine-tuning on the confident answers pooled over all runs
               (first pass of every run, fine-tuning, second pass of every run), instead of each run its own.
               Pays where a run alone has few confident answers (e.g. fewer than the 1000 needed); runs that
               differ (treatments, instruments) can lose. The collision energy is fitted on the pooled answers
               too; calibration, precursor votes and retention time stay per run
  --otf-pool-max N  with --pooled-otf: fine-tune on at most N pooled answers (random subset; default 100000;
               0 = all)
  between the passes:
    --no-prec-cal   precursor calibration: the run's precursor error (ppm) is removed
    --no-ms2-cal    MS2 calibration: the fragment m/z shift, fitted as a 4th-degree polynomial of m/z, is
               removed from the peaks
    --prec-vote replace|second|off   the y ions of every first-pass answer vote for the precursor mass;
               where they agree on a mass other than the measured one, the spectrum is decoded at it:
               replace (default) = instead of the measured mass; second = both masses (a little more
               recall, up to 1.14x the time); off
  --no-ce-raw  the intensity predictor ignores the collision energy (NCE) recorded in raw files; the run's
               collision energy is fitted on its confident answers instead, as for MGF input
  --no-rt      no retention-time term. By default (runs with retention times), a regression of retention
               time on sequence features and the model's iRT prediction is fitted on each run's confident
               answers (>= 500 needed), and |observed - predicted RT| re-ranks every spectrum's last candidates
  --rt-weight W  the retention-time term's weight per sigma of the run's RT error (default 1.0 with the
               model's iRT predictor; 0 = fitted on the run's confident answers)

False discovery rate
  --fdr        q-values from decoy spectra (after NovoBoard, Tran et al. 2024): every spectrum gets a decoy
               (same precursor, --decoy-remove X (0.5) of its peaks replaced by peaks of the run) that is
               decoded like a target, about 2x the time; the CSV gets a q_value column (several runs: the
               q-values of each run are its own). X: above 0 and at most 1

Hardware
  --threads N  CPU threads (default, or 0: all)
  --no-gpu     do not use the GPU (also --cpu). By default a GPU is used when one qualifies: CUDA 12+
               driver and 12 GB of memory or more (GPU build); otherwise everything runs on the CPU. It uses
               about 9 GB of the GPU's memory at most ($NOVOCRF_GPU_MEM=GB sets its 8 GB working budget)
  --device N   use GPU N (as numbered by CUDA / CUDA_VISIBLE_DEVICES; default: the first qualifying one); an
               error if it cannot be used
  --gpu-parts L  what runs on the GPU: comma-separated subset of crf,vit,rr1,rr1b,ip,rr2,otf, all, none (the
               CPU only, as --no-gpu), or default (= all but vit: the CRF Viterbi scan stays on the CPU, which
               is as fast with about 8 or more cores and moves no lattice scores over PCIe; the GPU scan gives
               the same results bit for bit)
```

---

## Errors

- Option values are checked before anything is read: a value that is not a number or is out of range, an unknown
  modification name, or an output that cannot be written stops the program with a message (exit code 2).
- MGF spectra that cannot be decoded (no `PEPMASS`, `CHARGE` 0 or negative, a precursor mass outside 75–50,000 Da,
  no `END IONS`) are skipped with one warning per file; a run with no readable spectrum is an error (exit code 1).
- A failed or interrupted run leaves no old or half-written CSV under its output names; its log says why it stopped.
- The built-in model is checked against its SHA-256 checksum every time the program starts. If the program file is
  damaged (an interrupted download or copy, a failing disk), NovoCRF stops with "the model built into this program
  is damaged" (exit code 1): download the release again. A model given with `--weights FILE` is checked against
  `FILE.sha256` when that file exists next to it.

---

## Examples

```
# A phospho-enriched sample, 16 threads (phospho added to the default oxidation and acetylation)
novocrf phospho.raw --ptms oxidation,acetylation,phospho --threads 16 -o phospho.csv

# Every built-in modification: phospho, N-terminal carbamylation and pyro-glu, unmodified C as well
novocrf sample.raw --ptms all -o all_ptms.csv

# A non-alkylated sample with cysteinylation
novocrf sample.raw --no-cam --ptms oxidation,acetylation,unmodified_c --extra-ptms C+119.004 -o out.csv

# One round only (half the time)
novocrf sample.raw --no-otf -o quick.csv

# CPU only, with decoy-based q-values
novocrf sample.mgf --no-gpu --fdr -o with_q.csv

# Fractions of one sample, fine-tuned together, one CSV per run
novocrf fractions/ --pooled-otf -o fractions_results/
```

---

## Citation

If you used NovoCRF for your research, please cite https://arxiv.org/pdf/2611.05260:

Rui Qiao, Lei Xin, Baozhen Shan. NovoCRF: Globally Normalized De Novo Peptide Sequencing on a Prefix-Mass Lattice.
arXiv:2611.05260, 2026. https://arxiv.org/pdf/2611.05260

```bibtex
@misc{qiao2026novocrf,
  title         = {NovoCRF: Globally Normalized De Novo Peptide Sequencing on a Prefix-Mass Lattice},
  author        = {Qiao, Rui and Xin, Lei and Shan, Baozhen},
  year          = {2026},
  eprint        = {2611.05260},
  archivePrefix = {arXiv},
  url           = {https://arxiv.org/abs/2611.05260}
}
```
