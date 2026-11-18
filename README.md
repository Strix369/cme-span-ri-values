# CME SPAN RI VALUES

A portable, self-contained skill by **RANX** for researching NQ/MNQ and ES/MES Risk Intervals from official CME margins.

The skill reads the four CME margin pages, checks contracts and input fields, applies a fixed candidate equation, and saves forecasts separately from observed results. It keeps missing data, late captures and misses visible.

**Research status:** the equation is inferred and has known mismatches. The provider's precise input selection and publication cutoff remain unresolved. This is not an official CME product or trading advice, and it does not guarantee future RI values.

## Repository layout

```text
cme-span-ri-values/
├── SKILL.md       # Complete instructions, calculation code and record workflow
├── skill.yaml     # Name, version, author, compatibility and tags
├── README.md      # Installation and usage
└── .gitignore     # Keeps generated research and machine-local files out of Git
```

No private research, account state, screenshots or credentials are required or included. Generated records stay in your own local `research/` folder. The repository contains the skill workflow only.

## Clone or download

In GitHub Desktop, choose **File → Clone repository**, enter:

```text
https://github.com/Strix369/cme-span-ri-values
```

Alternatively, use **Code → Download ZIP** on GitHub and extract it. With Git installed:

```sh
git clone https://github.com/Strix369/cme-span-ri-values.git
```

## Install

The full skill is in [SKILL.md](SKILL.md). It does not need the author's folders or earlier conversations.

For a Codex project, create `.codex/skills/cme-span-ri-values/` and copy `SKILL.md` into that folder. Keep the README and metadata in your downloaded repository for reference. Codex loads `SKILL.md`; `skill.yaml` is descriptive distribution metadata, not a scheduler or installer.

For another compatible agent, provide SKILL.md as its skill instructions and select a local working project. The agent must have a documented built-in browser, local file access and an exact calculation runtime. The embedded Python uses Python 3.9 or newer with working IANA timezone data. If the runtime cannot resolve `America/New_York`, supply timezone data before calculating session dates. No additional platform integration is part of this repository.

## Use

After installation in Codex, invoke:

```text
$cme-span-ri-values
```

Or ask:

```text
Give me the CME SPAN RI values for the upcoming Globex opening.
```

For a named date:

```text
Run CME SPAN RI VALUES for the 2026-11-18 trading session.
```

Report an observation for comparison:

```text
For the 2026-11-18 session, the observed RI was NQ 283.00 and ES 72.50.
```

Normal replies are short:

```text
NQ / MNQ: value
ES / MES: value
```

The skill labels an inferred forecast, a post-open calculation or an unavailable source when needed. Historical examples are not today's inputs. If no valid live input or authoritative saved value exists, it reports `Unavailable` instead of inventing a value.

## Method

For maintenance-long margin `M` in USD and contract value `V` in USD per index point:

```text
raw_RI = 11 * M / (80 * V)
RI = floor(11 * M / (20 * V)) / 4
```

NQ uses $20 per point; ES uses $50. MNQ and MES use $2 and $5 and provide consistency checks. The E-mini calculation defines each family. Settlement prices provide dated context and do not enter the equation. Read the source-selection, freshness and rounding rules in SKILL.md before using a result.

## Local records and timing

The agent creates or updates:

```text
research/RI_YYYY-MM-DD_prediction.json
research/RI_YYYY-MM-DD_prediction.md
research/RI_YYYY-MM-DD_result.json
research/RI_YYYY-MM-DD_result.md
```

The date is the Globex trading date, not simply the local calendar date. Forecast revisions preserve earlier snapshots and hashes. Observed results remain separate. Post-open calculations never receive a pre-open score. The ignored `research/` folder is created only when you run the skill.

Installation does not schedule anything. If a user requests the RANX schedule, its fixed default is 04:00 IST Tuesday through Saturday, with no Sunday or Monday run. The Saturday capture prepares Sunday's opening and uses Monday's trading date. Change that fixed Indian time manually when required. Always verify CME holiday hours and assess the actual capture time in New York.

## Protect local material

The supplied .gitignore excludes generated data, research, common credential files, environment files, logs, OS files and caches. It does not detect every possible secret and does not remove files already tracked by Git. Review **Changes** in GitHub Desktop before each commit. Publish only files you intend to share. Never paste credentials into a skill prompt or record.

## Reuse

RANX provides permission to clone, download and use this skill workflow. Preserve attribution when sharing it. CME data and trademarks remain subject to their respective owners' terms. This repository does not include a CME data license or permission to redistribute private source material.

Passing software checks verifies arithmetic and workflow behavior. It does not validate the inferred equation's future accuracy. Keep prospective comparisons, including misses, before treating a method as reliable.
