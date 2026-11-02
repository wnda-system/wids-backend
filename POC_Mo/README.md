# POC_Mo — design references from the proof of concept (Mo)

No Go code here. During the proof-of-concept phase (Aug–Oct 2026) everything
ran on the sensor in Python: feature extraction, detectors, alert building and
a dashboard. Those pieces move to this backend (ingest half A: alerts, captures)
in WNDA System, so this folder records how they work today, as input for the
schema, `feature.schema.json` and the alert logic.

| File | Use it for |
|---|---|
| [`alerts_and_rules.md`](alerts_and_rules.md) | How alerts are built and merged today: rule types, thresholds, event merging, confidence levels, anti-poisoning, new-AP approval → `alerts` table and the alert logic of ingest half A |
| [`data_formats.md`](data_formats.md) | The per-second features (36, `FEATURE_VERSION` 2), the capture sidecar `.json` and `ground_truth.csv` → `feature_windows`, `captures` and field-test tables |

The code these describe is in the edge repository under `POC_Mo/mo_app/`
(`monitor.py`, `analysis/detector.py`, `analysis/features.py`, `capture.py`,
`detection_rules.md`).
