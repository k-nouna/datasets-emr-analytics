# Supporting material for the article

> **An AI Module for Natural Language Querying of Electronic Medical Record Databases and Automated Data Visualization: Case of the MINSANTE EMR System in Cameroon**

This repository holds the two materials the article rests on: the dataset used to train the module, and the field survey that justifies its design.

The article describes an AI module that answers a question asked in French or English against an electronic medical record database, without going through an IT department and without an internet connection, and then picks the visualization that fits the result on its own. The case study is the MINSANTE EMR system in Cameroon, which is built on OpenMRS.

Each file supports one half of that claim.

| File | What it supports in the article |
|---|---|
| `dataset_openmrs_8000.json` | training set for the *natural language querying* half: 8000 question → SQL pairs over the OpenMRS schema, in French and English, with a `graphs` field that also feeds the *automated data visualization* half |
| `emr-analytics-form-answers.csv` | the field survey, anonymized: 28 responses from Cameroonian health workers on how they obtain a figure today, the questions they actually ask, and the form they want the answer in |
| `emr-analytics-eval-questions.csv` | the evaluation set derived from the survey: 159 real questions, one per row, with the chart format the respondent expected — used to score the module on real user input rather than on held-out synthetic data |

The training data is synthetic; the evaluation data is not. That split is the point: the module is trained on generated question → SQL pairs, then scored on questions that real health workers actually wrote, in their own words, with the answer format they said they wanted. The survey also establishes the need and grounds the design — respondents ranked the answer formats they would accept and the one they prefer, which gives the `graphs` values an observed basis.

**Personal data has been removed from the survey export.** Respondents' contact details (7 email addresses and 13 phone numbers, collected only to offer them a trial of the tool) have been deleted, along with the identifiable facility and company names appearing in free text. See [Anonymization](#anonymization) for exactly what was removed.

