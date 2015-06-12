###C. Retrieve the compositionId of Patient's most recent 'Transfer of Care Summary' composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use the Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the containers for all stored openEHR patient data. i.e. an openEHR ‘EHR’ is a collection of compositions belonging to a single patient.

The `name/value` of the Composition is the root name of the composition archetype `IDCR Transfer of Care summary (minimal)` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

Creating AQL statements is outside of the scope of this document. You should be able to use or adapt those provided for this scenario.

#####AQL statement

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time
from EHR e[ehr_id/value={{ehrId}}]
contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v0]
where a/name/value='Transfer of Care Summary'
order by a/context/start_time/value desc
offset 0 limit 1
````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `uid_value` element in the response is the unique identifier for the composition and `context_start_time` is the time that the document was authored.

Only a single row should be returned.

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time from EHR e[ehr_id/value= '{{ehrId}}'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary_.v1] where a/name/value='Transfer of Care Summary' order by a/context/start_time/value desc offset 0 limit 1
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

#####Return:
````json
"resultSet": [
        {
            "context_start_time": "2015-02-23T23:11:02.518+01:00",
            "uid_value": "798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2" //compositionId
        }
    ]
````

###D. Retrieve the Patient's most recent Transfer of Care Summary composition

We will use the results of the previous query to retrieve one of the compositions via its ''compositionId''.

Note that this composition has a `::2 suffix`, indicating that this is an updated version of the same document.

The previous version can be read by making the same call with `compositionId = 798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::1`.

Omitting the version suffix will return the current version `compositionId = 798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com`.


#####Call: Returns the specified Composition in FLAT JSON format.
````
GET /rest/v1/composition/{{compositionId}}?format=FLAT
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 Content-Type: application/json
````
#####Return:
````json
{
    "meta": {
        "href": "https://rest.ehrscape.com/rest/v1/composition/798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2",
        "precedingHref": "https://rest.ehrscape.com/rest/v1/composition/798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer::1"
    },
    "format": "FLAT",
    "templateId": "LCR Transfer of Care Summary.v0",
    "composition": {
        "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/causative_agent|terminology": "SNOMED-CT",
        "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/reaction_details/comment": "History unclear",
        "allergies_list/context/setting|238": true,
        "allergies_list/context/setting|code": "238",
        "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/causative_agent|91936005": true,
        "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/causative_agent|code": "91936005",
        "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/causative_agent|value": "allergy to penicillin",
        "allergies_list/context/setting|terminology": "openehr",
        "allergies_list/context/start_time": "2015-02-23T22:11:02.518Z",
        "allergies_list/_uid": "798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2",
        "allergies_list/context/setting|value": "other care"
    },
    "deleted": false,
    "lastVersion": true
}
````


###E. Persist an updated version of the Transfer of Care Summary Composition as a PUT

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  'target composition' data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example Ehrscape Flat JSON Composition](/technical/instances/LCR Handover/AllergiesList_2FLAT.json)  

In this scenario we want to **maintain only a single instance of the Transfer of Care Summary** and so will use a PUT call to update the previous version. The previous version remains avaialble for audit purpsoes but would not normally be found by routine querying.

This is in contrast to e.g a Nursing Observation Encouter, where every Encounter merits a separate instance of the document and a POST would norrmally be used, with PUT updates only being used to correct errors or to perform 'sign-off'.

**When a PUT call is used the compositionId that is passed in must include the domain and version suffix  of the last version**

e.g. `798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2`

#####Call: Updates a new openEhr composition and returns the new CompositionId
````
PUT /rest/v1/composition/{{compositionId}}?format=FLAT&templateId=LCR Transfer of Care Summary.v0

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId

 {
  "ctx/composer_name": "Dr Joyce Smith",
  "ctx/health_care_facility|id": "999999-345",
  "ctx/health_care_facility|name": "Northumbria Community NHS",
  "ctx/id_namespace": "NHS-UK",
  "ctx/id_scheme": "2.16.840.1.113883.2.1.4.3",
  "ctx/language": "en",
  "ctx/territory": "GB",
	  "ctx/time": "2015-02-24T00:11:02.518+02:00",
    "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/causative_agent|value": "allergy to penicillin",
    "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/causative_agent|code": "91936005",
    "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/causative_agent|terminology": "SNOMED-CT",
    "allergies_list/allergies_and_adverse_reactions:0/adverse_reaction:0/reaction_details/comment": "History unclear"
}
````
#####Return:
````json
{
 "href": "https://rest.ehrscape.com/rest/v1/composition/798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::3"
}
````

###F. Run an AQL call displaying a flat list from the Transfer of Care Summary composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

The `name/value` of the Composition is the root name of the composition archetype `Transfer of Care Summary` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

#####AQL statement

