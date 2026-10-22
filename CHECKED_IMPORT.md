# Checked IPT and IAM import — proposed extension

This extension was verified with FreeCAD 1.1.4 and its bundled Python 3.11.

## Changes

- Reconstructs the two analytic helical intersection branches between a circular screw sweep and a coaxial cylinder. The stored approximation is used only to identify a unique branch within its declared tolerance. Reconstructed curves are independently checked on both analytic surfaces to 0.0000001 mm. This fixes a helical test part’s `BOPAlgo_InvalidCurveOnSurface` STEP-reference failure without relaxing the strict geometry checks.
- Material removal is reconstructed as native `Part::Cut` features with linked editable cutting extrusions, explicit `[Cut]` labels, and retained source-operation metadata. Cutting tools and intermediate positive stages remain hidden; only final bodies are displayed. Custom additive/coil/profile features expose their dependencies in the tree.
- Every native IPT/component is compared automatically to the saved BRep exported to STEP and read by `Part.read`. The `InventorStepComparison` property records solid counts and extra/missing material volumes. A mismatch aborts the import. This also applies to definitions loaded through IAM.
- Sketch redundancies introduced when attaching a sketch to its support are resolved after attachment, preventing empty cutting profiles in two flange test parts. Final geometry must still pass the STEP comparison.
- Construction planes retain a persistent view provider so their hidden state survives saving, reopening and recomputation.
- Fix dated 2026-10-22: completes native profiles (including their sketch attachments and intermediate recomputations) before creating their extrusion object. This prevents transient `No object linked` errors while importing text-contour cuts. The live profile dependency and final solid validation are retained. Degenerate source sketch lines and changed IAM component revisions remain reported.
- Reads zlib and Zstandard Inventor streams (the `zstandard` Python package is required for newer files).
- Builds signed ACIS helices from their stored vector basis, avoiding the old quadrant/axis reflection bug.
- Reconstructs circular helical sweep surfaces from their analytic recipe rather than the coarse stored display spline. Curve and surface interpolation is checked against analytic samples to 0.0000001 mm.
- Reads the additional fields present since Inventor 2025 in assembly records `55CE9B23` and `375EAEE5` without shifting their references or transformations. Unknown optional graphics attributes are retained as opaque payloads; they are not used to reconstruct solid geometry.
- Reconstructs supported circular cone/cylinder intersection curves from their surfaces instead of exporting coarse ACIS guides. Ellipses remain analytic; hyperbolas use exact rational splines; other returned intersection splines are checked against both surfaces to 0.000001 mm. Ambiguous branches retain the original path and remain subject to solid validation.
- Preserves the ACIS intcurve reversal in STEP. Closed guide splines now use the correct arc, avoiding self-intersections around toroidal fillets.
- Translates saved BRep to OpenCascade, converts closed shells to solids, enforces a 0.0001 mm tolerance budget, and performs a Boolean geometry check on each solid. Excessive tolerances and open/invalid solids are rejected.
- Registers IAM in FreeCAD's Open/Import dialog. Loads external components as reusable definitions and placed App::Link occurrences, including nested IAMs. Resolves renamed files by stored identity/revision and original filenames. Searches OldVersions for the exact saved revision before accepting a changed current revision. Ambiguous or missing references produce an explicit error.
- Uses private temporary parser directories instead of creating or clearing dump directories beside the original documents.
- Imports transactionally, so a failure does not leave a partial assembly in an existing document.

## Native mode and limitations

Native mode reconstructs stored sketches, parameter spreadsheets, selected sketch regions, extrusion join/cut/intersection operations, constant-radius fillets, supported chamfers, circular-profile coils and supported projected contours. Profiles and Boolean operations retain dependencies and recompute. IAM definitions separate the history folder from the final bodies so recomputation never collects intermediate solids into the occurrence shape. Each component has its own parameter namespace. The native choice propagates through IAM component definitions and nested assemblies.

Fillets expose named radius properties linked to the source parameter spreadsheet. Chamfers expose distance/angle properties and a flip control; equal-distance, two-distance and distance/angle definitions drive the native edge list through expressions. Change dimensions in the component's parameter spreadsheet (or deliberately remove a property's expression before overriding it). This preserves source formulas; changing the numeric edge list alone would be overwritten by its expression. Variable fillets with intermediate radius stations are rejected.

Sketches on explicitly stored work planes use FreeCAD attachments. Named planar support faces are attached when exactly one matching plane exists on the stored creating feature. Attachment offsets preserve the original sketch frame. Independent projection planes share the same support without introducing circular dependencies. Stored text outlines follow their source sketch. `InventorAttachment` records whether a sketch's support was linked or its saved position was retained. Ambiguous/unsupported supports retain their placement; no neighbouring face is guessed. Work planes without a decoded construction relation retain their saved frame and can be moved directly.

Native output must match the saved final solid by solid count and occupied volume in both directions (relative volume tolerance 0.000001). Every final solid also passes a strict OpenCascade Boolean check. Unsupported or mismatching features fail transactionally with an explicit error; native mode never substitutes the final saved body. STEP mode remains available for checked saved geometry.

Imported base features do not contain their original construction history. A complete saved historical shell can be used as that base, but missing historical boundaries are rejected. This does not invent missing sketches. General tapered coils, coil end transitions and arbitrary unsupported Inventor features are not implemented. The complete set of Inventor sketch constraints and sketch-pattern associations is not reconstructed; some retained construction geometry has fixed coordinates. Supported driving dimensions remain linked to the parameter sheet. Topological edge changes after large edits can require repairing downstream references, as in other imported feature histories.

This is a checked reconstruction of supported source features, not universal parity with Inventor's feature/constraint solver. Text outlines are editable curves rather than a live text/font object. Changing sketch orientation does not reconstruct every Inventor direction/axis relation.

IAM support reproduces component hierarchy and rigid placements, with editable definitions in native mode. Inventor mates/constraints, motion, alternative model states, mirrored/scaled occurrences and drawing/presentation documents are not reconstructed. The supported external occurrence records require an unambiguous file stem/name and default saved placement. Unsupported layouts fail explicitly.

An IAM refers to external files; the IAM alone is not sufficient when those files are missing. Shared template IDs alone are not considered a reliable match. Changed revisions are marked with `InventorRevisionChanged` and logged.

## Local validation

The development corpus is private and is not included in this contribution.
See [VALIDATION.md](VALIDATION.md) for anonymized results, reproducible synthetic
tests and the limits of the evidence. No upstream release/version is claimed.
