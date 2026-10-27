SENTINEL Pod Report: Dataset Understanding
Oct 25, 2026 ·
1. Dataset overview
 SENTINEL is a synthetic payment-fraud dataset for PayVista. It has 1,000,000 main transactions, 6,000 of them fraud (0.6%), across 18 tables and 171 attributes. Its job is to let us design and test a fraud-scoring and explanation pipeline without touching real customer data.
Business problem. PayVista must decide in real time whether to allow, review or block each payment, then explain that decision to investigators. Fraud is rare, confirmed late (about 60 days) and costly when missed. Blocking good customers is costly too.
What one record represents. One row in transactions is one payment authorisation attempt: a customer paying a merchant, on a channel and device, at a moment in time. It includes failed attempts, so it is an authorisation history, not a settlement ledger.
Scale and time span.
Item	Value
Main transactions	1,000,000 (Jan 2025 to Feb 2026)
Warm-up transactions	100,000 (Oct to Dec 2024), history only
Fraud cases (main)	6,000, rate 0.6%
Customers / merchants / devices	25,000 / 2,000 / 100,120
Legacy case archive	2,000 cases, 2021 to 2024
Text documents / chunks	39,006 / 78,007
Label snapshot date	1 May 2026
Currency and time	INR, integer paise is authoritative; stored in UTC, read in IST
Time-based splits. The benchmark is retrospective, and the fraud rate is stable across splits.
Split	Period (IST)	Rows	Fraud	Fraud rate	Labels mature by next split
Train	Jan to Oct 2025	714,745	4,236	0.59%	571,505
Validation	Nov to Dec 2025	144,803	928	0.64%	2,903
Holdout	Jan to Feb 2026	140,452	836	0.60%	140,452
Types of data. Text IDs and categories, integers, exact money (bigint paise and numeric rupees), UTC timestamps, booleans, JSON objects (locations, evidence, rule expressions, reason codes) and free text (investigator notes, chargeback narratives, playbooks, regulatory exercises).
Table groups.
Group	Tables	Columns	Rows
Core event	transactions	18	1,100,000
Reference entities	customers, merchants, devices	6 / 5 / 3	25,000 / 2,000 / 100,120
Outcome and target	payment_outcomes, labels	5 / 7	1,100,000 each
Model inputs	features	22	1,100,000
Decisions and audit	decision_audit, reviewer_feedback, config_changes	18 / 7 / 9	1,000,000 / 19,376 / 1
Rules	hard_rules, threshold_sets, list_entries	7 / 8 / 6	2 / 1 / 137
Text and LLM training	documents, document_chunks, gold_explanations, preference_pairs, legacy_cases	16 / 8 / 8 / 9 / 9	39,006 / 78,007 / 8,500 / 5,000 / 2,000
The pod dashboard holds a 10% load of this data: 110,000 transactions, 2,500 customers, 200 merchants and 10,120 devices. Column-level profiles in section 2 (medians, ranges, channel shares) were measured on that load; full-scale counts come from manifest.json and quality_report.json.
2. Data dictionary
All 171 attributes are listed below, table by table. Types are the Postgres definitions from data_dictionary.json. Every table also carries is_synthetic (boolean, always true): a provenance flag that is never a model input, so it appears only in the transactions table below. AI role uses four words: input (safe model feature), key (join only), target, audit (metadata, keep out of the model).
2.1 transactions (18 columns, the hub)
Attribute	Type	Business meaning	Values and ranges	AI role
transaction_id	text, PK	Unique payment attempt	e.g. TX_0000096112; no duplicates	key
customer_id	text, FK to customers	Who paid	25,000 customers	key; builds history
merchant_id	text, FK to merchants	Who was paid	2,000 merchants	key; merchant context
device_id	text, FK to devices, nullable	Device used	null in 8,896 main rows (0.9%)	key; new or shared device signals
amount_paise	bigint, > 0	Exact amount in paise (authoritative)	820 to about 800 million; median 125,034	input (one of the two amount columns)
transaction_amount	numeric(14,2)	Same amount in rupees	₹8.20 to ₹79.9 lakh; median ₹1,250; 317 above ₹10 lakh	duplicate of amount_paise; keep one
payment_channel	text	Rail used	UPI 57.9%, card 31.0%, wallet 7.0%, net_banking 4.1%	input
merchant_category	text	Merchant type (MCC group)	grocery, quick_commerce, food, electronics, gaming, gift_cards, healthcare, travel, education, other	input; basis of MCC encoding
transaction_time	timestamptz	When the attempt happened	UTC; Oct 2024 to Feb 2026; night = 00:00 to 05:00 IST	input via hour and night; drives splits
customer_age	integer	Payer age at the time	18 to 80, median 49	input with fairness care
customer_location	jsonb, nullable	Where the payer was	{city, lat, lon}; null in 30,097 rows (3.0%)	input via distance and new city
merchant_location	jsonb	Where the merchant is	{city, lat, lon}; never null	input via distance
account_age	integer	Days since account opened	0 to 2,242, median 740	input; under 30 days = new account
previous_transactions	integer	Prior attempts by this payer	0 to 35, median 3	input; cold-start signal
average_transaction_value	numeric(14,2), nullable	Payer's typical spend in ₹	₹22 to ₹40 lakh; null when no history	input via amount-to-average ratio
failed_transactions	integer	Recent failed attempts	0 to 8	input; card-testing signal
Split	text	Evaluation period	warmup, train, validation, holdout	audit; never a feature
is_synthetic	boolean	Provenance flag	always true	audit
2.2 customers, merchants, devices (reference entities)
Table.attribute	Type	Business meaning	Values and ranges	AI role
customers.customer_id	text, PK	Payer identity	25,000 rows	key
customers.created_at	timestamptz	Account opening time	before each payer's first transaction	source of account_age
customers.customer_age	integer	Age	18 to 80	input with fairness care
customers.home_city	text	Registered city	Indian cities	input via home vs transaction city
customers.region	text	Broad region	south, east, north, west, northeast, central	input; segment analysis
merchants.merchant_id	text, PK	Payee identity	2,000 rows	key
merchants.synthetic_name	text	Fictional business name	free text	audit; never a feature
merchants.merchant_category	text	Business type	same 10 values as transactions	input
merchants.merchant_location	jsonb	Merchant city and coordinates	{city, lat, lon}	input via distance
devices.device_id	text, PK	Device identity	100,120 rows	key
devices.device_kind	text	Device type	mobile (about 75%), browser (about 25%)	input
2.3 payment_outcomes and labels (what happened, and the answer)
Table.attribute	Type	Business meaning	Values and ranges	AI role
payment_outcomes.transaction_id	text, PK, FK	Links outcome to attempt	one per transaction	key
payment_outcomes.outcome_time	timestamptz	When the result was known	shortly after the attempt	audit; outcomes apply only after this time
payment_outcomes.status	text	Authorisation result	SUCCESS or FAILED (about 89% / 11%)	feeds failure-history inputs
payment_outcomes.failure_code	text, nullable	Reason for failure	SIMULATED_AUTH_FAILURE; null on success	audit
labels.transaction_id	text, PK, FK	Links label to attempt	one per transaction	key
labels.fraud_label	smallint, 0 or 1	Confirmed fraud or not	1 = fraud; 0.6% of main set	target
labels.label_source	text	How the label was decided	mature_no_fraud, investigator_verdict, confirmed_chargeback	audit; label trust
labels.label_available_at	timestamptz	When the label became known	about 60 days after the transaction	audit; leakage guard
labels.split	text	Evaluation period	train, validation, holdout, warmup	audit
labels.eligible_at_next_split_start	boolean	Label known before next split began	true / false	filter for strict backtests
2.4 features (22 engineered columns)
All features are computed at authorisation time, before the current event updates any history. Unknown values stay null.
Attribute	Type	Business meaning	AI role
transaction_id	text, PK, FK	Links to the attempt	key
feature_as_of	timestamptz	Snapshot time of the feature row	audit
amount_to_avg_ratio	double, nullable	Amount ÷ payer's mean over the previous 90 days	input
is_new_device	boolean, nullable	Device not seen for this payer in 90 days	input
customer_merchant_distance_km	double, nullable	Haversine distance payer to merchant	input
is_new_city	boolean, nullable	City not seen for this payer in 90 days	input
hour_of_day	integer	IST hour, 0 to 23	input
is_night	boolean	00:00 to 05:00 IST (14.2% of main rows)	input
has_recent_failures	boolean	3 or more failures in 24 hours	input
is_new_account	boolean	Account younger than 30 days	input
txn_count_last_1h	integer	Attempts in the last hour, declines included	input; velocity
txn_amount_last_24h_paise	bigint	Amount attempted in the last 24 hours	input; velocity
distinct_customers_per_device_7d	integer, nullable	Other payers on this device in 7 days	input; fraud-ring signal
combined_risk_flags	integer	Count of device, geography, night, failure and new-account flags	input; rule-style summary
mcc_target_encoded_rate	double	Smoothed fraud rate of the category, from older mature labels only	input
mcc_encoding_as_of	timestamptz	Cut-off of that encoding (frozen 1 Nov 2025 for validation and holdout)	audit
mcc_encoding_count	integer	Labels behind that rate	input (confidence)
missing_customer_location	boolean	Location was null	input
missing_device	boolean	Device was null	input
cold_start	boolean	Payer has no usable history	input
reason_codes	jsonb	Signals that fired	explanation, not a model input
feature_version	text	Pipeline version	audit
2.5 decision_audit, reviewer_feedback, config_changes (decisions and audit)
Every value in decision_audit is simulated: model_version reads SYNTHETIC-HEURISTIC-NOT-A-MODEL. None of these columns may be a model input.
Table.attribute	Type	Business meaning	Values
decision_audit.transaction_id	text, PK, FK	Decision for this attempt	1,000,000 rows (main set)
decision_audit.scored_at	timestamptz	When it was scored	
decision_audit.fraud_probability	numeric(7,6)	Simulated risk score	0 to 1, uncalibrated
decision_audit.decision	text	Action taken	ALLOW 981,690; REVIEW 11,644; BLOCK 6,666
decision_audit.risk_level	text	Score band from threshold_sets	medium, high, critical bands
decision_audit.reason_codes	jsonb	Signals cited	list of codes
decision_audit.explanation	text	Narrative shown to the analyst	free text
decision_audit.decision_source	text	Model, rule or list that decided	
decision_audit.model_version	text	Scorer version	SYNTHETIC-HEURISTIC-NOT-A-MODEL
decision_audit.rules_version	text	Rule set version	
decision_audit.latency_ms	integer	Scoring time	simulated
decision_audit.explanation_latency_ms	integer	Explanation time	simulated
decision_audit.prompt_format	text	LLM prompt template used	
decision_audit.prompt_hash	text	Hash of the prompt	for reproducibility
decision_audit.request	jsonb	Incoming API request	
decision_audit.feature_vector	jsonb	Features as scored	
decision_audit.split	text	Evaluation period	
reviewer_feedback.feedback_id	text, PK	One human review	19,376 rows
reviewer_feedback.transaction_id	text, FK	Reviewed attempt	
reviewer_feedback.reviewed_at	timestamptz	Review time	
reviewer_feedback.verdict	text	Reviewer's call on the decision	
reviewer_feedback.corrected_explanation	text	Reviewer's rewrite	free text
reviewer_feedback.reviewer_id	text	Who reviewed	pseudonymous ID
config_changes.change_id	text, PK	One configuration change	1 row
config_changes.entity, entity_id	text	What was changed (rule, threshold, list)	
config_changes.before_value, after_value	jsonb (before nullable)	Value before and after	
config_changes.changed_by, changed_at, reason	text, timestamptz, text	Who, when, why	
2.6 hard_rules, threshold_sets, list_entries (rule engine)
Table.attribute	Type	Business meaning	Values
hard_rules.rule_id	text, PK	Rule identity	2 rules
hard_rules.code	text	Short rule code	
hard_rules.expression	jsonb	Condition the rule tests	
hard_rules.action	text	What it forces	e.g. block or review
hard_rules.priority	integer	Order of evaluation	
hard_rules.enabled	boolean	Active or not	
threshold_sets.threshold_id	text, PK	Threshold set identity	1 set
threshold_sets.version	integer	Version number	
threshold_sets.mcc_group	text, nullable	Category the set applies to	null = all
threshold_sets.medium_from, high_from, critical_from	numeric(4,3)	Score cut-offs for each risk band	0 to 1
threshold_sets.status	text	Active, draft or retired	
list_entries.entry_id	text, PK	One list entry	137 rows
list_entries.list_type	text	Block list or allow list	
list_entries.entity_id	text	Customer, device or merchant listed	
list_entries.effective_from	timestamptz	Start date	
list_entries.reason	text	Why it was listed	
2.7 Text and LLM-training tables
Table.attribute	Type	Business meaning	Values
legacy_cases.legacy_case_id	text, PK	Closed fraud case, 2021 to 2024	2,000 rows
legacy_cases.customer_id	text, FK	Customer in the case	
legacy_cases.transaction_time, closed_at	timestamptz	When it happened and when it closed	
legacy_cases.amount_paise, payment_channel	bigint, text	Amount and rail	
legacy_cases.fraud_label	smallint	Case outcome	0 or 1
legacy_cases.evidence	jsonb	Evidence gathered	
documents.document_id	text, PK	One document	39,006 rows
documents.kind	text	Document type	investigator note 25,834; chargeback narrative 11,667; regulatory exercise 900; playbook 600; guideline 4; RBA guide 1
documents.title, content	text	Title and full text	
documents.transaction_id, legacy_case_id	text, FK, nullable	What the document is about	
documents.split	text	Evaluation period	
documents.published_at	timestamptz	When it became available	late training notes excluded from CPT
documents.content_sha256	text	Hash of content	used for exact dedup; 0 duplicates
documents.template_family	text	Template it was generated from	30,902 fingerprints
documents.cpt_eligible	boolean	Allowed in continued pre-training	
documents.review_status	text	Human review state	none reviewed
documents.storage_path, record_in_shard	text, integer	Where the raw file sits	
documents.metadata	jsonb	Extra fields	
document_chunks.chunk_id	text, PK	One search chunk	78,007 rows
document_chunks.document_id, chunk_index	text FK, integer	Parent document and position	
document_chunks.content, word_count	text, integer	Chunk text and size	
document_chunks.split, cpt_eligible	text, boolean	Period and CPT eligibility	
gold_explanations.example_id	text, PK	One supervised example	8,500 rows (8,000 + 500 evaluation)
gold_explanations.transaction_id, split	text	Case and period	
gold_explanations.prompt, target	text, jsonb	Input and ideal explanation	
gold_explanations.fraud_label, review_status	smallint, text	Label and review state	not human approved
preference_pairs.pair_id	text, PK	One preference example	5,000 rows
preference_pairs.transaction_id, split, prompt	text	Case, period and prompt	
preference_pairs.chosen, rejected	jsonb	Better and worse answer	
preference_pairs.rejection_reason, review_status	text	Why one lost; review state	
3. Business mapping
The dataset maps one-to-one onto what PayVista needs from SENTINEL: score each payment in real time, act on it, explain it, and learn from what investigators confirm later.
SENTINEL requirement	Data that supports it
Real-time fraud score per payment	transactions + features (22 pre-computed signals)
Allow, review or block decision	threshold_sets, hard_rules, list_entries; decision_audit shows the output shape
Plain-language explanation for analysts	reason_codes, gold_explanations, preference_pairs, documents
Learn from confirmed outcomes	labels (fraud_label, label_source, label_available_at), reviewer_feedback
Audit trail and governance	decision_audit, config_changes, feature_version, prompt_hash
Investigator knowledge search	documents, document_chunks, legacy_cases
Why a real organisation collects this data. Payment firms carry the loss on fraud and chargebacks, and regulators (RBI, NPCI) expect a risk-based approach with explainable decisions. Every column here is either evidence at decision time, the outcome learned later, or a record of what the system did and why.
Insights the data can give.
•	Where fraud concentrates: channel, merchant category (gift cards and gaming are cash-out routes), night hours, new accounts.
•	How a customer's behaviour shifts before fraud: amount far above their average, a new device or city, bursts of attempts.
•	Device sharing across many customers, the mark of organised fraud rings.
•	How much good traffic each rule or threshold blocks, from decision_audit and reviewer_feedback.
Patterns and anomalies to expect.
•	Heavy class imbalance: 6 fraud cases per 1,000 payments.
•	Velocity bursts: 135,199 main rows have 4 or more attempts in the last hour.
•	Failure probing: 89,753 main rows follow 3 or more recent failures.
•	Night activity: 141,672 main rows (14.2%) fall between 00:00 and 05:00 IST.
•	Extreme amounts: 317 payments above ₹10 lakh.
Better decisions it enables. Tune thresholds per merchant category instead of one global cut-off, route borderline cases to review rather than blocking, cut false positives on good customers, and give investigators the reason behind each flag.
Attributes that matter most. amount_to_avg_ratio, txn_count_last_1h and txn_amount_last_24h_paise, is_new_device and distinct_customers_per_device_7d, customer_merchant_distance_km and is_new_city, has_recent_failures, is_new_account, cold_start, payment_channel and mcc_target_encoded_rate. The target is labels.fraud_label.
4. Data quality and relationships
The data is clean at the structural level: no duplicate keys or documents, and every gap is explicit. The real risks are leakage through time, label delay and simulated columns that look like real model output.
4.1 Findings
Area	Finding	Impact on design
Missing values	customer_location null in 30,097 rows (3.0%); device_id null in 8,896 (0.9%); average_transaction_value null when there is no history	Keep nulls; use the missing_* and cold_start flags, never impute a fake location
Duplicates	No duplicate transaction_id; 0 exact duplicate documents; 30,902 template fingerprints; semantic near-duplicates not certified	Safe joins; text corpus may still hold near-identical notes
Redundancy	amount_paise and transaction_amount hold the same value; customer_age and merchant_category are copied into transactions	Use one amount column; check the copies agree with customers and merchants
Outliers	317 payments above ₹10 lakh; amounts span 820 paise to about ₹80 lakh	Log-scale amounts; review the tail before training
Class imbalance	0.6% fraud	Judge with precision, recall and PR-AUC, not accuracy
Label delay	Labels mature 60 days later; only 2,903 of 144,803 validation rows are mature by the holdout start	Filter eligible_at_next_split_start for strict backtests; extend calibration history
Time zones	Stored in UTC, night and splits defined in IST	Convert before any hour or date feature
Nested data	Locations, evidence, rule expressions and reason codes are JSON	Parse and validate before use
Constant columns	is_synthetic always true; failure_code always SIMULATED_AUTH_FAILURE	Drop from modelling
Simulated outputs	decision_audit scores, decisions and latencies are heuristic, not a model	Never use as features or as a performance baseline
Text corpus	4,015 late training documents excluded from CPT; prompt tokens estimated (max 429), not tokenizer-verified	Re-measure with the real tokenizer
4.2 Relationships
transactions is the hub. customers, merchants and devices join to it by foreign key. payment_outcomes, labels, features, decision_audit, gold_explanations, preference_pairs and reviewer_feedback each hold one row per transaction_id. documents link to a transaction or a legacy case, and document_chunks link to documents. Rules, thresholds, lists and config_changes stand alone and are applied by the decision layer.
 
