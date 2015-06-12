###C. Retrieve the compositionId of Patient's most recent 'Relevant contacts' composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Relevant contacts ``composition`` records. Compositions are document-level records which act as the containers for all stored openEHR patient data. i.e. an openEHR EHR is a collection of compositions belonging to a single patient.

The `name/value` of the Composition is the root name of the composition archetype `Relevant contacts` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

Creating AQL statements is outside of the scope of this document. You should be able to use or adapt those provided for this scenario.

#####AQL statement

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time
from EHR e[ehr_id/value={{ehrId}}]
contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v0]
where a/name/value=‘Relevant contacts'
order by a/context/start_time/value desc
offset 0 limit 1
````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `uid_value` element in the response is the unique identifier for the composition and `context_start_time` is the time that the document was authored.

Only a single row should be returned, as we are adopting a

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time from EHR e[ehr_id/value= '{{ehrId}}'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v0] where a/name/value=‘Relevant contacts list' order by a/context/start_time/value desc offset 0 limit 1
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

###D. Retrieve the Patient's most recent Relevant Contacts List composition

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
    "templateId": "LCR Relevant contacts.v0",
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


###E. Persist an updated version of the Relevant Contacts List Composition as a PUT

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  'target composition' data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example Ehrscape Flat JSON Composition](/technical/instances/LCR Handover/RelevantContactsList_2FLAT.json)  

In this scenario we want to **maintain only a single instance of the Relevant contacts** and so will use a PUT call to update the previous version. The previous version remains available for audit purposes but would not normally be found by routine querying.

This is in contrast to e.g a Nursing Observation Encounter, where every Encounter merits a separate instance of the document and a POST would normally be used, with PUT updates only being used to correct errors or to perform 'sign-off'.

**When a PUT call is used the compositionId that is passed in must include the domain and version suffix  of the last version**

e.g. `798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2`

#####Call: Updates a new openEhr composition and returns the new CompositionId
````
PUT /rest/v1/composition/{{compositionId}}?format=FLAT&templateId=LCR Relevant contacts.v0

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

###F. Run an AQL call displaying a flat list from the Rlevant Contacts List composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

The `name/value` of the Composition is the root name of the composition archetype `Relevant contacts list` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

#####AQL statement

````
select
    a/uid/value as uid_value,
    b_a/data[at0001]/items[at0002]/value/value as Causative_agent,
    b_a/data[at0001]/items[at0002]/value/defining_code/code_string as Causative_agent_defining_code_code_string,
    b_a/data[at0001]/items[at0002]/value/defining_code/terminology_id as Causative_agent_defining_code_terminology_id,
    b_a/data[at0001]/items[at0025]/items[at0004]/value/value as Reaction,
    b_a/data[at0001]/items[at0025]/items[at0022]/value/value as Comment
from EHR e[ehr_id/value='{{ehrId}}']
contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v0]
contains EVALUATION b_a[openEHR-EHR-EVALUATION.adverse_reaction_uk.v1]
where a/name/value='Relevant contacts'
offset 0 limit 100

````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select
    a/uid/value as uid_value,
    b_a/data[at0001]/items[at0002]/value/value as Causative_agent,
    b_a/data[at0001]/items[at0002]/value/defining_code/code_string as Causative_agent_defining_code_code_string,
    b_a/data[at0001]/items[at0002]/value/defining_code/terminology_id as Causative_agent_defining_code_terminology_id,
    b_a/data[at0001]/items[at0025]/items[at0004]/value/value as Reaction,
    b_a/data[at0001]/items[at0025]/items[at0022]/value/value as Comment
from EHR e[ehr_id/value='{{ehrId}}']
contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v0]
contains EVALUATION b_a[openEHR-EHR-EVALUATION.adverse_reaction_uk.v1]
where a/name/value='Relevant contacts'
offset 0 limit 100

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

#####Return:
````json
"resultSet":[
    [
        "798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2",
        "allergy to penicillin",
        "91936005",
        {
            "value": "SNOMED-CT"
        },
        null,
        "History unclear"
    ]
]

````