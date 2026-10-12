# Resolving sections inside S3D

Read this before writing code that runs inside S3D (an S3D App, a Left Menu panel, or any
in-app script) and either creates a section or reads a new section's properties. Sections
include library sections (`load_section`) and template shapes (`info`, e.g. a concrete
rectangle). The API format describes a section by **what it is**: a library path or a shape
with dimensions. S3D then has to **build** the section object, which holds area, Iz, Iy, J,
shear areas and the polygon geometry. Building it is a round trip to the SkyCiv server, so
the resolved section only exists **in a callback**.

## The resolved section is never available synchronously

```javascript
// WRONG — the section has not been built yet
const out = S3D.API.API2S3D({ sections: { "1": { material_id: 1, info: { shape: "rectangle", dimensions: { b: 350, h: 500 } } } } });
console.log(out.sections[1].area);   // undefined / not resolved
```

Any code that reads `area`, `Iz` or `Iy` for display, or pushes the section into the live
model, must wait for the callback. The same is true of `load_section` paths.

## Two ways to resolve, by job

| Job | Call | Where the resolved section is |
|---|---|---|
| Change the model and you don't need the section's properties yourself | `S3D.structure.set(temp_s3d_model, null, true, callback)` | Already resolved inside the model once `callback` runs. Read `S3D.structure.get().sections` there, not before. |
| You need the section object itself: show its properties, check them, or add it to the live model without a full `set` | `S3D.API.API2S3D(apiModel, null, callback)` | `callback(s3dModel)` — `s3dModel.sections` and `s3dModel.materials` are S3D-format, null-padded arrays (index `1` is the `"1"` you passed). |

`apiModel` takes the same API-format pieces as the `s3d-api` skill. You only need the parts
you are converting, e.g. `{ materials, sections }`.

## Minimal working example: a concrete section from dimensions

Metric model (`section_length` mm, `material_strength` MPa, `density` kg/m³). The material
and section are converted together, because the section's `material_id` points into the
`materials` you pass.

```javascript
S3D.API.API2S3D({
  materials: { "1": { name: "Concrete C35/45", class: "concrete", density: 2500,
                      elasticity_modulus: 34000, poissons_ratio: 0.2,
                      yield_strength: 35, ultimate_strength: 35 } },
  sections:  { "1": { material_id: 1, info: { shape: "rectangle", dimensions: { b: 350, h: 500 } } } },
}, null, function (conv) {
  const mat = conv && conv.materials && conv.materials[1];
  const sect = conv && conv.sections && conv.sections[1];
  // Check: a rectangle's area must be b × h in section_length² (a circle's π D² / 4). If it
  // isn't, a key was wrong or the dimensions were read in other units, so add nothing.
  if (!mat || !sect || Math.abs(sect.area - 350 * 500) > 0.01 * 350 * 500) {
    SKYCIV_UTILS.alert.sideNotify({ title: "Section", content: "The section did not resolve.", theme: "error" });
    return;
  }
  const model = S3D.structure.get();
  if (model.materials.length === 0) model.materials[0] = null;   // ids start at 1
  model.materials.push(mat);
  sect.material_id = model.materials.length - 1;
  if (model.sections.length === 0) model.sections[0] = null;
  model.sections.push(sect);
  const sectionId = model.sections.length - 1;
  // ...assign: model.elements[memberId][2] = sectionId
  S3D.UI.update({ redraw: true, update_undo: true });
});
```

The full list of shapes and dimension keys is in the `section-selector` skill under
"Supported shapes and their dimension keys". The keys are case-sensitive: `circle` and
`hollow circle` take a capital `D`, and the I and T shapes take `TFw`, `TFt`, `BFw`, `BFt`,
`h`, `Wt`, `r`. A wrong key does not resolve. Dimensions are in the model's
`section_length` units.

## Ordering

Build geometry (nodes, members, loads) **synchronously first**, and do only the section work
in the callback. A conversion that fails, or a library path that misses, then costs the
section, not the whole build. Never nest node or member creation inside the callback.

## Units

Material values (`elasticity_modulus`, `yield_strength`, `density`) are in the model's
`material_strength` and `density` units. Check `S3D.structure.get().settings.units`. In an
imperial model, convert with `SKYCIV_UTILS.units.convert(value, from, to)`; don't hand-roll
the factors.
