# NovoCRF

NovoCRF performs **de novo peptide sequencing** from tandem mass spectra: for each MS/MS spectrum it predicts
the most likely peptide sequence, with no protein database. It reads:

- **MGF** peak lists, and
- **Thermo `.raw`** files and **Bruker timsTOF `.d`** folders directly (with the vendor reader libraries you
  download once — see [Reading Thermo and Bruker raw files](#reading-thermo-and-bruker-raw-files)).

Everything NovoCRF needs, including the trained model, is inside the single executable. There is nothing to
install and no model file to manage.

---

## Files in this release

```
README.md
LICENSE.md                <- licence terms for NovoCRF
THIRD-PARTY-NOTICES.txt   <- licences of the bundled third-party components
vendor_reader/            <- where you put the downloaded vendor libraries (starts empty)
windows/
  novocrf.exe             <- the program (Windows x64)
  DotnetBridge.exe        <- used to read Thermo .raw
  System.*.dll (4 files)  <- used to read Thermo .raw
linux/
  novocrf                 <- the program (Linux x64)
```

Use the `windows/` or `linux/` folder for your platform. Keep the whole release folder together: each program
finds the shared `vendor_reader/` folder automatically, and on Windows `novocrf.exe` uses `DotnetBridge.exe` and
the `System.*.dll` files sitting next to it.

> **Linux note:** reading Thermo `.raw` is Windows-only in this release (it relies on `DotnetBridge.exe`). On
> Linux, NovoCRF reads MGF and Bruker `.d`. The Linux binary needs glibc 2.35 or newer (Ubuntu 22.04+,
> Debian 12+, RHEL 9+).

---

## Quick start

Open the `windows/` or `linux/` folder for your platform and run NovoCRF from there.

```
# MGF in, CSV out
novocrf spectra.mgf -o results.csv

# a Thermo raw file (after installing the Thermo reader, see below)
novocrf sample.raw -o results.csv

# a Bruker timsTOF .d folder (after installing the Bruker reader)
novocrf sample.d -o results.csv

# every .mgf / .raw / .d in a folder
novocrf my_runs/ -o results.csv
```

(On Linux, write `./novocrf` instead of `novocrf`.)

---

## Output

A CSV file with one row per spectrum and these columns:

| Column | Meaning |
|--------|---------|
| `ScanNumber` | the MS/MS scan number, so you can map the result back to the original scan in the raw file (empty if the input did not provide one) |
| `RTInSeconds` | retention time of the scan, in seconds (empty if not available) |
| `PredictedSequence` | the predicted peptide. Modifications are written inline, e.g. `C+57.021`, `M+15.995`, `S+79.966` |
| `Score` | confidence from 0 to 1 (higher is more confident) |
| `PrecursorMz` | precursor m/z used for the prediction |
| `Charge` | precursor charge |

Example:

```
ScanNumber,RTInSeconds,PredictedSequence,Score,PrecursorMz,Charge
1021,634.8120,LFHGLKSTDVAK,0.89815,439.252370,3
1022,635.1044,GTANLQLHGPPR,0.85246,630.845807,2
```

---

## Reading Thermo and Bruker raw files

NovoCRF does not include the vendor reader libraries — you download them once from the vendor and accept the
vendor's licence. Put the downloaded files anywhere inside the `vendor_reader/` folder of this release; the exact
sub-folder and version do not matter, NovoCRF searches that folder by file name.

### Thermo `.raw` (Windows)

1. Download **RawFileReader** from the official Thermo repository:
   <https://github.com/thermofisherlsms/RawFileReader>
2. In the repository, open `Libs/Net48`. Each `.nupkg` file is a ZIP archive; from the two packages
   `ThermoFisher.CommonCore.RawFileReader` and `ThermoFisher.CommonCore.Data`, extract the DLLs
   `lib/net48/ThermoFisher.CommonCore.RawFileReader.dll` and `lib/net48/ThermoFisher.CommonCore.Data.dll`
   (rename a `.nupkg` to `.zip` if your extractor needs it).
3. Put those two `.dll` files anywhere inside `vendor_reader/`, for example `vendor_reader/thermo/`.

`DotnetBridge.exe` and the `System.*.dll` files shipped in this release provide the rest.

### Bruker timsTOF `.d`

1. Download the **TDF SDK** from the official Bruker download page:
   <https://www.bruker.com/service/support-upgrades/software-downloads/mass-spectrometry.html>
2. From the SDK, take the shared library for your platform:
   - Windows: `timsdata.dll` (also install the *Microsoft Visual C++ 2015–2022 Redistributable (x64)*)
   - Linux: `libtimsdata.so`
3. Put that file anywhere inside `vendor_reader/`, for example `vendor_reader/bruker/`.

If a vendor library is missing when you open a `.raw` or `.d` file, NovoCRF prints a message telling you which
file it could not find.

---

## Options

```
novocrf INPUT... [options]

  INPUT                  one or more .mgf files, .raw files, .d folders, or folders containing them

  -o, --out FILE         write results to FILE (default: predictions.csv)
      --ptms LIST        variable modifications to allow, comma-separated, from:
                           oxidation, unmodified_c, deamidation, acetylation,
                           carbamylation, pyroglu, phospho
                         default: all except phospho. 'all' adds phospho; 'none' = unmodified only.
      --extra-ptms LIST  additional modifications as residue+mass delta, e.g. C+119.004,S-18.011
      --m1               also try the precursor as its +1 (13C) isotope peak
  -t, --threads N        number of worker threads (default: all CPU cores)
      --fast             skip the reranking stage (faster, slightly lower recall)
      --weights FILE     use an external model file instead of the built-in model
  -h, --help             show help and exit

  raw-file options (Thermo .raw / Bruker .d):
      --ms1-scans N      MS1 scans summed on each side of the precursor (default 0 Thermo, 1 timsTOF)
      --tol-ppm X        tolerance for the isolated precursor peak (default 15 Thermo, 20 timsTOF)
      --tims-centroid own|sdk   timsTOF MS1 centroiding (default own)
```

### Modifications

NovoCRF searches a built-in set of modifications. **Carbamidomethylation of cysteine (`C+57.021`, from the
standard iodoacetamide / chloroacetamide alkylation) is always searched** — it is a fixed modification and is on
no matter what you pass. Every cysteine may be reported as `C+57.021`.

The `--ptms` option chooses which *variable* modifications are considered too, as a comma-separated list from:

| Name | Modification |
|------|--------------|
| `oxidation` | oxidation of M (`M+15.995`) |
| `deamidation` | deamidation of N and Q (`+0.984`) |
| `acetylation` | N-terminal acetylation (`+42.011`) |
| `carbamylation` | N-terminal carbamylation (`+43.006`) |
| `pyroglu` | pyroglutamate from N-terminal Q (`-17.027`) |
| `phospho` | phosphorylation of S/T/Y (`S+79.966`) |
| `unmodified_c` | allow an **unmodified cysteine** (a bare `C`) in addition to `C+57.021` |

- No `--ptms` (or `--ptms default`) enables every name **except** `phospho`.
- `--ptms all` adds `phospho`; `--ptms none` turns all variable modifications off.
- An explicit list enables **only** the names you give and turns the rest off.

#### Fixed carbamidomethylation of cysteine — on or off

`C+57.021` is always available, so you control cysteine through the `unmodified_c` option — that is, you choose
whether a **bare, unmodified `C`** is allowed alongside the carbamidomethylated form:

- **Alkylated samples (the usual case) — strict fixed modification.** To report every cysteine as `C+57.021` and
  never a bare `C`, leave `unmodified_c` **out** of `--ptms`. The quickest way is to name the variable
  modifications you want explicitly:

  ```
  novocrf spectra.mgf --ptms oxidation,deamidation,acetylation -o results.csv
  ```
  This list has no `unmodified_c`, so unmodified cysteine is off and carbamidomethylation is effectively fixed.
  (Note: the **default** does include `unmodified_c`, so plain `novocrf spectra.mgf` still allows a bare `C`.)

- **Non-alkylated samples, or when you are unsure.** Keep `unmodified_c` **on** so a bare `C` can also be
  predicted. It is on in the default, so either run with no `--ptms`, or include it in your list:

  ```
  novocrf spectra.mgf -o results.csv                                       # default: bare C allowed
  novocrf spectra.mgf --ptms oxidation,deamidation,unmodified_c -o out.csv  # explicit list, bare C allowed
  ```

> `C+57.021` itself cannot be switched off through `--ptms`; it is always searched. If your cysteines carry a
> *different* alkylation — for example NEM (`C+125.048`) or acrylamide (`C+71.037`) — add it with `--extra-ptms`
> (below); NovoCRF will then consider that form as well as `C+57.021`.

#### Other / custom modifications

Add modifications that are not in the list above with `--extra-ptms`, giving a residue letter and a signed mass
delta (comma-separated for several):

```
novocrf spectra.mgf --extra-ptms "C+125.048,K+42.011" -o results.csv
```

---

## Examples

```
# Basic MGF run
novocrf spectra.mgf -o results.csv

# A phospho-enriched sample, using 8 threads
novocrf phospho.mgf --ptms all -t 8 -o phospho_results.csv

# A Thermo raw file, considering the +1 isotope as an alternative precursor
novocrf sample.raw --m1 -o sample_results.csv

# A whole folder of runs
novocrf 2026-04-run/ -o run_results.csv
```
