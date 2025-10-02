### Full pipeline (for reference)

```bash
find examples -type f -name '*.fasta' | sort \
| awk -F'/' '{printf "%d\t%s\t%s\n", NR, $2, $3}' \
| awk 'NR>5' \
| awk -F'\t' 'BEGIN{OFS="\t"} {sub(/^ex[0-9]+_/,"",$2); print $1,$2,$3}' \
| awk -F'\t' '$3 ~ /chain/' \
| awk -F'\t' 'BEGIN{OFS="\t"} {print n++,$2,$3}' \
| awk -F'\t' 'BEGIN{OFS="\t"} {print $1,$2,$3,"examples/"$2"/"$3}' \
| awk -F'\t' '$2 !~ /\+/'
```

---

## 1) `find … | sort`

```bash
find examples -type f -name '*.fasta' | sort
```

* **Goal:** list all FASTA files under `examples`, then make the order deterministic.
* `find examples -type f -name '*.fasta'`: only **files** (not dirs), names ending in `.fasta`. The pattern is evaluated by `find`, not the shell—so no accidental globbing.
* `| sort`: ensures stable, lexicographic ordering. (Raw `find` order can vary across filesystems.)

**Output format now:** `examples/<exDir>/<file>`

---

## 2) Extract 3 columns; number them within this `awk`

```bash
… | awk -F'/' '{printf "%d\t%s\t%s\n", NR, $2, $3}'
```

* `-F'/'`: **input field separator** is `/`. Each path is split into fields:

  * `$1 = examples`
  * `$2 = exDir` (e.g., `ex03_1crn_A` or `ex07_2zta_A+B`)
  * `$3 = file` (e.g., `seq_1crn_chainA.fasta`)
* `NR`: AWK’s **record number** (line counter) **for this `awk` process**.
* `printf "%d\t%s\t%s\n", …`: prints **tab-separated** columns: `NR`, `$2`, `$3`.

**Output columns now:**
`col1=NR` `col2=exDir` `col3=file` (tab-separated)

---

## 3) Drop the first 5 rows (after sort)

```bash
… | awk 'NR>5'
```

* New `awk` process → its own `NR` starts at 1 **here**.
* `NR>5`: passes only rows **6, 7, …**. This trims the top of the sorted list.

**Why here?** You usually demo “skip header/top chunk” *after* you’ve seen the list.

---

## 4) Clean `exDir`: remove the `exNN_` prefix

```bash
… | awk -F'\t' 'BEGIN{OFS="\t"} {sub(/^ex[0-9]+_/,"",$2); print $1,$2,$3}'
```

* `-F'\t'`: we split **the previous tabbed output**.
* `BEGIN{OFS="\t"}`: ensure `print` joins fields with tabs.
* `sub(/^ex[0-9]+_/,"",$2)`: in **column 2** (exDir), delete a leading `exNN_`.

  * `^` = start of string; `[0-9]+` = one or more digits; `_` = underscore.
  * Example: `ex03_1crn_A` → `1crn_A`.
* `print $1,$2,$3`: keep the same three columns, with cleaned col2.

**Output columns (tabbed):** `rowNumber` `exDirClean` `file`

---

## 5) Keep only **per-chain** files

```bash
… | awk -F'\t' '$3 ~ /chain/'
```

* `-F'\t'`: still tabbed input.
* `$3 ~ /chain/`: regex match—keep rows whose **filename** contains `chain`.

  * Passes things like `seq_1crn_chainA.fasta`.
  * Drops single-sequence entries like `1crn.fasta` or combined `A+B` files unless “chain” is in the name.

---

## 6) Renumber after filtering; keep just `exDir` and `file`

```bash
… | awk -F'\t' 'BEGIN{OFS="\t"} {print n++,$2,$3}'
```

* `n++`: **post-increment** → prints 0,1,2,… (If you want 1-based, use `++n`.)
* We intentionally **drop the old rowNumber** (col1) and emit a new one.
* Output cols: `newIndex` (0-based) `exDirClean` `file`.

---

## 7) Construct a full path column

```bash
… | awk -F'\t' 'BEGIN{OFS="\t"} {print $1,$2,$3,"examples/"$2"/"$3}'
```

* Build `path = examples/<exDirClean>/<file>`.
* Concatenation in AWK is just adjacency: `"examples/" $2 "/" $3`.
* Output cols: `newIndex` `exDirClean` `file` `path`.

---

## 8) Drop combined-chain dirs (e.g., `A+B`)

```bash
… | awk -F'\t' '$2 !~ /\+/'
```

* `!~` = **does not match**.
* `/\+/` = literal `+` (escaped; otherwise `+` means “one or more of previous”).
* We test **column 2** (the cleaned exDir). Any dir like `2zta_A+B` is excluded.
* This leaves **single-chain** entries only (e.g., `1crn_A`, `1yrf_A`, `1a3n_B`).

---

## Why this is solid (and what could break)

* Deterministic order via `sort` → reproducible output.
* We **never parse free-form**; we define field separators (`/` then `\t`).
* `sub()` is **anchored** (`^`), so it won’t delete mid-string patterns.
* Filtering by filename (`/chain/`) is explicit and readable for non-CS folks.
* The final `!~ /\+/` removes multi-chain directories cleanly.

**Edge cases (not present here, but worth knowing):**

* Paths with tabs or newlines would break tab-based splitting. (Rare; if you ever expect those, you’d use NUL-separated pipelines.)
* Deeper directory trees would make `$2,$3` assumptions unsafe. Here your layout is fixed (`examples/<exDir>/<file>`), so it’s fine.

---
