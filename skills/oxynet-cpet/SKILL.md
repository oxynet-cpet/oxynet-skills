---
name: oxynet-cpet
description: Analyse cardiopulmonary exercise test (CPET) files with Oxynet's hosted API. Use when the user has a metabolic cart export (COSMED, Cortex, PNOE, VO2 Master and other formats, as CSV, XLSX, TXT, XML or JSON) or asks for ventilatory thresholds (VT1, VT2, GET, RCP), VO2max, the VE/VCO2 slope, FATMAX or fat and carbohydrate oxidation, exercise oscillatory ventilation (beta), or the signal quality of a CPET recording. Oxynet's validated models compute the physiology; do not estimate thresholds from the trace yourself. Needs an Oxynet API key in OXYNET_API_KEY.
license: Apache-2.0
---

# CPET analysis with Oxynet

Oxynet is a hosted computation service for cardiopulmonary exercise tests. You send it
the vendor file, it returns a handle (`cpet_id`), and you ask for analyses by name. You
orchestrate and explain; Oxynet's validated models decide the physiology.

If an Oxynet MCP connector is already available in this session (tools such as
`analyze_vt`), use it instead and ignore the rest of this file.

Every command below is self-contained, because shell variables may not survive between
your commands. `OXYNET_API_URL` is only for testing against another server; leave it unset.

## 1. The key

If `OXYNET_API_KEY` is not set, ask the user to make it available as that environment
variable (for example in their shell profile) rather than pasting it into the
conversation. If they paste it anyway, use it for this session only: never write it into a
file and never repeat it back. Someone without a key can get a free one at
https://app.oxynet.net/signup (it takes a minute).

Check what the key may do, and tell the user which analyses it includes:

```bash
curl -s -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" \
  "${OXYNET_API_URL:-https://app.oxynet.net}/v1/capabilities"
```

## 2. Rules

1. **Upload the file exactly as exported.** Do not open it, parse it, rename columns or
   convert units. Oxynet detects the format and knows each one's units and clock quirks;
   converting first is how information gets lost.
2. **Only upload a file the user named.** Never search their disk for CPET files. A CPET is
   health data and exports often carry the patient's name. Before the first upload, say
   once: Oxynet needs no patient identifiers, so anonymise a name or date of birth first.
3. **Work by handle.** Never read the recording into the conversation. Fetch a series only
   to draw a picture, never to compute something Oxynet can compute.
4. **Ask for the smallest thing that answers the question.** One analysis you need beats
   three you are curious about. When the user wants everything, one `assess` call gives
   the full answer (section 4).
5. **Leave the model to Oxynet.** Name one only when the user asks for it. Unnamed, Oxynet
   answers with the best available method for the recording and says which, and why, in
   `provenance.selection`.
6. **A refusal is the answer.** `not_analysable`, `unavailable` or an error code states
   why; report that reason instead of working around it. The one exception is
   `UNSUPPORTED_FORMAT`, whose `recovery` block says exactly what conversion is allowed.
   The upload's `preprocessing.checks` announce a refusal before any analysis runs: tell
   the user then.
7. **There is no confidence score, only a confidence category.** Nothing is calibrated
   against clinical outcomes, so never give or invent a number. Each result's
   `confidence.level` is "low" or "normal", set by stated rules: when it is "low", say so
   and relay what its `reasons` say. "normal" means no rule fired, not that the answer is
   certain. `quality` describes the recording, not the certainty of the answer.
8. **Oscillatory ventilation (`eov`) is beta**: developed on one heart failure cohort,
   transportability untested. Say so whenever you report it.
9. **Research software, not a diagnosis.** Report measurements, not clinical conclusions.

## 3. Upload

One file:

```bash
curl -s -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" \
  -F "file=@path/to/export.xlsx" \
  "${OXYNET_API_URL:-https://app.oxynet.net}/v1/cpet" \
  | jq 'if .error then . else {cpet_id, source, flags, checks: .preprocessing.checks} end'
```

