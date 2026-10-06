# Run Quick Design Calculator

## Overview

Run any calculator from the SkyCiv Quick Design library via a single REST endpoint.
The library covers 154 calculators across structural, foundation, steel, concrete, timber, aluminium, connection, and load categories.

- **API endpoint:** `POST https://qd.skyciv.com/run`
- **Auth:** Requires a SkyCiv API token — get yours at https://platform.skyciv.com/api
- **[Full calculator catalogue](./assets/catalogue.md)**
- **[S3D integration](#how-to-integrate-a-calc-pack-with-s3d)**: make a calc pack run inside S3D against every member of a solved model (`s3d_integration.js`, the hidden `analysis_results` input, `hide_in_s3d`)

---

## Before you pick a member design calculator: check S3D Member Design first

> **Preferred method:** to design members of an S3D model (steel, cold-formed steel, timber), use [`S3D.design.member.check`](../s3d-api/SKILL.md#s3ddesign-functions) when it supports that code. Don't use the Quick Design member calculator for the same code. For example, for AS/NZS 4600 use `S3D.design.member.check` with `design_code: "ASNZS_4600-2018"`, not `2304-as4600-cfs-design-calculator`.

**Why:** Member Design runs on the whole solved model. It handles every load combination, effective lengths, bracing/restraints, and member grouping automatically, and usually gives a better design result. A Quick Design calculator only sees the forces and lengths you pass in by hand. Quick Design calculators are still accurate and are the right tool when Member Design doesn't cover the job.

**Decide like this:**
1. Is it a **member** check (beam/column/brace) on an **S3D model**, and is the code in the [Member Design list](../s3d-api/SKILL.md#supported-member-design_code-values)? → use **`S3D.design.member.check`**. This applies to steel, cold-formed steel and timber only.
2. Otherwise, use a Quick Design calculator. That covers:
   - **all concrete design** (beams, columns, slabs, walls, footings). Always use the Quick Design concrete calculators here, never `S3D.design.rc.check`;
   - codes Member Design doesn't have: all aluminium, CSA O86, EN 1995, NZS 1720;
   - an edition the user explicitly requires that only Quick Design has (AISC 360-22, CSA S16-24);
   - connections, base plates, plates, lugs, purlins, foundations, and loads;
   - a single hand check with no model;
   - building an S3D calc pack (below).

The Quick Design calc → Member Design `design_code` mapping is in [`s3d-api`](../s3d-api/SKILL.md#member-design-vs-quick-design-which-to-use).

---

## How to call

```js
const axios = require('axios');

axios.post('https://qd.skyciv.com/run', {
    payload: JSON.stringify({
        uid: "8004-as3600-strip-footing-design", // calculator UID from the catalogue
        auth: "you@example.com",                  // authenticated email address
        key: "YOUR_API_TOKEN",                    // from https://platform.skyciv.com/api
        pdf_report: true,                         // set false to skip PDF generation
        input: {
            // Input object matching the calculator's schema.json
            // See sample_input.json for a ready-to-use example
        }
    })
})
.then(response => console.log(response.data))
.catch(error => console.error(error));
```

### Request payload fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uid` | string | yes | Calculator unique identifier (see [catalogue](./assets/catalogue.md)) |
| `auth` | string | yes | Authenticated user email address |
| `key` | string | yes | API token from https://platform.skyciv.com/api |
| `pdf_report` | boolean | no | Generate a PDF report (default: `false`) |
| `input` | object | yes | Input parameters — see each calculator's `schema.json` |

---

## Response format

```json
{
  "status": 0,
  "data": {
    "report": "https://pdf.skyciv.com/...  (PDF link, valid for 1 hour)",
    "results": {
      "Utilization Ratio": {
        "value": 0.85,
        "info": "Ratio of demand to capacity",
        "units": "utility",
        "label": "Utilization Ratio"
      },
      "Design Check": {
        "value": "PASS",
        "units": "custom_box",
        "label": "Design Check",
        "color": "#21BA45"
      },
      "Footing Width": {
        "value": 1200,
        "info": "Calculated footing width",
        "units": "mm",
        "label": "Footing Width"
      }
    }
  },
  "log": "",
  "warnings": [],
  "msg": "Design Calc ran successfully"
}
```

### Status codes

| `status` | Meaning |
|-----------|---------|
| `0` | Success |
| `1` | Error — check `msg` and `log` for details |

### Result `units` field meanings

| `units` value | Meaning |
|----------------|---------|
| `"heading"` | Section heading — no `value` field |
| `"utility"` | Utilization ratio — pass if `value ≤ 1.0` |
| `"utility_boolean"` | Boolean pass/fail — `0` = pass, `1` = fail |
| `"custom_box"` | Pass/fail text — `value` is `"PASS"` or `"FAIL"` |
| any other string | Physical unit (e.g. `"mm"`, `"kN"`, `"MPa"`, `"kPa"`) |

---

## Troubleshooting

**A call fails with `status: 1`, a generic `msg` like `"Failed to run."`, and an `err` like `"Cannot read properties of undefined (reading 'length')"` (or similar low-level JS error), even though every field matches the calculator's `schema.json`:**
This is not a always request-shape bug — it could mean one of your *values* falls outside what that calculator's underlying material/section/species database actually has data for, and the backend crashes on the missing lookup instead of returning a clear validation error. Material/alloy/species/grade `enum`s in `schema.json` are frequently stubs (e.g. `2601-aluminium-design`'s `alloy` enum shows only `[1100]` as if that were the only option) — the live calculator accepts a much broader real set, but only specific (material, temper/grade) *pairs* have actual data rows, and there is no way to know which pairs are valid from the schema alone. Confirmed directly against the live API for `2601-aluminium-design`: `alloy: 6063, temper: "T5"` (a common architectural aluminium alloy/temper) crashes every call this way, while `alloy: 6061, temper: "T6"` and `alloy: 5052, temper: "H32"` both work. If you hit this, bisect by reverting to the calculator's own `sample_input.json` (known-good) and changing one field at a time against the live API until you find which value is unsupported — don't assume it's a bug in your request construction.

**`sample_output.json` often doesn't reflect the calculator's real result keys:** some of these files are generic placeholder boilerplate (e.g. a "Design Summary" heading + one generic "Utilization Ratio" + one "Example Dimension" — the exact same three entries appear verbatim across multiple unrelated calculators' `sample_output.json`), not an actual captured response for that specific calculator. Treat `schema.json`/`sample_input.json` as authoritative for input field names/units; only trust `sample_output.json`'s result *key names* if they look calculator-specific (not this generic pattern) — otherwise expect the real `results` object to have different, calculator-specific keys, and code defensively (e.g. scan for `units: "utility"` entries rather than a fixed key name — see `extractGoverningUtilization`-style logic in the prototype apps under `prototypes/`).

**Some result entries carry `"paid_only": true`:** this has been observed on `2601-aluminium-design`'s combined-action checks (e.g. "Combined Tension + Bending", "Combined Bending and Shear"). A value is still present, but on a free/trial account it may not reflect a genuinely computed result. If you're scanning for the governing (largest) `units: "utility"` value across all entries, be aware a `paid_only` entry could be silently governing the result on an account that hasn't verified it actually computes real values for that entry.

## Per-calculator assets

Each calculator folder under `assets/<uid>/` contains:

| File | Description |
|------|-------------|
| `schema.json` | JSON Schema for the `input` object |
| `sample_input.json` | Ready-to-use example input |
| `sample_output.json` | Example API response |

Browse the [full catalogue](./assets/catalogue.md) to find the right calculator.

---

## Regenerating this skill

```bash
node skills/compile.js            # all calculators
node skills/compile.js --public   # public calculators only
```

# How to Batch Call a single calculator

If you need to run a single calculator multiple times with different inputs, you can use the `runBatch` endpoint.

This is more efficient than calling the `run` endpoint multiple times, and should be used if you need to optimise a run. For example, if you are trying to find the optimal depth, you can batchRun the same calculator with different depths, and find the optimal depth that way:

```js

//BATCH RUN
let how_many_to_test = 50;
let input_batch = [
    {
        // Input object matching the calculator's schema.json
        // See sample_input.json for a ready-to-use example
    },
    {
        // Input object matching the calculator's schema.json
        // See sample_input.json for a ready-to-use example
    },
]

fetch('https://qd.skyciv.com/runBatch', {
    method: "POST",
    headers: {
        "Content-Type": "application/json"
    },
    body: JSON.stringify({
        payload: JSON.stringify({
            uid: "2015-as4100-i-beam-capacity-calculator",
            auth: "SKYCIV_USERNAME_HERE",
            key: "SKYCIV_KEY_HERE",
            input_arr: input_batch,
            pdf_report: true,
        })
    })
})
.then(response => response.json())
.then(data => {
    console.log(data);
});

```

The result object will come back as an array of results, with the same order as the input batch. So if you send in 50 inputs, you will get back an array of 50 results, which you can then iterate through to find the optimal depth.

# How to open a file you've just created

You can also allow the user to open the file by appending the parameters after the URL. For example:

https://platform.skyciv.com/quick-design?uid=3027-au-concrete-column&member_label=C1&shape=rectangular&D=400&W=300&cover=30&size_bars=20&n_bars_z=4&n_bars_y=4&reinforcement_class_long=N&size_shear_bars=8&n_shear_bars_y=2&n_shear_bars_z=2&s=150&reinforcement_class_shear=N&L=3000&k_y=1&k_z=1&f_c=40&f_y=500&V_y=100&V_z=50&N=1350&G=500&Q=500&second_order=first&braced_or_unbraced=ZY&M_z_top=100&M_z_bot=50&M_y_top=50&M_y_bot=-50&member_label=C1&shape=rectangular&D=401&W=301&cover=30&size_bars=20&n_bars_z=4&n_bars_y=4&reinforcement_class_long=N&size_shear_bars=8&n_shear_bars_y=2&n_shear_bars_z=2&s=150&reinforcement_class_shear=N&L=3000&k_y=1&k_z=1&f_c=40&f_y=500&V_y=100&V_z=50&N=1350&G=500&Q=500&second_order=first&braced_or_unbraced=ZY&M_z_top=100&M_z_bot=50&M_y_top=50&M_y_bot=-50

So it would be helpful if you provide these links after you run the calculation, so the user can open these designs up.

---

# How to integrate a calc pack with S3D

How to make a Quick Design calc pack run inside S3D, against every member of a solved model. You write a
`s3d_integration.js` file that turns the model and its analysis results into an array of calculator inputs, one per
member. The platform then runs `calculate.js` once per input and shows the results in the S3D Quick Design panel.

> **Prerequisite:** an existing calc pack (`config.json` + `calculate.js`, optional `ui.js` / `utils/`) that already
> runs as a standalone Quick Design calculator. This guide only adds the S3D side. It reads the same
> `s3d_model` schema as [`s3d-api`](../s3d-api/SKILL.md) (but in the **UI format**, see below) and the results
> described in [`analysis-results`](../analysis-results/SKILL.md). Nothing in this section uses the REST endpoint
> above.

Working examples on the platform: `2603-csa-aluminium-design` (CSA S157 aluminium) and
`2401-as3990-mechanical-steel-design` (AS 3990 steel, with per-station end and span moments).

---

## The three pieces

A pack is S3D integrated when all three of these are in place. **Skipping the second or third is the common gap:**
without them, S3D shows the manual load inputs (or a load entry table) that the user should not be editing, and
`calculate.js` may run off those defaults instead of the S3D results.

| Piece | File | What it does |
|---|---|---|
| 1. Integration | `s3d_integration.js` | Reads `s3d_model` + `analysis_results`, returns one input object per member. Puts every load combination's actions into a single JSON string field, `analysis_results`. |
| 2. Hidden S3D input | `config.json` | Declares `analysis_results` as a `custom_object` input that is hidden in both Quick Design and S3D. Marks the manual load inputs, and every input the integration fills from the model, `hide_in_s3d`. Sets `meta.s3d_integrated: true`. |
| 3. Load source switch | `calculate.js` | If `input_json.analysis_results` is present, uses it as the load cases; otherwise uses the manual load inputs / table. The standalone calculator is unchanged. |

---

## 1. `s3d_integration.js`

```js
module.exports = function (s3d_model, analysis_results) {
	if (!analysis_results) throw new Error("No Analysis Results. Please solve model first.");
	// ... build inputs ...
	return design_members; // array of input objects; index = member id (sparse is fine)
};
```

| Argument | In the S3D console | Notes |
|---|---|---|
| `s3d_model` | `S3D.model_data` | **UI format**: `s3d_model.elements`, not `members`. |
| `analysis_results` | `S3D.results.getAll(true)` | **Legacy array**, not the API object: indexed by load combination, entries can be `null`, each has `name` and `type`. |

Throw an `Error` for anything that stops the whole run (no results, no members imported). Its message is shown to the
user, so list why members were skipped.

### Platform globals

| Global | Use |
|---|---|
| `StructureHelpers.ezDesignForces(analysis_results, s3d_model, action_list, include_lc)` | Actions per member per load combination. Use `include_lc = true` (detailed): per-station `values` about the **section builder axes**, so section rotations / mirrors are handled. `include_lc = false` (express) gives envelope `max` / `min` only and throws if any section is transformed. |
| `StructureHelpers.getDesignForces(analysis_results)` | Older helper: worst + / - of each action over all combinations, per member. |
| `StructureHelpers.getMemberLength(s3d_model, member_id, consider_offsets)` | Member length, in `settings.units.length`. |
| `UnitHelpers.convert(value, from, to)` | Unit conversion, e.g. `UnitHelpers.convert(L, units.length, "mm")`. |
| `logger(message)` | Writes to the Quick Design logs. Log each skipped member and why. |
| `warn(message)` | Shows a warning to the user (e.g. "loads are unfactored"). |

`ezDesignForces` detailed output: `ez[member_id][lc_index][action] = { values: [...], max, min, abs_max }`, where
`lc_index` is the index into `analysis_results`. Actions: `axial`, `bmd_y`, `bmd_z`, `sfd_y`, `sfd_z`, `torsion`.
`values` runs from end 1 (first) to end 2 (last). Flatten it defensively: the API format stores a discontinuity
station as a `[left, right]` pair, so treat any array entry as two values.

### Units

Results and dimensions are **never** in fixed units; they follow `s3d_model.settings.units`. Convert everything to the
units the calculator's `config.json` inputs use.

| Quantity | Units key | Example convert |
|---|---|---|
| Member length | `units.length` | `UnitHelpers.convert(L, units.length, "mm")` |
| Section dimensions, fillet radius | `units.section_length` | `UnitHelpers.convert(dim, units.section_length, "mm")` |
| Forces | `units.force` | `UnitHelpers.convert(V, units.force, "kN")` |
| Moments | `units.moment` | `UnitHelpers.convert(M, units.moment, "kN-m")` |
| Material strength | `units.material_strength` | `UnitHelpers.convert(Fy, units.material_strength \|\| "MPa", "MPa")` |

### Choosing load combinations

| `analysis_results[j].type` | Use? |
|---|---|
| `user_defined` | Yes: the combinations to design for. |
| `load_case`, `load_group` | Only as a fallback when there are no `user_defined` combinations (`warn` the user). |
| `envelope` | No: envelopes mix combinations. `Envelope Min` / `Max` / `Absolute Max` all report `type: "envelope"`. |

Check what the Standard needs: a limit state code needs factored combinations, a working stress code (e.g. AS 3990)
needs unfactored service combinations. Say which with `warn()`.

### Sign conventions (S3D)

Checked against a solved model (a simply supported beam deflecting down, and a column under self weight):

| Action | Positive means |
|---|---|
| `axial` | Compression |
| `bmd_z` | Sagging: compression at the top of the section, as drawn in the section builder |

Compare these with the calculator's own conventions before passing signed values through. The sign of `bmd_y` has not
been confirmed; it only matters for sections that are not symmetric about Y (e.g. channels), so check one before
relying on it.

### Members, sections and materials (UI format)

| Path | Contents |
|---|---|
| `s3d_model.elements[i]` | Member `i`. `elements[i][2]` is the section id. Entries can be `null`. |
| `s3d_model.sections[id]` | `material_id`, and `aux` for a section builder section (no `aux` = numeric section, no shape). |
| `section.aux.polygons` | Shapes of the section. A hollow shape can have a second polygon with `cutout_parents` (its hole), so count only polygons without it. More than one solid polygon, or `aux.composite`, is a built-up section. |
| `polygon.shape` | Template shape name, see below. |
| `polygon.design[key]` | Dimension as a number (newer models). |
| `polygon.dimensions[key].value` | Same dimension (older models). Read `design` first, then fall back to this. |
| `polygon.operations` | `fillet_radius`, `rotation`, `mirror_y`, `mirror_z`. |
| `s3d_model.materials[id]` | `class` (`"steel"`, `"aluminium"`, ...), `yield_strength`, `ultimate_strength`. Aluminium packs also read `aux.selections` (standard, alloy, temper). |

Dimension keys by `polygon.shape` (from sample models):

| `shape` | Keys |
|---|---|
| `ibeam`, `channel` | `h`, `TFw`, `TFt`, `BFw`, `BFt`, `Wt` (check top = bottom flange if the calculator assumes equal flanges) |
| `tbeam` | `h`, `TFw`, `TFt`, `Wt` (table on top) |
| `hollow rectangle` | `h`, `b`, `t` (top / bottom walls), `tb` (side walls). Treat as square when `h = b` and `t = tb`. |
| `hollow circle` | `D`, `t` |
| `circle` | `D` |
| `rectangle` | `h`, `b` |
| `lbeam` | `h`, `BFw`, `BFt`, `LFt` |

Skip (and `logger()`) every member the calculator can't design: wrong material class, unsupported shape, built-up
section, no `aux`, or no design forces (rigid links return none).

---

## 2. `config.json`

| Key | Where | Purpose |
|---|---|---|
| `meta.s3d_integrated` | `meta` | `true`, so the calculator is listed in S3D. |
| `hide_in_s3d` | any input | Hides the input in the S3D panel only. It still shows in the standalone calculator. |
| `"type": "custom_object"` | the `analysis_results` input | Holds the S3D load data. It has no form field. |
| `hidden` | the `analysis_results` input | Also hides it in the standalone Quick Design module, where it is never used. |
| `exclude_from_input_table` | the `analysis_results` input | Keeps the raw JSON out of the report's input table. |
| `s3d_symbol`, `s3d_info` | results | Heading and info tip of a result column in the S3D Results table. |

The hidden S3D input:

```json
"analysis_results": {
	"type": "custom_object",
	"info": "Load cases imported from S3D by s3d_integration.js.",
	"hidden": true,
	"hide_in_s3d": true,
	"exclude_from_input_table": true
}
```

**What to mark `hide_in_s3d`:**

- **Always:** the manual load inputs (single values like `M_fz_input`, or a load entry `table`) and their heading.
  In S3D each member's loads are fixed by the analysis results.
- **Every input the integration sets per member from the model:** shape, section / grade dropdowns, dimensions, fillet
  radius, yield strengths, member label, section drawing `div`, and lengths taken from the member. The S3D panel
  shows one set of inputs for every member, so a visible per-member value would be the same for all of them.
  Hide a heading too when every input under it is hidden.
- **Leave visible:** design assumptions that apply to the whole run and that the model does not know, e.g.
  fabrication, restraint conditions, effective length factors, sidesway, report type.

Hiding lengths stops users entering shorter lengths for intermediate bracing in S3D. Decide this per pack.

---

## 3. `calculate.js`

Read the S3D loads first and fall back to the manual inputs, so one `calculate.js` serves both:

```js
function S3DLoadCases(input_json) { // null outside S3D
	let s3d = input_json.analysis_results;
	if (!s3d) return null;
	try {
		s3d = typeof s3d === "string" ? JSON.parse(s3d) : s3d;
	} catch (e) {
		logger("Problem importing analysis_results from S3D: " + e.message);
		return null;
	}
	let rows = Object.keys(s3d).map(name => Object.assign({ name: name }, s3d[name]));
	return rows.length ? rows : null;
}

let rows = S3DLoadCases(input_json) || input_json.load_cases_table || [];
```

Store `analysis_results` in the same shape as the manual load inputs (e.g. the load table's row columns), keyed by
load combination name, so the rest of the calculation does not care where the loads came from. Make repeated
combination names unique in the integration (keys of one object).

---

## Minimal working example

`s3d_integration.js` for a calculator with inputs `d_input`, `b_input` (mm), `L_input` (mm) and a load table with
columns `N` (kN) and `M_z` (kNm):

```js
module.exports = function (s3d_model, analysis_results) {
	if (!analysis_results) throw new Error("No Analysis Results. Please solve model first.");
	const units = s3d_model.settings.units;
	const ez = StructureHelpers.ezDesignForces(analysis_results, s3d_model, ["axial", "bmd_z"], true);
	const absMax = (values) => values.flat().reduce((g, v) => Math.abs(v) > Math.abs(g) ? v : g, 0);

	let design_members = [];
	let errors = [];
	s3d_model.elements.forEach((member, i) => {
		if (!member) return;
		const section = s3d_model.sections[member[2]];
		const polygon = section && section.aux ? section.aux.polygons[0] : null;
		const material = section ? s3d_model.materials[section.material_id] : null;
		if (!polygon || polygon.shape != "rectangle" || !material || material.class != "steel" || !ez[i]) {
			errors.push(`Member ${i} ignored. `);
			logger(`Member ${i} ignored`);
			return;
		}
		const dim = (k) => polygon.design ? polygon.design[k] : polygon.dimensions[k].value;
		let loads = {};
		analysis_results.forEach((lc, j) => {
			if (!lc || lc.type != "user_defined" || !ez[i][j]) return;
			loads[lc.name] = {
				N: UnitHelpers.convert(absMax(ez[i][j].axial.values), units.force, "kN"),
				M_z: UnitHelpers.convert(absMax(ez[i][j].bmd_z.values), units.moment, "kN-m")
			};
		});
		design_members[i] = {
			d_input: UnitHelpers.convert(dim("h"), units.section_length, "mm"),
			b_input: UnitHelpers.convert(dim("b"), units.section_length, "mm"),
			L_input: UnitHelpers.convert(StructureHelpers.getMemberLength(s3d_model, i), units.length, "mm"),
			analysis_results: JSON.stringify(loads)
		};
	});
	if (design_members.length == 0) throw new Error("No Members Imported \n" + errors.join(""));
	return design_members;
};
```

Inputs left out of the returned objects take their `config.json` defaults, or the values shown in the S3D panel.

---

## Testing

1. **Locally**, before uploading: stub the globals (`StructureHelpers`, `UnitHelpers`, `logger`, `warn`), build a
   small UI-format `s3d_model` with one member per supported shape plus members that should be skipped, and run every
   returned input through `calculate.js`. Run it in metric and imperial model units: the inputs must be identical.
   Also give an input a stale load table alongside `analysis_results` and check that the S3D loads win.
2. **On the platform:** save the pack as a draft, open a solved model in S3D and run
   `S3D.quick_design.import("<draft uid>")` in the browser console. It opens the calculator in the left panel and
   runs the latest `s3d_integration.js` on the server.
3. In S3D, check that every `hide_in_s3d` input is hidden. Inputs that a dropdown's `visible_variables` rules
   `show` are worth checking in particular.

## Checklist

- [ ] `meta.s3d_integrated: true`
- [ ] `analysis_results` input: `custom_object`, `hidden`, `hide_in_s3d`, `exclude_from_input_table`
- [ ] Manual load inputs / load table and their heading are `hide_in_s3d`
- [ ] Every input set per member from the model is `hide_in_s3d`
- [ ] `calculate.js` uses `analysis_results` when present, the manual loads otherwise
- [ ] Every value converted from `s3d_model.settings.units` to the calculator's units
- [ ] Envelopes excluded; combination type (factored / service) matches the Standard, with a `warn()`
- [ ] Sign conventions compared with the calculator's
- [ ] Skipped members logged, and a clear `Error` when none are imported
