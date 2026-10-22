# Checked import: validation and review notes

This is a proposed extension, tested locally on Windows with FreeCAD 1.1.4 and
its bundled Python 3.11.14. It has not been tested on other FreeCAD versions or
platforms. The existing package version and license files are unchanged.

## Behavior to review

- New orchestration: `importerInventor.py`, `importerAssembly.py`.
- Feature reconstruction: `importerNative.py`, `importerReference.py`.
- Saved-solid validation: `importerSolid.py`.
- Helical geometry and intersection repair: `Acis.py`, `Acis2Step.py`,
  `importerHelix.py`, `importerIntersection.py`.
- Parser compatibility: compression, Qt button handling, Inventor 2025+ fields,
  optional graphics payloads, profile B-splines and ACIS persistent names.

Each native part/component is checked against the saved Inventor BRep translated
through STEP and read with `Part.read`. This is independent of native feature
reconstruction, but uses this addon's ACIS translator; it is **not** an independent
Autodesk STEP export. Native and reference solids must pass strict OpenCascade
Boolean checks. Extra and missing occupied material are measured in both
directions; their combined allowance is `max(1e-5 mm³, reference volume * 1e-6)`.
Equal total volumes alone are insufficient. Comparison results are recorded in
`InventorStepComparison`. A mismatch aborts the import transaction.

## Anonymized private-corpus results

These observations are development evidence, not a reproducible public fixture
suite. Original CAD files, generated models, paths and original logs are excluded.
The results combine the full corpus run and targeted follow-up regressions; the
entire corpus was not rerun after the final helical-intersection change.

| Check | Observed result |
| --- | --- |
| 19 current IPT parts | 18 accepted with native/STEP comparison; 1 rejected for unresolved empty fillet edge references |
| Material-removal inventory | 45 source extrusion cuts; 43 verified native cuts in accepted parts |
| Dedicated source Hole features | None in this corpus; holes occur as profile loops/cut extrusions |
| Larger native IAM | 7 component definitions, 10 solids; placements and saved revisions checked |
| Smaller native IAM | 2 component definitions, 2 solids; a changed component revision correctly reported |
| Exact revision resolution | Matching component found in `OldVersions` |
| Save/reopen/recompute | Checked for accepted parts and native assemblies |
| Parameter edits | Cutting depth, sketch dimensions, coil, fillet/chamfer properties and linked IAM occurrences checked |
| Input integrity | Original source hashes unchanged in the checked import runs |

The final helical-part regression passed strict native and STEP checks and
save/reopen/recompute. Its native and STEP difference volumes were both zero;
2,302 sampled boundary points had a maximum bidirectional distance of about
`6.52e-8 mm`. Two other helical parts were rechecked after that change.

## Reproducible synthetic tests

Use FreeCAD's Python with the repository on the import path and the addon's Python
dependencies available (`olefile`, `xlrd`, `xlwt`, `xlutils`, `ezdxf`, `zstandard`).
The current runtime also requires FreeCAD's GUI libraries, even when offscreen.
From the repository root:

```sh
python -m unittest discover -s tests -p test_checked_import.py -v
```

`python` here must resolve to the interpreter able to import `FreeCAD`,
`FreeCADGui`, `Part` and `Sketcher`. On Windows this can be FreeCAD's bundled
`bin/python.exe`; add separately installed addon packages through `PYTHONPATH`
if necessary. Tests do not need Inventor or private CAD files.

The tests cover zlib/Zstandard streams, signed helix endpoints, both screw/cylinder
intersection branches, wrong/filled/extra holes including equal-volume errors,
extra solids, shell conversion, Qt result-code regression and failed-import
preservation of an existing document.

On the contribution checkout, all 8 synthetic tests passed. The geometric test
includes 5 material-comparison cases and both analytic intersection branches.

## Limits

This reconstructs supported editable features; it does not reproduce the entire
Inventor constraint solver. Unsupported/ambiguous features can fail explicitly.
Missing imported-base history is not fabricated. Tapered coils/end transitions,
all sketch constraints/pattern associations, IAM mates/motion/model states and
mirrored/scaled occurrences are not implemented. Some support relations retain
their saved placement. Large edits can invalidate downstream topological edge
references. See [CHECKED_IMPORT.md](CHECKED_IMPORT.md) for further details.