SENTINEL table relationships · from data_dictionary.json foreign keys
4.3 Validation rules to enforce
•	amount_paise > 0, and transaction_amount = amount_paise / 100.
•	fraud_label in {0, 1}; split in {warmup, train, validation, holdout}.
•	customer_age between 18 and 80; account_age ≥ 0; hour_of_day 0 to 23.
•	outcome_time ≥ transaction_time; label_available_at ≥ transaction_time.
•	feature_as_of ≤ transaction_time, so no feature uses the future.
•	Every foreign key resolves; latitude and longitude within India's range.
•	medium_from < high_from < critical_from in each threshold set.
4.4 Privacy and security
•	All data is synthetic; the PII pattern audit of 1,000 of 39,006 documents found no hits, but no human redaction audit was done.
•	In production, customer_id, device_id, locations and free-text notes are personal data under India's DPDP Act: mask or tokenise them, restrict access by role, and log reads.
•	customer_age and region need a fairness check so the model does not penalise groups unfairly.
•	Regulatory exercises are teaching scenarios, not real RBI or NPCI circulars; replace them with licensed official text before real use.
•	The Supabase setup does not reproduce AWS residency, private networking or immutable Object Lock that production needs.
5. AI/ML opportunities
The target is labels.fraud_label, and the 22 features are the starting inputs. The dataset supports three AI parts: a real-time fraud scorer, an LLM that explains decisions, and a search layer over investigator knowledge. Per the brief, this is design thinking only, not implementation.
5.1 Inputs, targets and exclusions
Role	Attributes
Target	labels.fraud_label (binary)
Secondary targets	decision class for routing (allow, review, block); explanation text from gold_explanations
Model inputs	all features columns except keys and audit fields; payment_channel, merchant_category, amount, customer_age, account_age, previous_transactions, failed_transactions, device_kind, region
Keep out	every ID and synthetic_name; split; is_synthetic; label_source, label_available_at, eligible_at_next_split_start; all decision_audit outputs; reviewer verdicts; source documents. The data card forbids these as predictive features.
5.2 Preprocessing
1.	Convert UTC timestamps to IST before deriving hour or date.
2.	Parse JSON locations into city, latitude and longitude.
3.	Keep one amount column and log-transform it.
4.	Keep nulls and pair them with the missing_* and cold_start flags; for models that cannot take nulls, impute and keep the flag.
5.	Encode categories (one-hot for channel; the existing time-safe target encoding for MCC).
6.	Split by time only, never randomly, and respect label maturity.
7.	Handle imbalance with class weights or focal loss rather than duplicating fraud rows.
5.3 Feature engineering opportunities
•	Longer velocity windows (10 minutes, 7 days) and merchant-side velocity.
•	Customer to merchant first-time pairing and recency.
•	Home city versus transaction city, from customers.home_city.
•	Device graph features: customers per device, devices per customer, links to past fraud.
•	Merchant fraud rates by time, built only from mature labels.
•	Sequence features from the 20-event customer histories.
5.4 Relevant approaches
Use case	Approach	Main data
Fraud scoring	Gradient-boosted trees (XGBoost, LightGBM) as the baseline; logistic regression for a transparent benchmark	features, transactions, labels
Unknown fraud patterns	Anomaly detection (isolation forest, autoencoder)	features
Fraud rings	Graph analytics or graph neural networks on customer–device–merchant links	transactions, devices
Behaviour over time	Sequence models on customer histories	20-event histories
Score to action	Calibrated probability + threshold_sets + hard_rules + lists	threshold_sets, hard_rules, list_entries
Analyst explanations	SHAP for the score; LLM fine-tuned with SFT on gold_explanations and DPO on preference_pairs	reason_codes, gold_explanations, preference_pairs
Domain knowledge	Continued pre-training (CPT) on eligible documents; retrieval (RAG) over document_chunks	documents, document_chunks, legacy_cases
Human in the loop	Retrain and recalibrate from reviewer_feedback and matured labels	reviewer_feedback, labels
5.5 Real-time and batch processing
•	Real time (at authorisation): read transaction fields, look up rolling customer and device state, compute features, score, apply rules and thresholds, return allow, review or block, and log to decision_audit. features is built exactly this way: from history before the current event.
•	Batch: nightly labelling as chargebacks and verdicts mature, MCC re-encoding, model retraining and calibration, drift monitoring, threshold tuning, embedding generation for document_chunks, and reporting.
•	The streaming state (1-hour and 24-hour velocity, 7-day device sharing, 90-day history) is what links the two.
6. Discussion notes
The pod agreed that time is the main design constraint in SENTINEL: what was known at the moment of payment decides what can be used.
Key observations
•	Accuracy is meaningless at a 0.6% fraud rate; a model that always says "not fraud" scores 99.4%.
•	The features table is already leakage-safe, because every value is computed before the current event updates state.
•	decision_audit looks like model output but is a heuristic; it shows the output format, not a performance baseline.
•	The validation window is almost entirely unlabelled in real time (2,903 of 144,803 rows mature by the next split).
Decisions
•	Use labels.fraud_label as the only target for the scoring model.
•	Split by time only, following the dataset's train, validation and holdout periods.
•	Exclude IDs, split, label timing fields, decision_audit outputs and is_synthetic from inputs.
•	Keep missing values explicit through the missing_* and cold_start flags.
•	Use amount_paise as the single amount column.
Assumptions
•	Synthetic behaviour is close enough to real PayVista traffic to design the pipeline, but not to claim accuracy or throughput.
•	The 10% load in the pod dashboard has the same distributions as the full dataset; its counts are exactly one-tenth of the manifest's.
Open questions
•	What review capacity does PayVista have per day? This sets the REVIEW threshold.
•	What is the cost of a missed fraud versus a wrongly blocked customer? This decides the operating point.
•	Should thresholds differ by merchant category or channel?
•	What latency budget applies at authorisation time?
•	Which official RBI and NPCI texts will replace the teaching exercises before real domain adaptation?
Learnings
•	Understanding label timing matters as much as understanding columns.
•	Business meaning (cash-out categories, card testing, device rings) explains why a feature works.
•	A dataset card and quality report should be read before any modelling.
7. Pod contributions
The pod split the work into five topics that follow the brief, and each member presented one topic to the whole pod. The table records what each topic covered and the main point it brought out.
Topic	What was explored	Key point brought to the pod
1. Dataset exploration	Purpose, record meaning, scale, table map, time splits	One row = one authorisation attempt; transactions is the hub that every other table keys to
2. Attribute-by-attribute understanding	Every column of transactions, reference tables, outcomes, labels and features	IDs are keys, not predictors; amount_paise is authoritative; features are computed before the decision
3. Business understanding	Why PayVista collects the data, insights, patterns, decisions	Gift cards and gaming are cash-out routes; velocity, new device and failures mark fraud behaviour
4. Data quality and relationships	Nulls, duplicates, outliers, leakage, validation rules, privacy	Label delay and simulated decision_audit outputs are the biggest traps
5. AI system design perspective	Inputs and target, preprocessing, feature ideas, approaches, real time vs batch	Boosted trees for the score, LLM for explanations, RAG for knowledge, time-based splits throughout

