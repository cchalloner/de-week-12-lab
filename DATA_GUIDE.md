# Week 12 data guide: IT service requests: 1 means specialist escalation occurred
File: data/service_requests.csv. Supplied rows: 480. All data is synthetic, generated for this course. No external dataset or real people/organizations are represented. Labels are simulated from several request attributes plus random variation. Repeated short summaries are intentional: different requests can share a description. Do not deduplicate requests by summary. record_id is unique.

Prediction time: request arrival. Target: escalated (0=no, 1=yes).
Source predictors: requested_units, available_units, asset_age_years (numeric); priority (ordinal low < normal < high); channel (nominal); summary (text).
Exclude record_id (identifier) and actual_work_hours (known only after work is finished). actual_work_hours must never be used to derive a predictor.

Keep raw_df unchanged. Reject rows with any observed negative source numeric value or a target outside {0,1}, including missing targets. Record rejected IDs and reconcile counts. Missing numeric/category/text values are optional and remain valid. No unit conversion, datetime extraction, or invented transformations are required. Numeric columns load as numeric. pandas read_csv default missing parsing is appropriate for these supplied files.

Derive unit_shortage = max(requested_units - available_units, 0) for each row. Preserve a missing derived value if either source is missing. Derive before imputation inside a reusable FunctionTransformer. Numeric branch includes the three original numeric fields plus unit_shortage. Use median imputation then StandardScaler.
Nominal channel: SimpleImputer(strategy='constant', fill_value='missing'), then OneHotEncoder(handle_unknown='ignore') with default sparse output and drop=None.
Ordinal priority: SimpleImputer(strategy='most_frequent'), then OrdinalEncoder(categories=[['low','normal','high']], handle_unknown='use_encoded_value', unknown_value=-1). Missing priority receives the training mode. Unexpected ordinal labels map to -1, a technical fallback that requires review if encountered in practice.
Text summary: missing -> empty string, strip surrounding whitespace, preserve wording, no stop-word removal. Route with a scalar column selector. Use TfidfVectorizer(max_features=100, ngram_range=(1,1), stop_words=None). Default lowercasing and tokenization apply. Do not fit on test/new text. Blank and unknown-only summaries have zero recognized text features.
ColumnTransformer: four branches, remainder='drop', sparse_threshold=1.0. Output may still be dense for unusual density; handle either form when inspecting. A missing required column is a schema error. Unknown channel is an allowed new category and gives an all-zero channel block.
Outer Pipeline: engineer, prepare, classifier. Model: LogisticRegression(max_iter=1000, C=1.0, solver='lbfgs', random_state=42).

Split 80/20, random_state=42, stratify=target, before fitting learned transformations. Use DummyClassifier(strategy='most_frequent') trained on training labels. Compare accuracy, positive-class precision/recall/F1 and a confusion matrix on identical held-out rows. Use zero_division=0 and explain undefined values when relevant. Performance criteria: accuracy strictly above baseline and positive recall >= 0.60. Do not tune from test results. Criteria are practice checks, not deployment authorization.

New-data checks: expected derived values, missing optional inputs, unseen nominal category mobile_app, blank/missing/unknown-only text, stable matrix width, finite values, one prediction per case, unchanged learned vocabulary and median statistics. Trigger and correct a missing required column without refitting.

Limitations: simplified synthetic relationships, small template vocabulary, random rather than future-time evaluation, and no evidence about real service settings. Technical success on edge cases does not prove prediction quality. No personal data is included. In real text workflows, review provenance and sensitive content before sharing with an external AI tool.
