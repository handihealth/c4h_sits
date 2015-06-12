###C. Retrieve the compositionId of Patient's most recent ‘Problem list' composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Problem List encounter ``composition`` records. Compositions are document-level records which act as the containers for all stored openEHR patient data. i.e. an openEHR EHR is a colleciton of compositions belonging to a single patient.

The `name/value` of the Composition is the root name of the composition archetype `Problem list` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

Creating AQL statements is outside of the scope of this document. You should be able to use or adapt those provided for this scenario.

#####AQL statement

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time
from EHR e[ehr_id/value={{ehrId}}]
contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v0]
where a/name/value=‘Problem list'
order by a/context/start_time/value desc
offset 0 limit 1
````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `uid_value` element in the response is the unique identifier for the composition and `context_start_time` is the time that the document was authored.

Only a single row should be returned, as we are adopting a

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time from EHR e[ehr_id/value= '{{ehrId}}'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v0] where a/name/value=‘Problem list' order by a/context/start_time/value desc offset 0 limit 1
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

###D. Retrieve the Patient's most recent Allergies List composition

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
    “meta”: {
        “href”: “https://rest.ehrscape.com/rest/v1/composition/83063201-c51e-4db8-afbf-22a863738d1e::m_gateway.c4h.com::1”
    },
    “format”: “FLAT”,
    “templateId”: “IDCR Problem List.v0”,
    “composition”: {
        “problem_list/context/setting|238”: true,
        “problem_list/context/setting|code”: “238”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:0/clinical_description”: “only in summer”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|code”: “71923017”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/date_time_clinically_recognised”: “1985-03-22T12:30:37.553Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|value”: “angina pectoris”,
        “problem_list/context/setting|terminology”: “openehr”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/date_time_clinically_recognised”: “1984-07-17T12:30:37.553Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|value”: “eczema”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|301485011”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:0/date_time_clinically_recognised”: “1995-01-22T16:59:41.736Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|value”: “asthma”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|71923017”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|code”: “301485011”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|250503019”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:0/problem_diagnosis_name”: “Hay fever”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|value”: “homeless single person”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/date_time_clinically_recognised”: “2014-07-24T12:30:37.553Z”,
        “problem_list/_uid”: “83063201-c51e-4db8-afbf-22a863738d1e::m_gateway.c4h.com::1”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|code”: “299757012”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|299757012”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/context/start_time”: “2015-02-21T22:11:02.518Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/date_time_clinically_recognised”: “2000-07-17T12:30:37.553Z”,
        “problem_list/context/setting|value”: “other care”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|code”: “250503019”
    },
    “deleted”: false,
    “lastVersion”: true
}````


###E. Persist an updated version of the Problem List Composition as a PUT

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  'target composition' data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example Ehrscape Flat JSON Composition](/technical/instances/LCR Handover/ProblemList_2FLAT.json)  

In this scenario we want to **maintain only a single instance of the Allergies List** and so will use a PUT call to update the previous version. The previous version remains available for audit purposes but would not normally be found by routine querying.

This is in contrast to e.g a Nursing Observation Encounter, where every Encounter merits a separate instance of the document and a POST would normally be used, with PUT updates only being used to correct errors or to perform 'sign-off'.

**When a PUT call is used the compositionId that is passed in must include the domain and version suffix  of the last version**

e.g. `798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2`

