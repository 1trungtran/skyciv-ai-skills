# How to integrate a calc pack with S3D

How to make a Quick Design calc pack run inside S3D, against every member of a solved model. You write a
`s3d_integration.js` file that turns the model and its analysis results into an array of calculator inputs, one per
member. The platform then runs `calculate.js` once per input and shows the results in the S3D Quick Design panel.

> **Prerequisite:** an existing calc pack (`config.json` + `calculate.js`, optional `ui.js` / `utils/`) that already
> runs as a standalone Quick Design calculator (see [`SKILL.md`](../SKILL.md)). This guide only adds the S3D side.
> It reads the same `s3d_model` schema as [`s3d-api`](../../s3d-api/SKILL.md) (but in the **UI format**, see below)
> and the legacy form of the results described in [`analysis-results`](../../analysis-results/SKILL.md). No API key,
> session or REST call is involved.

> **Check S3D Member Design first.** A calc pack is the right tool for codes or checks that
> [`S3D.design.member.check`](../../s3d-api/SKILL.md#member-design-vs-quick-design-which-to-use) doesn't cover (all
> aluminium, concrete, AS 3990, connections...). Don't build a pack to duplicate a steel/CFS/timber code it already supports.

Working example on the platform: `2603-csa-aluminium-design` (CSA S157 aluminium). The official
[`quick-design-s3d.md`](../assets/documentation/quick-design-s3d.md) has the helper output shapes; see
[Where this differs from the official doc](#where-this-differs-from-the-official-doc) before copying its example.

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
| `StructureHelpers.ezDesignForces(analysis_results, s3d_model, action_list, include_lc)` | Actions per member per load combination. Use `include_lc = true` (detailed): per-station `values` about the axes of the shape **as drawn in the section builder, before any rotation or mirror** (the official doc: "with no transformations"). Those are the axes the `polygon.design` dimensions are in, so read the dimensions as drawn and don't adjust for `polygon.operations`: a UB rotated 90° still gets its strong-axis moment as `bmd_z`. `include_lc = false` (express) gives envelope `max` / `min` only and throws if any section is transformed. |
| `StructureHelpers.getDesignForces(analysis_results)` | Older helper: worst + / - of each action over all combinations, per member. |
| `StructureHelpers.getMemberLength(s3d_model, member_id, consider_offsets)` | Member length, in `settings.units.length`. The official doc only says it returns the length "including if offsets are used"; its default isn't documented. Pass `consider_offsets` explicitly, and state in a comment which length the calculator's Standard wants. |
| `UnitHelpers.convert(value, from, to)` | Unit conversion, e.g. `UnitHelpers.convert(L, units.length, "mm")`. |
| `logger(message)` | Writes to the Quick Design logs. Log each skipped member and why. |
| `warn(message)` | Shows a warning to the user (e.g. "loads are unfactored"). |

`ezDesignForces` detailed output: `ez[member_id][lc_index][action] = { values: [...], max, min, abs_max }`, where
`lc_index` is the index into `analysis_results`. Actions: `axial`, `bmd_y`, `bmd_z`, `sfd_y`, `sfd_z`, `torsion`.
`values` runs from end 1 (first) to end 2 (last). Flatten it defensively: the API format stores a discontinuity
station as a `[left, right]` pair, so treat any array entry as two values.

**Turning stations into design values.** There's no single right answer: read the calculator's own manual load
inputs (their labels, `info` text and units) and produce exactly those, per combination. Common cases:

| The calculator asks for | Take from the stations |
|---|---|
| One value per action (`N`, `V_y`, `M_z`...) | The signed value of the largest magnitude. This also covers an axial force that changes sign along the member. |
| Separate tension and compression values (`N_t`, `N_c`) | The largest positive and the largest negative value, each as a magnitude. |
| Values at a position, e.g. end moments for a moment gradient factor | The first and last station for the ends. For a peak between the ends, the largest magnitude of the interior stations. Follow the calculator's rule for when that peak counts (e.g. an AS 3990 pack takes a span moment only where it exceeds both ends, otherwise 0). |
| Something derived from the results, e.g. whether the member is ever in compression | Check it over every station of every combination used. |

If the calculator only takes one set of actions (no load table), `calculate.js` still has to check every combination
in `analysis_results` and report the governing one. Don't pre-envelope the combinations in the integration, because
the governing combination differs between checks.

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

If nothing is left (a model with envelopes only), throw an `Error` that says so.

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
| `hollow rectangle` | `h`, `b`, `t` (top / bottom walls), `tb` (side walls). Treat as square when `h = b` and `t = tb`. **Not confirmed** whether `operations.fillet_radius` is the outside or inside corner radius, or whether the hole polygon carries its own. Check which one the calculator wants (often the internal radius), and state the assumption in a comment. |
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

**How S3D shows the inputs:** an input table with **one editable row per member** (Element ID) and a column for
every input that isn't `hide_in_s3d`. Each row starts from the object the integration returned for that member.

**Return a value for every visible input.** S3D does not fill in `config.json` defaults for inputs left out of the
returned object: a number input shows as a **blank cell**, and a checkbox is **unticked** even when its `default` is
`true` (seen on the platform). Dropdowns did show a value.
Start each member from a copy of the visible inputs' defaults, then set the per-member values over it. A blank
cell can reach `calculate.js` as `""` or `null`, not `undefined`. That can quietly change the result rather than
raise an error: a factor that should default to 1 can be read as 0, or fall to the bottom of a lookup table.

**What to mark `hide_in_s3d`:**

- **Always:** the manual load inputs (single values like `M_fz_input`, or a load entry `table`) and their heading.
  In S3D each member's loads are fixed by the analysis results.
- **Inputs that must match the analysed member:** section / grade dropdowns, dimensions, fillet radius, yield
  strengths, member label, section drawing `div`, and lengths taken from the member. Editing them in a row would
  design a different member from the one analysed. Hide a heading too when every input under it is hidden.
- **Leave visible**, with a starting value per member:
  - design assumptions the model doesn't know, starting at their `config.json` defaults. For example: restraint
    conditions, effective length factors and sidesway for a member check; exposure and cover for concrete; load
    duration or service class for timber; the report type in any pack;
  - assumptions that depend on the section or material. Set each member's value from what the model gives:
    shape, section name or material (`aux.selections` for aluminium temper). A single default may not suit every
    member, and may not be conservative. For example, an AS 3990 steel pack sets fabrication from the section:
    hollow sections are cold-formed, `WB` / `WC` are welded, the rest hot-rolled;
  - inputs whose value follows from the results. For example, the same pack sets its slenderness limit from whether
    the member is ever in compression.

  The engineer can then change any of them for one member in its row.

Hiding lengths stops users entering shorter lengths for intermediate bracing in S3D. Decide this per pack.

**Report the model's section and material names.** Once the section / grade dropdowns are hidden and set to
`Custom`, every report would otherwise say "Section: Custom". Add hidden text inputs for the names and report them
in place of the dropdowns:

| Piece | What to add |
|---|---|
| `config.json` | `s3d_section_name` and `s3d_material_name`: `"type": "text"`, `hidden`, `hide_in_s3d`, `exclude_from_input_table` |
| `s3d_integration.js` | Section: `section.name`, else the last entry of `section.load_section` (confirmed on the platform). Material: `material.name` (not yet confirmed). Fall back to something identifiable, e.g. `Section 3`, not `Custom`. |
| `calculate.js` | Report `input_json.s3d_section_name \|\| section` (and the same for the material), so the standalone calculator is unchanged. |

It's untested whether a dropdown's `visible_variables` rules (e.g. `shape` showing the inputs for one shape) apply
per row in the S3D table. Check this on the platform.

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

This example assumes the calculator has no other visible number inputs. If it has any, return their defaults too:
a number input left out of the returned object shows as a blank cell in the S3D table (see
[What to mark `hide_in_s3d`](#2-configjson)).

---

## Testing

1. **Locally**, before uploading: stub the globals (`StructureHelpers`, `UnitHelpers`, `logger`, `warn`), build a
   small UI-format `s3d_model` with one member per supported shape plus members that should be skipped, and run every
   returned input through `calculate.js` **as returned**. Don't merge in the `config.json` defaults: S3D doesn't, and
   merging hides blank number inputs.
   Run it in metric and imperial model units: the inputs must be identical. Round converted values (e.g. to 6
   significant figures) so floating-point noise doesn't make them differ. Include a rotated section, a model with
   envelopes only, and a check that every visible input is returned. Also give an input a stale load table
   alongside `analysis_results` and check that the S3D loads win. A minimal stub:

   ```js
   const TO_SI = { m: 1, mm: 1e-3, ft: 0.3048, in: 0.0254, kN: 1, kip: 4.4482216, "kN-m": 1, "kip-ft": 1.3558179, MPa: 1, ksi: 6.8947573 };
   global.UnitHelpers = { convert: (v, from, to) => v * TO_SI[from] / TO_SI[to] };
   global.StructureHelpers = {
   	getMemberLength: (model, i) => LENGTHS[i], // in model.settings.units.length
   	ezDesignForces: () => EZ // { [member_id]: { [lc_index]: { axial: { values: [...] }, bmd_z: { values: [...] }, ... } } }
   };
   global.logger = () => {};
   global.warn = () => {};
   ```
2. **On the platform:** save the pack as a draft, open a solved model in S3D and run
   `S3D.quick_design.import("<draft uid>")` in the browser console. It opens the calculator in the left panel and
   runs the latest `s3d_integration.js` on the server.
3. In S3D, check that every `hide_in_s3d` input is hidden. Inputs that a dropdown's `visible_variables` rules
   `show` are worth checking in particular.

## Checklist

- [ ] `meta.s3d_integrated: true`
- [ ] `analysis_results` input: `custom_object`, `hidden`, `hide_in_s3d`, `exclude_from_input_table`
- [ ] Manual load inputs / load table and their heading are `hide_in_s3d`
- [ ] Every input that must match the analysed member (section, dimensions, F_Y, lengths) is `hide_in_s3d`
- [ ] Every visible input is returned for every member (no blank cells in the S3D table): `config.json` defaults, with per-member values for section-dependent assumptions (e.g. fabrication)
- [ ] Hidden `s3d_section_name` / `s3d_material_name` inputs, reported in place of the hidden dropdowns
- [ ] `calculate.js` uses `analysis_results` when present, the manual loads otherwise
- [ ] Every value converted from `s3d_model.settings.units` to the calculator's units
- [ ] Envelopes excluded; combination type (factored / service) matches the Standard, with a `warn()`
- [ ] Sign conventions compared with the calculator's
- [ ] Skipped members logged, and a clear `Error` when none are imported

---

## Where this differs from the official doc

[`quick-design-s3d.md`](../assets/documentation/quick-design-s3d.md) is the published reference, but its aluminium
example predates the pattern above. Don't copy it as-is:

| Official doc | Do this instead | Why |
|---|---|---|
| Uses `StructureHelpers.getDesignForces()` | `ezDesignForces(..., true)` and pick combinations yourself | `getDesignForces` scans **every** result set, envelopes and raw load cases included. Its documented sample has `governing_lc_pos: '#21 Envelope Absolute Max'`. |
| Reads `design_forces.Mz_abs`, `.Vx_pos` | `design_forces.Mz.abs`, `.Vx.pos` if you do use `getDesignForces` | The flat `Mz_abs` keys don't match the nested output shape the same page documents. |
| Reads `polygon.dimensions[k].value` only | `polygon.design[k]` first, then `dimensions[k].value` | Newer models store dimensions in `design`. |
| Writes the loads into visible inputs (`Mz`, `Vy`, `Nc`...) | Hidden `analysis_results` `custom_object` + `hide_in_s3d` on the manual loads | Otherwise the S3D panel shows load inputs the user shouldn't edit, and a single row can't hold every combination. |