````
select
    a_a/data[at0001]/items[at0002]/value as Causative_agent,
    a_b/items[at0001]/value as Medication_name,
    a_b/items[at0020]/value as Dose_amount_description,
    a_b/items[at0021]/value as Dose_timing_description,
    a_c/data[at0001]/items[at0002]/value as Problem_diagnosis_name,
    a_c/data[at0001]/items[at0003]/value as Date_time_clinically_recognised,
    a_c/data[at0001]/items[at0002]/value as Problem_diagnosis_summary_Problem_diagnosis_name,
    a_d/data[at0001]/items[at0002, ‘Summary’]/value as Summary
from EHR e
contains COMPOSITION a
contains (
    EVALUATION a_a[openEHR-EHR-EVALUATION.adverse_reaction_uk.v1] and
    CLUSTER a_b[openEHR-EHR-CLUSTER.medication_item.v1] and
    EVALUATION a_c[openEHR-EHR-EVALUATION.problem_diagnosis.v1] and
    EVALUATION a_d[openEHR-EHR-EVALUATION.clinical_synopsis.v1])
offset 0 limit 100
````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select     a_a/data[at0001]/items[at0002]/value as Causative_agent,     a_b/items[at0001]/value as Medication_name,     a_b/items[at0020]/value as Dose_amount_description,     a_b/items[at0021]/value as Dose_timing_description,     a_c/data[at0001]/items[at0002]/value as Problem_diagnosis_name,     a_c/data[at0001]/items[at0003]/value as Date_time_clinically_recognised,     a_c/data[at0001]/items[at0002]/value as Problem_diagnosis_summary_Problem_diagnosis_name,     a_d/data[at0001]/items[at0002, ‘Summary’]/value as Summary from EHR e contains COMPOSITION a contains (     EVALUATION a_a[openEHR-EHR-EVALUATION.adverse_reaction_uk.v1] and     CLUSTER a_b[openEHR-EHR-CLUSTER.medication_item.v1] and     EVALUATION a_c[openEHR-EHR-EVALUATION.problem_diagnosis.v1] and     EVALUATION a_d[openEHR-EHR-EVALUATION.clinical_synopsis.v1]) offset 0 limit 100

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

#####Return:
````json
“resultSet”: [
{
“Dose_timing_description”: {
“@class”: “DV_TEXT”,
“value”: “as required”
},
“Causative_agent”: {
“@class”: “DV_CODED_TEXT”,
“value”: “allergy to penicillin”,
“defining_code”: {
“@class”: “CODE_PHRASE”,
“terminology_id”: {
“@class”: “TERMINOLOGY_ID”,
“value”: “SNOMED-CT”
},
“code_string”: “9193600”
}
},
“Dose_amount_description”: {
“@class”: “DV_TEXT”,
“value”: “2 puffs”
},
“Date_time_clinically_recognised”: {
“@class”: “DV_DATE_TIME”,
“value”: “2015-03-04T11:55:38.145+01:00”
},
“Summary”: {
“@class”: “DV_TEXT”,
“value”: “Exascerbation of asthma after exposure to friend’s cat. Stormy course at first but quickly settled. Social services have been contacted re housing problems.”
},
“Problem_diagnosis_summary_Problem_diagnosis_name”: {
“@class”: “DV_TEXT”,
“value”: “Asthma”
},
“Problem_diagnosis_name”: {
“@class”: “DV_TEXT”,
“value”: “Asthma”
},
“Medication_name”: {
“@class”: “DV_CODED_TEXT”,
“value”: “Salbutamol 100micrograms/dose breath actuated inhaler CFC free”,
“defining_code”: {
“@class”: “CODE_PHRASE”,
“terminology_id”: {
“@class”: “TERMINOLOGY_ID”,
“value”: “SNOMED-CT”
},
“code_string”: “320151000”
}
}
},
{
“Dose_timing_description”: {
“@class”: “DV_TEXT”,
“value”: “as required”
},
“Causative_agent”: {
“@class”: “DV_CODED_TEXT”,
“value”: “allergy to penicillin”,
“defining_code”: {
“@class”: “CODE_PHRASE”,
“terminology_id”: {
“@class”: “TERMINOLOGY_ID”,
“value”: “SNOMED-CT”
},
“code_string”: “9193600”
}
},
“Dose_amount_description”: {
“@class”: “DV_TEXT”,
“value”: “2 puffs”
},
“Date_time_clinically_recognised”: null,
“Summary”: {
“@class”: “DV_TEXT”,
“value”: “Exascerbation of asthma after exposure to friend’s cat. Stormy course at first but quickly settled. Social services have been contacted re housing problems.”
},
“Problem_diagnosis_summary_Problem_diagnosis_name”: {
“@class”: “DV_TEXT”,
“value”: “Housing problem”
},
“Problem_diagnosis_name”: {
“@class”: “DV_TEXT”,
“value”: “Housing problem”
},
“Medication_name”: {
“@class”: “DV_CODED_TEXT”,
“value”: “Salbutamol 100micrograms/dose breath actuated inhaler CFC free”,
“defining_code”: {
“@class”: “CODE_PHRASE”,
“terminology_id”: {
“@class”: “TERMINOLOGY_ID”,
“value”: “SNOMED-CT”
},
“code_string”: “320151000”
}
}
}
]
````