#####Call: Updates a new openEhr composition and returns the new CompositionId
````
PUT /rest/v1/composition/{{compositionId}}?format=FLAT&templateId=LCR Problem List.v0

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId

 {
 {
    “meta”: {
        “href”: “https://rest.ehrscape.com/rest/v1/composition/83063201-c51e-4db8-afbf-22a863738d1e::m_gateway.c4h.com::1”
    },
    “format”: “FLAT”,
    “templateId”: “IDCR Problem List.v0”,
    “composition”: {
        “problem_list/context/setting|238”: true,
        “problem_list/context/setting|code”: “238”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:0/clinical_description”: “only in summer”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|code”: “71923017”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/date_time_clinically_recognised”: “1985-03-22T12:30:37.553Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|value”: “angina pectoris”,
        “problem_list/context/setting|terminology”: “openehr”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/date_time_clinically_recognised”: “1984-07-17T12:30:37.553Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|value”: “eczema”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|301485011”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:0/date_time_clinically_recognised”: “1995-01-22T16:59:41.736Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|value”: “asthma”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|71923017”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:2/problem_diagnosis_name|code”: “301485011”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|250503019”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:0/problem_diagnosis_name”: “Hay fever”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|value”: “homeless single person”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/date_time_clinically_recognised”: “2014-07-24T12:30:37.553Z”,
        “problem_list/_uid”: “83063201-c51e-4db8-afbf-22a863738d1e::m_gateway.c4h.com::1”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|code”: “299757012”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/problem_diagnosis_name|299757012”: true,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:3/problem_diagnosis_name|terminology”: “SNOMED-CT”,
        “problem_list/context/start_time”: “2015-02-21T22:11:02.518Z”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:1/date_time_clinically_recognised”: “2000-07-17T12:30:37.553Z”,
        “problem_list/context/setting|value”: “other care”,
        “problem_list/problems_and_issues:0/problem_diagnosis_summary:4/problem_diagnosis_name|code”: “250503019”
    },
    “deleted”: false,
    “lastVersion”: true
}````
#####Return:
````json
{
 "href": "https://rest.ehrscape.com/rest/v1/composition/798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::3"
}
````

###F. Run an AQL call displaying a flat list from the Allergies List composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

The `name/value` of the Composition is the root name of the composition archetype `Allergies list` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

#####AQL statement

````
select
    a_a/data[at0001]/items[at0002]/value as Problem_diagnosis_name,
    a_a/data[at0001]/items[at0003]/value as Date_time_clinically_recognised,
    a_a/data[at0001]/items[at0069]/value as Comment,
    a_a/data[at0001]/items[at0009]/value as Clinical_description
from EHR e
contains COMPOSITION a
contains EVALUATION a_a[openEHR-EHR-EVALUATION.problem_diagnosis.v1]
offset 0 limit 100

````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select     a_a/data[at0001]/items[at0002]/value as Problem_diagnosis_name, a_a/data[at0001]/items[at0003]/value as Date_time_clinically_recognised, a_a/data[at0001]/items[at0069]/value as Comment,  a_a/data[at0001]/items[at0009]/value as Clinical_description from EHR e contains COMPOSITION a contains EVALUATION a_a[openEHR-EHR-EVALUATION.problem_diagnosis.v1] offset 0 limit 100

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

#####Return:
````json
{
   
    “resultSet”: [
        {
            “Comment”: null,
            “Date_time_clinically_recognised”: {
                “@class”: “DV_DATE_TIME”,
                “value”: “2015-03-04T11:55:38.145+01:00”
            },
            “Problem_diagnosis_name”: {
                “@class”: “DV_TEXT”,
                “value”: “Asthma”
            },
            “Clinical_description”: {
                “@class”: “DV_TEXT”,
                “value”: “Exposure to cat dander”
            }
        },
        {
            “Comment”: null,
            “Date_time_clinically_recognised”: null,
            “Problem_diagnosis_name”: {
                “@class”: “DV_TEXT”,
                “value”: “Housing problem”
            },
            “Clinical_description”: {
                “@class”: “DV_TEXT”,
                “value”: “Currently homless”
            }
        },
        {
            “Comment”: null,
            “Date_time_clinically_recognised”: {
                “@class”: “DV_DATE_TIME”,
                “value”: “1995-01-22T17:59:41.736+01:00”
            },
            “Problem_diagnosis_name”: {
                “@class”: “DV_TEXT”,
                “value”: “Hay fever”
            },
            “Clinical_description”: {
                “@class”: “DV_TEXT”,
                “value”: “only in summer”
            }
        },
        {
            “Comment”: null,
            “Date_time_clinically_recognised”: {
                “@class”: “DV_DATE_TIME”,
                “value”: “2000-07-17T14:30:37.553+02:00”
            },
            “Problem_diagnosis_name”: {
                “@class”: “DV_CODED_TEXT”,
                “value”: “angina pectoris”,
                “defining_code”: {
                    “@class”: “CODE_PHRASE”,
                    “terminology_id”: {
                        “@class”: “TERMINOLOGY_ID”,
                        “value”: “SNOMED-CT”
                    },
                    “code_string”: “299757012”
                }
            },
            “Clinical_description”: null
        },
        {
            “Comment”: null,
            “Date_time_clinically_recognised”: {
                “@class”: “DV_DATE_TIME”,
                “value”: “1984-07-17T14:30:37.553+02:00”
            },
            “Problem_diagnosis_name”: {
                “@class”: “DV_CODED_TEXT”,
                “value”: “asthma”,
                “defining_code”: {
                    “@class”: “CODE_PHRASE”,
                    “terminology_id”: {
                        “@class”: “TERMINOLOGY_ID”,
                        “value”: “SNOMED-CT”
                    },
                    “code_string”: “301485011”
                }
            },
            “Clinical_description”: null
        },
        {
            “Comment”: null,
            “Date_time_clinically_recognised”: {
                “@class”: “DV_DATE_TIME”,
                “value”: “1985-03-22T13:30:37.553+01:00”
            },
            “Problem_diagnosis_name”: {
                “@class”: “DV_CODED_TEXT”,
                “value”: “eczema”,
                “defining_code”: {
                    “@class”: “CODE_PHRASE”,
                    “terminology_id”: {
                        “@class”: “TERMINOLOGY_ID”,
                        “value”: “SNOMED-CT”
                    },
                    “code_string”: “71923017”
                }
            },
            “Clinical_description”: null
        },
        {
            “Comment”: null,
            “Date_time_clinically_recognised”: {
                “@class”: “DV_DATE_TIME”,
                “value”: “2014-07-24T14:30:37.553+02:00”
            },
            “Problem_diagnosis_name”: {
                “@class”: “DV_CODED_TEXT”,
                “value”: “homeless single person”,
                “defining_code”: {
                    “@class”: “CODE_PHRASE”,
                    “terminology_id”: {
                        “@class”: “TERMINOLOGY_ID”,
                        “value”: “SNOMED-CT”
                    },
                    “code_string”: “250503019”
                }
            },
            “Clinical_description”: null
        }
    ]
}
````