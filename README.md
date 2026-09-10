# Hybrid vote-fare to patch FPtP voting systems

> *"Coordinate without a common ground, and leave the agreements to your representatives."*

**A single HTML file that helps voters in First-Past-the-Post democracies coordinate tactically — without agreeing on anything except the method.**

---

## The problem in one paragraph

In FPTP systems, the candidate with the most votes wins each constituency — not a majority, just a plurality. When three or four opposition candidates split the vote, a ruling party candidate can win with 30–35% of the vote. Multiplied across hundreds of constituencies, this produces governments that most voters did not choose. And once in power, a rational government has every incentive to keep the opposition fragmented: culture wars, identity politics, and structural rules that make coordination harder with every election cycle.

---

## The breakthrough: constitutional continuity

Every other response to this problem — protests, boycotts, radical movements — operates *outside* the system, which gives the system's defenders something to push back against. A protest can be dispersed. A boycott can be ignored. An emigrant is gone.

**This tool works entirely inside the existing rules.** It uses the ballot box — the system's own mechanism — to correct the system's own distortion. There is nothing to ban. There is no one to arrest. There is no argument about legitimacy, because voters are simply voting.
![FPTP Coordination Illustration](fptp_illustration.svg)
This is change through constitutional continuity. It is radical in effect and conservative in method.

---

## How it works

The tool asks for three inputs:

1. **Constituency number** — publicly available before any election
2. **Opposition candidates with an estimated popularity percentage** — a previous election result or any honest estimate; the tool quantizes it into a coarse bucket, so nearby estimates produce identical results
3. **A seed: the opening value of the country's main stock exchange index on the first trading day of the week before election day** (normally Monday; if that is a holiday, the next trading day) — published in every newspaper, announced on every radio station, available without internet access, and controlled by nobody. **The value is rounded to a whole number**, so formatting and rounding differences between sources do not matter: `45123.78`, `45 123,78` and `45124` are all the same seed.

From these, it computes a single recommendation:

**Step 1 — Normalize candidate names**

Each name is normalized so that any reasonable spelling produces the same string on every device:

1. Unicode NFD decomposition, then strip combining diacritics (`Á→A`, `ő→o`, `ç→c`)
2. Explicit mapping for letters that do not decompose (`Ł→L`, `ß→SS`, `Đ→D`, `Ø→O`, `Æ→AE`, …)
3. Uppercase
4. Strip every character that is not `A–Z` or `0–9` (spaces, hyphens, apostrophes, dots)

```
"Gergely Karácsony"  →  GERGELYKARACSONY
"gergely karacsony"  →  GERGELYKARACSONY
"O'Brien-Smith"      →  OBRIENSMITH
"Łukasz Żółć"        →  LUKASZZOLC
```

For non-Latin scripts, the convention is: use the official Latin transliteration as printed on the ballot paper.

**Step 2 — Sort deterministically and index**

The normalized names are sorted by **plain code-point comparison** — never by locale-aware collation, whose result depends on the device's language settings. Each name gets a 1-based index in this order.

If two names normalize to the same string, the tool asks for the candidates' **official ballot paper position numbers** and uses them to break the tie. The ballot numbers then appear in the verification mapping.

**Step 3 — Quantize popularity into weight buckets**

The entered percentage is snapped to a logarithmic codebook:

```
%:       1 | 2 | 3 | 4–6 | 7–9 | 10–14 | 15–22 | 23–32 | 33–48 | 49–100
weight:  1 | 2 | 3 |  5  |  8  |  12   |  18   |  27   |  40   |  60
```

The bucket boundaries are the geometric means of neighbouring codebook values, precomputed as integer thresholds — no floating-point arithmetic, identical on every device. Win probability depends only on weight *ratios*, so logarithmic quantization keeps the error uniform on the relative scale, and 33%, 35% and 36% all map to the same weight. Each candidate's bucket depends only on their own percentage: a dispute about one party never shifts anyone else's weight.

**Step 4 — The hash lottery**

```
winner = max( SHA-256( constituency | name_index | salt | seed ) )
         for each candidate, salt ∈ [0, weight)
```

Each candidate gets `weight` lottery tickets; the candidate holding the globally largest hash is the coordinated choice. The probability of winning is exactly `weight / total_weight` — proportional, so smaller parties retain a real chance, and over many constituencies the proportionality plays out fairly.

**Step 5 — Compare fingerprints, then verify**

The tool displays an **input fingerprint**: the first 8 hex characters of a SHA-256 over the full canonical input set (constituency, sorted normalized names, weights, ballot numbers, seed). Two people in the same constituency only need to compare this short code — `A3F9C2E1` — to confirm they entered identical data. A mismatch means someone mistyped something, and it surfaces *before* election day.

The full result can be reproduced from scratch with nothing but a terminal:

```bash
echo -n "CONSTITUENCY|NAME_INDEX|BEST_SALT|SEED" | sha256sum
```