The sections below document each file: [the training dataset](#the-openmrs-text-to-sql-dataset), [the field survey](#the-field-survey), then [the evaluation set](#the-evaluation-set).

## The OpenMRS text-to-SQL dataset

A text-to-SQL dataset built to train a CodeT5 model to translate natural language questions into SQL queries over the OpenMRS database schema.

The starting point: in an electronic medical record setting, clinical and administrative teams ask very concrete questions ("how many patients were followed in the HIV program this quarter", "which providers handled more than 50 encounters"). Those questions translate into SQL over an OpenMRS schema of more than a hundred tables. Public text-to-SQL datasets (Spider, WikiSQL) do not cover this domain, hence this one.

`dataset_openmrs_8000.json` contains 8000 examples, in French and English.

### What is in the file

The JSON has two keys: `metadata` and `data`.

```json
{
  "metadata": {
    "generated_on": "2025-07-21",
    "total_examples": 8000,
    "examples_per_type": 1000,
    "languages": ["fr", "en"],
    "difficulty_levels": ["facile", "moyen", "complexe"],
    "query_types": ["aggregation", "grouping", "grouping_having", "join",
                    "nested_join", "subquery", "multi_table", "filtering"],
    "schema": "OpenMRS-inspired",
    "version": "2.0"
  },
  "data": [ ... ]
}
```

Each entry in `data` looks like this:

```json
{
  "input": "schema: provider(provider_id(INT), person_id(INT)), encounter_provider(encounter_provider_id(INT), provider_id(INT), encounter_id(INT)) | question: Providers with more than 50 encounters",
  "sql": "SELECT p.provider_id, COUNT(DISTINCT ep.encounter_id) as encounter_count FROM provider p JOIN encounter_provider ep ON p.provider_id = ep.provider_id GROUP BY p.provider_id HAVING COUNT(DISTINCT ep.encounter_id) > 50",
  "type": "grouping_having",
  "level": "moyen",
  "language": "en",
  "graphs": ["bar_chart", "table"],
  "type_of_bd": "medical",
  "domain_schema_id": "openmrs_v1"
}
```

The fields:

* `input`: what the model is given. The linearized schema, then the question, separated by a pipe.
* `sql`: the expected query, i.e. the training target.
* `type`: the query family (8 values, detailed below).
* `level`: `facile`, `moyen` or `difficile` (easy, medium, hard).
* `language`: `fr` or `en`.
* `graphs`: the visualization or visualizations that suit the query result. A multi-label field, usable as an auxiliary task.
* `type_of_bd`: always `medical` here. The field exists so other domains can be mixed in later.
* `domain_schema_id`: always `openmrs_v1`, same reason.
* `uid`: a UUID, present on 7310 of the 8000 examples (see the limitations section).

### The input format

```
schema: table1(col(TYPE), col(TYPE)), table2(col(TYPE), ...) | question: <the question>
```

The schema is pruned: only the tables and columns the question needs are listed, not all hundred-odd OpenMRS tables. That keeps sequences short, which matters a lot with CodeT5 and its 512-token window. In practice the input averages 189 characters (median 175, max 364), and the target SQL 162 characters (median 146, max 472). Truncation is never close.

Column types are MySQL types: INT, VARCHAR, TEXT, CHAR, DATE, DATETIME, DECIMAL, BOOLEAN.

### The OpenMRS schema

OpenMRS is an open source electronic medical record widely deployed across Africa and Asia. Its data model is unusual in two ways, and that is what makes text-to-SQL interesting on it.

First, everything goes through a concept dictionary. A clinical observation (`obs`) does not store "systolic blood pressure = 130", it stores a `concept_id` and a `value_numeric`. Getting a readable label means joining `concept` and then `concept_name`, filtered on locale. Many apparently simple questions therefore turn into three- or four-table joins.

Second, the `person` / `patient` / `users` / `provider` split means one physical person appears across several tables linked by `person_id`. Any query that crosses identity and role has to go through those joins.

The dataset covers roughly 110 real tables of the schema. The most frequent ones, with the number of examples they appear in:

| Table | Role | Occurrences |
|---|---|---|
| `patient` | patients | 1007 |
| `encounter` | encounters | 740 |
| `concept` | concept dictionary | 728 |
| `person` | physical person | 542 |
| `obs` | clinical observations | 528 |
| `orders` | orders | 523 |
| `concept_name` | localized concept labels | 467 |
| `patient_program` | care program enrollments | 433 |
| `visit` | visits | 394 |
| `drug_order` | drug orders | 369 |
| `users` | user accounts | 322 |
| `drug` | drugs | 295 |
| `location` | care sites | 256 |
| `patient_state` | states within a program workflow | 254 |
| `encounter_provider` | encounter / provider link | 246 |

The rest covers the main functional blocks of OpenMRS:

* identity: `person_name`, `person_address`, `person_attribute`, `person_attribute_type`, `patient_identifier`, `patient_identifier_type`, `relationship`, `relationship_type`, `person_merge_log`
* clinical: `encounter_type`, `encounter_role`, `encounter_attribute`, `visit_type`, `visit_attribute`, `note`, `diagnosis`, `encounter_diagnosis`, `conditions`, `allergy`, `allergy_reaction`
* concepts: `concept_class`, `concept_datatype`, `concept_numeric`, `concept_answer`, `concept_set`, `concept_complex`, `concept_proposal`, `concept_reference_map`, `concept_reference_term`, `concept_reference_source`, `concept_map_type`, `concept_state_conversion`
* orders: `order_type`, `order_group`, `order_set`, `order_set_member`, `order_frequency`, `order_attribute`, `drug_ingredient`, `drug_reference_map`, `test_order`, `referral_order`, `care_setting`
* programs: `program`, `program_workflow`, `program_workflow_state`, `patient_program_attribute`, `program_attribute_type`, `cohort`, `cohort_member`
* providers and security: `provider`, `provider_role`, `provider_attribute`, `user_role`, `user_property`, `role`, `privilege`, `role_privilege`
* locations: `location_tag`, `location_tag_map`, `location_attribute`, `location_attribute_type`
* forms: `form`, `form_field`, `form_resource`, `field`, `field_type`, `field_answer`
* appointments: `appointment`, `appointment_type`, `appointment_service_type`
* reporting and infrastructure: `report_definition`, `report_request`, `report_object`, `hl7_source`, `hl7_in_queue`, `hl7_in_archive`, `notification_alert`, `notification_alert_recipient`, `scheduler_task_config`, `global_property`, `clob_datatype_storage`

One detail to watch if you parse the schemas automatically: names like `parent`, `winner`, `loser`, `question`, `answer`, `child`, `creator` or `recipient` appear in some `input` values. They are not OpenMRS tables, they are self-join aliases.

### Distribution

The 8 query types are perfectly balanced, 1000 examples each.

| Type | What it covers |
|---|---|
| `aggregation` | COUNT, AVG, SUM, MIN, MAX on a single table |
| `filtering` | WHERE, simple or compound conditions |
| `grouping` | GROUP BY |
| `grouping_having` | GROUP BY followed by HAVING |
| `join` | two-table join |
| `nested_join` | chained joins over 3 or 4 tables, INNER and LEFT |
| `multi_table` | queries spanning several tables |
| `subquery` | subqueries with IN, EXISTS, or correlated |

On difficulty: 3288 `facile` examples, 3175 `moyen`, 1537 `difficile`. The split is not uniform across types, deliberately: `aggregation` and `filtering` are mostly easy cases, while `nested_join` (666 hard cases out of 1000) and `grouping_having` (473) carry the hard end.

Languages are near parity, 4045 examples in French and 3955 in English.

What the SQL contains, in number of queries affected:

| Construct | Queries |
|---|---|
| SELECT | 8000 |
| WHERE | 5220 |
| COUNT | 2968 |
| JOIN (including 929 LEFT JOIN) | 2923 |
| GROUP BY | 2382 |
| DISTINCT | 1847 |
| HAVING | 1156 |
| IN | 953 |
| AVG | 370 |
| CASE | 223 |
| LIKE | 105 |
| SUM | 89 |
| ORDER BY | 45 |
| EXISTS | 42 |
| LIMIT | 37 |

For the `graphs` field: `table` on 7314 examples, `bar_chart` on 2616, `pie_chart` on 572, `map` on 387, `timeline` on 238, `line_chart` on 218. Most examples carry one or two values.

### Usage

Direct loading:

```python
import json

with open("dataset_openmrs_8000.json", encoding="utf-8") as f:
    payload = json.load(f)

meta = payload["metadata"]
data = payload["data"]

print(len(data))
print(data[0]["input"])
print(data[0]["sql"])
```

With the `datasets` library:

```python
from datasets import Dataset
import json

data = json.load(open("dataset_openmrs_8000.json", encoding="utf-8"))["data"]
ds = Dataset.from_list(data)

splits = ds.train_test_split(test_size=0.1, seed=42, stratify_by_column="type")
```

Preparing for CodeT5:

```python
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM

MODEL = "Salesforce/codet5-base"
tok = AutoTokenizer.from_pretrained(MODEL)
model = AutoModelForSeq2SeqLM.from_pretrained(MODEL)

def preprocess(batch):
    enc = tok(batch["input"], max_length=512, truncation=True)
    lab = tok(text_target=batch["sql"], max_length=256, truncation=True)
    enc["labels"] = lab["input_ids"]
    return enc

tokenized = ds.map(preprocess, batched=True, remove_columns=ds.column_names)
```

The `input` field is ready to tokenize as is, there is no task prefix to add. If you add one ("translate to SQL: " for instance), use the same one at training and at inference.

To work on a subset:

```python
fr_hard = [x for x in data if x["language"] == "fr" and x["level"] == "difficile"]
joins = [x for x in data if x["type"] in ("join", "nested_join", "multi_table")]
```

### Known limitations

A few things to know before using it.

There is no official split. Train / validation / test is up to you, and I would stratify on `type`, `level` and `language` at once, otherwise the hard cases end up badly distributed.

The SQL is MySQL. 737 queries use engine-specific functions (`CURDATE()`, `TIMESTAMPDIFF`, `DATE_SUB`, `DATE_FORMAT`, `YEAR()`, `MONTH()`, `DATEDIFF`). They will not run as is on PostgreSQL or SQLite. If you target another engine, either convert them or drop them.

The queries have not been executed. They were written from the OpenMRS schema, but not validated by running them against a real instance. An `EXPLAIN` pass on a test OpenMRS database is a good idea before any serious use.

690 examples out of 8000 have no `uid` field. If you need a stable key, use `input`: there is not a single duplicate on that field in the whole file.

`metadata` announces a `complexe` level in `difficulty_levels`, but the data uses `difficile`. It is the metadata that is out of date, trust the data.

Finally, everything is synthetic. There is no real patient data in this file, only schema definitions and queries.

## The field survey

The dataset above is synthetic: its questions were written from the schema, not collected from real users. The survey fills that gap. It gives the article three things the dataset cannot: evidence of the problem (how health workers obtain a figure today, and at what cost), the real wording of the questions asked in Cameroonian health facilities, and an observed preference in visualization, which supports the *automated data visualization* half.

`emr-analytics-form-answers.csv` is the export, with personal data removed. The questionnaire was distributed in two versions, French and English, to health workers practising in Cameroon. The recruitment message, in its original French wording:

> Bonjour à tous,
>
> Je travaille sur un outil qui permettrait d'interroger les données du DME de son hôpital en posant simplement la question en français ou en anglais, sans passer par un informaticien et sans connexion internet.
>
> Pour que ça serve à quelque chose, j'ai besoin de savoir quelles questions vous vous posez réellement dans votre service. C'est tout ce que demande ce questionnaire : pas de test, rien de technique. 10 à 15 minutes, anonyme.
>
> 🇫🇷 En français : https://docs.google.com/forms/d/e/1FAIpQLSdibeo9vCWooTsXZvT2mkdCy9UnhSzin1El1lNbxZjeYTTODw/viewform
>
> 🇬🇧 In English: https://docs.google.com/forms/d/e/1FAIpQLSfZU4wpSCoVT6ogx7usAwAVcWBJXMfazESHCXRJJZM6YqgPJA/viewform
>
> Un seul des deux, celui dans lequel vous êtes le plus à l'aise.

In short: an offline, no-IT-department tool for querying your hospital's EMR in plain French or English; the questionnaire asks only which questions you actually ask in your unit; 10 to 15 minutes, anonymous; pick whichever language version you are most comfortable in.

The CSV holds the French form responses only, collected between 11 and 23 September 2026: 28 responses, 36 columns after anonymization. All column headers and answer values are in French, as exported by Google Forms.

### Anonymization

The file distributed here is not the raw Google Forms export. The form was announced as anonymous, but its last question invited respondents to leave contact details so they could be offered a trial of the tool. Those details, and every other directly identifying element, were removed before the file was committed:

| Removed | How |
|---|---|
| The `Si oui, comment vous joindre ?` column (7 email addresses, 13 phone numbers) | column deleted entirely — the export drops from 37 to 36 columns |
| A named private facility, appearing once as a free-text answer and three times inside question text | replaced by `[structure]` |
| Two named companies cited as partners of that facility | replaced by `[entreprise 1]` and `[entreprise 2]` |

Nothing else was altered: all questions, ratings, format choices and free remarks are the respondents' own wording, typos included. No respondent name was ever collected, and the willingness-to-test answer (`Accepteriez-vous d'essayer l'outil quand il sera prêt ?`) is kept since on its own it identifies no one.

What remains is indirect: occupation, unit and facility type together could narrow down a respondent in a small facility. The combination is needed for the analysis, so it is kept, but it is a reason to treat the file as pseudonymized rather than anonymous, and not to republish it joined with any other source.

### File structure

One row per respondent. The columns read in three parts.

First, profile and practice, 9 columns: `Horodateur` (timestamp), consent, occupation, unit or specialty, type of facility, working language, how they obtain a figure today, how long that takes, how often they need one.

Then six question blocks, identical in structure, matching the six prompts of the form (a question put to the data, one put to a patient, one put to the unit, a comparison between two periods or two groups, and so on). Each block is 4 columns, all identically named in the header — which matters if you parse the file:

| Column | Content |
|---|---|
| `Votre question` | the question, free text |
| `Sous quelle forme accepteriez-vous de voir la réponse ?` | which answer formats would you accept — multiple choice, comma-separated values |
| `Et parmi celles-là, votre préférée ?` | which of those you prefer — single choice |
| `Cette question compte beaucoup pour vous ?` | how much this question matters — rating from 1 to 5 |

Because the six blocks share the same headers, a `csv.DictReader` overwrites the first five. You have to use `csv.reader` and slice by position: columns 9 to 32, six blocks of four.

Finally three closing columns: other questions in free form, willingness to try the tool, free remark. The contact column that sat between the last two was deleted during anonymization.

### What the responses say

Occupations, partly free text: 8 physicians, 7 nurses, the rest spread across laboratory, dentistry, pharmacy, sanitary engineering, public health, plus a few students and interns. The labels are not normalized (`Technicien dentaire`, `Technicien dentaires` and `Odontologie` all denote the same job).

Facilities: 8 in a private or faith-based facility, 5 in a general or central hospital, 4 in a district hospital, the rest one each (regional hospital, military hospital, teaching hospital, integrated health centre, pharmacy, regional health delegation). The units named are mostly paediatrics, ENT, dentistry, laboratory, HIV, intensive care, surgery, pharmacy.

On language, 23 work in French and 5 in both equally. No respondent reported working mainly in English — which, together with the absence of responses to the English form, limits for now what the survey can say about the anglophone need.

The clearest finding is how a figure is obtained today. Across 28 responses (the question is multiple choice), 17 mention counting by hand in paper registers, 7 going through the IT department, 4 asking the monitoring and evaluation officer. Only one cites DHIS2, one a business application (SMART), two an existing dashboard. The delay that follows: a few hours for 13 people, a few minutes for 9, one to three days for 5, and one person answering "most of the time, I get nothing". The need itself is monthly for 13 respondents, daily or several times a day for 8, weekly for 3.

148 questions were collected across the six blocks, plus 11 "other questions" fields. 18 respondents out of 28 filled all six slots. Self-reported importance averages around 3.4 out of 5, fairly stable from block to block.

Accepted answer formats, all blocks combined (multiple choice, 148 questions):

| Format | Accepted | Preferred |
|---|---|---|
| Table | 102 | 68 |
| Bar chart | 48 | 23 |
| Just the number | 42 | 19 |
| Pie chart | 40 | 14 |
| Line chart over time | 31 | 19 |
| Doughnut chart | 11 | 0 |
| Map | 6 | 2 |

This is the most directly usable result for the dataset: the table dominates by a wide margin, which matches the distribution of the `graphs` field (7314 examples out of 8000 carry `table`). The pie chart is often accepted but rarely preferred; the doughnut is never chosen first.

Finally, 17 people agree to test the tool, 8 answer "maybe", 3 decline. 20 had left contact details, since removed.

## The evaluation set

`emr-analytics-eval-questions.csv` turns the survey into something the module can be scored against. The training data is synthetic, so evaluating on a held-out slice of it mostly measures how well the model reproduces its own generator. These 159 questions were written by real health workers, in their own words, about their own units — including the messy, elliptical and out-of-scope ones. That is the realistic input distribution.

One row per question. 159 rows: 148 from the six prompt slots, 11 from the free-form "other questions" field. Only the columns needed for scoring are kept; everything else from the survey was dropped.

| Column | Content |
|---|---|
| `question_id` | `R07-Q3`, i.e. third question of respondent 7 |
| `respondent_id` | `R01` to `R28`, so per-respondent effects can be controlled for |
| `role` | normalized occupation: `physician`, `nurse`, `lab_technician`, `dental_technician`, `dentist`, `pharmacist`, `sanitary_engineer`, `student`, `other` |
| `unit` | unit or specialty as written, empty when not given |
| `facility_type` | normalized: `central_hospital`, `district_hospital`, `regional_hospital`, `teaching_hospital`, `military_hospital`, `health_centre`, `private_facility`, `pharmacy`, `health_administration`, `nursing_school`, `other` |
| `language` | `fr` or `fr_en`, the respondent's working language |
| `prompt_slot` | `1` to `6`, or `extra` for the free-form field — which form prompt elicited the question |
| `question_fr` | the question, verbatim, whitespace collapsed |
| `charts_accepted` | every format the respondent would accept, `;`-separated, in the vocabulary of the `graphs` field |
| `chart_expected` | the one they preferred — the gold label for the visualization task |
| `importance` | 1 to 5, self-reported |
| `usable` | `1` / `0`, see below |
| `answerable_sql` | empty, to be annotated by hand |
| `gold_sql` | empty, to be annotated by hand |

Chart labels are mapped onto the `graphs` vocabulary of the training set, so predictions and gold labels share one namespace: `number`, `table`, `bar_chart`, `pie_chart`, `doughnut_chart`, `line_chart`, `map`. 145 rows carry a `chart_expected`; the rest come from the free-form field, which did not ask for a format.

### Scoring with it

Two tasks can be scored, and they need different amounts of work.

The visualization task is ready as is: `chart_expected` is a gold label on 145 rows, so accuracy can be computed without further annotation — and, since `charts_accepted` lists every format the respondent said they would accept, so can a laxer "acceptable format" rate. Report the two side by side: a model answering `bar_chart` where the respondent preferred `table` but would have accepted a bar chart is not failing the way one returning `map` is.

The SQL task cannot be scored automatically yet, which is why the last two columns are empty. `answerable_sql` needs a human pass marking whether each question can be answered at all from an OpenMRS database — a good share cannot — and `gold_sql` needs a reference query for those that can. Until that pass is done, the file supports qualitative error analysis but not an execution-accuracy figure.

The `usable` flag is a first, crude filter, not that annotation. It marks `0` on 35 rows that are not standalone questions: sentence fragments answering the comparison prompts (`Deux périodes`, `Mois après mois`), filler (`RAS`, `Oui`, `Non rien`), and strings under 15 characters. The 124 rows left at `1` are questions in form, but not all of them are answerable in SQL — the heuristic works on shape, not meaning. Filter on `usable == 1` to get a working set, then annotate.



Two things to keep in mind when reporting the score. The questions are French only, so this set says nothing about English performance. And with 28 respondents contributing up to 6 questions each, the rows are not independent — cluster by `respondent_id` before computing any confidence interval.