`checks` lists what Oxynet found in the recording and, for each analysis it affects,
whether that analysis will be `refused` or only needs `caution`. Read it before analysing.

Several files, in one command, printing each file's handle or refusal code, then the code
of each check on it:

```bash
for f in path/to/folder/*.csv; do
  printf '%s\t' "$f"
  curl -s -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" -F "file=@$f" \
    "${OXYNET_API_URL:-https://app.oxynet.net}/v1/cpet" \
    | jq -r '[.cpet_id // .error] + [.preprocessing.checks[]?.code] | join("\t")'
done
```

To demonstrate Oxynet without patient data, upload the synthetic sample instead:

```bash
curl -s "${OXYNET_API_URL:-https://app.oxynet.net}/v1/sample" \
  | jq '{filename, content: (.content | tojson)}' \
  | curl -s -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" \
      -H 'Content-Type: application/json' -d @- \
      "${OXYNET_API_URL:-https://app.oxynet.net}/v1/cpet/content" | jq -r .cpet_id
```

## 4. Analyse

Everything Oxynet can say about a recording, in one call:

```bash
curl -s -X POST -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" \
  "${OXYNET_API_URL:-https://app.oxynet.net}/v1/cpet/CPET_ID/assess"
```

`assess` runs every analysis the key includes that the recording supports, with the
derived metrics, and returns a `headline` (the main numbers, with each part's confidence
level; the reasons are in that part's result), a `plan` (every part and, when it did not
run, why) and the full `results`. Each analysis that ran counts one unit of the key's
monthly allowance; the metrics add nothing, and a refusal is free. For everything, it
costs less than the analyses and metrics asked for one by one.

One analysis, when that is the question:

```bash
curl -s -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" \
  -H 'Content-Type: application/json' -d '{"analyses": ["vt"]}' \
  "${OXYNET_API_URL:-https://app.oxynet.net}/v1/cpet/CPET_ID/analyze"
```

`analyses` takes `vt` (ventilatory thresholds VT1 and VT2), `eov` (oscillatory ventilation,
beta) and `substrate` (fat and carbohydrate oxidation, FATMAX, the crossover). Each
analysis counts one unit; a refusal is free. Each result, from either call, carries
`status`, `quality`, `confidence`, `notes`, `provenance` and `preprocessing` (what the
analysis read and which of the recording's checks applied) beside its `findings`.

A single number goes to `compute`: `substrate_summary` answers "what is their FATMAX" in a
fraction of the full analysis's output, and `gas_quality`, `sampling_adequacy`,
`clock_integrity` and `phases` describe the recording itself. A `compute` call counts one
unit, like an analysis: what it saves is tokens, not units.

```bash
curl -s -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" \
  -H 'Content-Type: application/json' -d '{"metrics": ["substrate_summary"]}' \
  "${OXYNET_API_URL:-https://app.oxynet.net}/v1/cpet/CPET_ID/compute"
```

`GET /v1/metrics` lists every metric with its units and requirements.

## 5. Report

Give the values Oxynet returned, with their units, and quote `notes` when they qualify a
result. Relay a "low" confidence with its reasons. Say which analyses were refused or did
not run, and why (an assessment's `plan` says). For a cohort, build the comparison table
yourself from the returned numbers: that is your job; deciding where a threshold is, or
whether an oscillation is real, is not.

Records are deleted after 24 hours. If the user wants one gone sooner:
`curl -s -X DELETE -A oxynet-skill/1 -H "X-API-Key: $OXYNET_API_KEY" "${OXYNET_API_URL:-https://app.oxynet.net}/v1/cpet/CPET_ID"`.

## More

Everything else (fields, error codes, formats, upload tickets, series for plotting) is in
https://app.oxynet.net/llms-full.txt. Read the relevant section rather than guessing an
endpoint or a field name.