The tool displays the complete name→index→weight mapping after every calculation. The result is not deterministic until the seed value is known; once known, it is identical for everyone using the tool, everywhere.

---

## Key properties

| Property | Detail |
|----------|--------|
| **No ideology required** | Voters who agree on nothing except their dissatisfaction can use it together |
| **Lawful and unassailable** | Works inside the electoral system using its own rules |
| **Self-activating** | Spreads when genuinely needed; dormant otherwise |
| **Self-correcting** | Can be turned against any entrenched government, including a future one that disappoints |
| **Probabilistically fair** | Win probability is exactly proportional to weight; smaller parties retain a real chance |
| **Robust to disagreement** | Logarithmic weight buckets absorb differing popularity estimates; normalization absorbs typos and accents |
| **Locale-independent** | Code-point sorting and integer thresholds — the same result on every device, browser and language setting |
| **Cross-checkable** | The 8-character input fingerprint lets any two people confirm matching inputs in seconds |
| **Independently verifiable** | Anyone can reproduce the result with `echo -n "..." \| sha256sum` |
| **Offline and untraceable** | Single HTML file; no server, no account, no network required |
| **Statistically robust** | Doesn't require universal adoption — only a critical mass per constituency |

---

## Verification

The result can be verified by anyone, independently, without this tool:

```bash
echo -n "CONSTITUENCY|NAME_INDEX|BEST_SALT|SEED" | sha256sum
```

Where `NAME_INDEX` is the candidate's position in the deterministically sorted list of normalized names, `BEST_SALT` is the salt in `[0, weight)` producing the candidate's largest hash, and `SEED` is the stock index opening value rounded to a whole number. The tool displays the full mapping after every calculation.

If a different version of this tool produces a different result for the same inputs, **it is provably tampered with.** This is the defence against fake forks: mathematical transparency makes central trust unnecessary.

> ⚠️ **Warning for forks:** Five things must never change: the hash algorithm (SHA-256), the field order (`constituency|name_index|salt|seed`), the name normalization and sorting method (NFD + diacritic strip + special-character map → uppercase → strip non-alphanumerics → code-point sort → ballot-number tiebreak), the weight codebook and its integer thresholds, and the seed rounding rule. Altering any of these silently fragments coordination in a way that is technically indistinguishable from a deliberate attack on the tool's purpose.

> ⚠️ **Version compatibility:** **v3.1 is the canonical reference and is NOT hash-compatible with earlier versions** (v2 and earlier used locale-dependent sorting, no accent normalization and raw popularity counts — all of which could fragment coordination across devices). Do not circulate older versions alongside v3.1.

---

## Files

| File | Description |
|------|-------------|
| `Distributed_loaded_dice.html` | The coordination tool (canonical version) — open in any browser, works offline |
| `Introduction.html` | Long-form explainer / landing page for non-technical readers |

---

## Where this applies

Any democracy using FPTP or a mixed system with FPTP constituencies:

**Full FPTP:** United Kingdom, India, Canada, USA, Bangladesh, Nigeria, Ghana, Pakistan, Jamaica, and others — approximately **1.5 billion voters**

**Mixed (FPTP constituencies + proportional lists):** Hungary, Germany, Japan, South Korea, Mexico, and others — several hundred million more

Not applicable in proportional systems (Netherlands, Scandinavia) or two-round systems (France), where the coordination problem is handled differently by the electoral architecture itself.

---

## The self-correction mechanism

The tool has no loyalty to any side. If the coalition that wins through this method eventually becomes complacent or self-serving, it too becomes "the entrenched minority government" — and the same method can be used against it. No rupture required. The next election comes, the same tool applies, the outcome corrects.

This is not a tool for the left or the right. It is a tool for the outvoted.

---

## Usage

1. Download `Distributed_loaded_dice.html`
2. Open it in any browser — no internet required
3. Enter your constituency number, the opposition candidates with estimated popularity percentages, and the stock index opening value from the first trading day of the week before election day
4. **Compare your input fingerprint with friends in the same constituency** — a matching code means matching inputs
5. Follow the recommendation
6. Share the file — over Bluetooth, USB, QR code, Signal, anything

---

## Licence

MIT — free to use, study, modify, and distribute. The only informal constraint: if you fork this, **do not change the hash algorithm, the hash input format, the normalization/sorting rules, the weight codebook, or the seed rounding rule.** Doing so silently fragments coordination in a way that is technically indistinguishable from a deliberate attack on the tool's purpose.

---

## Background

This project emerged from a conversation about the structural mechanics of FPTP systems — why they systematically reward fragmentation, how incumbents exploit that mathematically, and why every traditional response (coalitions, protests, boycotts) has failed to solve the underlying coordination problem. The tool is the result of asking: what is the minimum intervention that breaks the coordination failure, requires no trust between parties, and operates entirely within the law?

The answer turned out to be a few hundred lines of HTML.
