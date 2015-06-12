##Code4Health Ehrscape ‘Visual Acuity’ Technical Guide

This document describes the series of [Ehrscape API](https://dev.ehrscape.com/api-explorer.html) calls required for the 'Handover Summary' scenario and assumes a base level of understanding of that API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

The steps covered are...  

* Open the Ehrscape Session

* Retrieve Patient's ehrId from Ehrscape, based on their subjectId 
* If the patent does not have an EHR, create a new EHR in Ehrscape, based on their subjectId
* Retrieve the compositionID of Patient's most recent Visual Acuity Report composition
* Retrieve the Patient's most recent Visual Acuity Report composition
* Persist an updated AllergyList Composition as a POST i.e update a new instance of the same composition.
* Run an AQL query which returns a 'flat list' of key datapoints from the Visual Acuity Report for read-only display.

* Close the Ehrscape Session

* Other API services
 * Access the Indizen SNOMED CT Terminology browser service  

#### Ehrscape API base URL

The baseURL for all Ehrscape API calls is https://rest.ehrscape.com

#### Substitutable parameters

In various places you will see the use of double curly brackets e.g. {{ehrId}}. This signifies that ehrId is a placeholder and needs to be substituted by the actual value of ehrId.
 
###Key API parameters for your C4H Ehrscape domain

Domain login:
"Username": {{userLogin}} 
"Password": {{userPassword}}  

This is the Code4Health Ehrscape domain username and password which you will have been issued.

Dummy patient: (Steve Walford M dob 12-Jul-1965):

```json
{
    "address": {
        "address": "60 Florida Gardens, Cardiff, LS23 4RT",
        "id": "7256",
        "version": 1
    },
    "dateOfBirth": "1965-07-12T00:00:00.000Z",
    "firstNames": "Steve",
    "gender": "MALE",
    "id": "7256",
    "lastNames": "Walford",
    "partyAdditionalInfo": [
        {
            "id": "7257",
            "key": "title",
            "value": "Mr",
            "version": 0
        },
        {
            "id": "7258",
            "key": "uk.nhs.nhsnumber",
            "value": "7430555",
            "version": 1
        }
    ],
    "version": 1
}
```
###A. Open the Code4Health Ehrscape session

The first step in working with Ehrscape is to open a Session and retrieve the ``sessionId`` token. This allows subsequent API calls to be made without needing to login on each occasion.  
The session should be formally closed when you are finished.

The baseURL for all Ehrscape calls is https://rest.ehrscape.com

i.e. the call below should be to https://rest.ehrscape.com/rest/v1/session?username={{userLogin}}&password={{userPassword}}

#####Call: Create a new openEHR session:
 ````
 POST /rest/v1/session?username={{userLogin}}&password={{userPassword}} //substitute your Code4Health Ehrscape user name and password
 ````
#####Returns:
````json
{
"sessionId": "fc234d24-7b59-49f5-a724-8d37072e832b"
}

````

###B1. Retrieve Patient's ehrId from Ehrscape based on their subjectId and subjectNamespace

If the patient already has an EHR and you have a patient's subjectId, you may retrieve the patient's internal `ehrID` asociated with that subjectId. 

#####Call: Returns the EHR for the specified subject ID and namespace.
````
GET /rest/v1/ehr/?subjectId=63436&subjectNamespace=ehrscape
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
#####Return:
````json
{
  "ehrId": "0da489ee-c0ae-4653-9074-57b7f63c9f16" // Note that the ehrId will be different in your domain.
}
````


###C. Retrieve the compositionId of Patient's most recent ‘Visual Acuity Report’ composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the containers for all stored openEHR patient data. i.e. an openEHR EHR is a colleciton of compositions belonging to a single patient.

The `name/value` of the Composition is the root name of the composition archetype `Allergies list` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

Creating AQL statements is outside of the scope of this document. You should be able to use or adapt those provided for this scenario.

#####AQL statement

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time
from EHR e[ehr_id/value=‘f3da219d-4ca6-461f-a9ea-a1ad5df069e9’]
contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] 
where a/name/value= ‘Visual Acuity Report’ 
order by a/context/start_time/value desc
offset 0 limit 1
````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `uid_value` element in the response is the unique identifier for the composition and `context_start_time` is the time that the document was authored.

Only a single row should be returned, as we are adopting a

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select     a/uid/value as uid_value,     a/context/start_time/value as context_start_time from EHR e[ehr_id/value=‘f3da219d-4ca6-461f-a9ea-a1ad5df069e9’] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1]  where a/name/value= ‘Visual Acuity Report’  order by a/context/start_time/value desc offset 0 limit 1 HTTP/1.1
Host: rest.ehrscape.com
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

#####Return:
````json
{
  “meta”: {
    “href”: “https://rest.ehrscape.com/rest/v1/query/?aql=select%20%20%20%20%20a/uid/value%20as%20uid_value,%20%20%20%20%20a/context/start_time/value%20as%20context_start_time%20from%20EHR%20e%5Behr_id/value%3D'f3da219d-4ca6-461f-a9ea-a1ad5df069e9’%5D%20contains%20COMPOSITION%20a%5BopenEHR-EHR-COMPOSITION.report.v1%5D%20%20where%20a/name/value%3D%20’Visual%20Acuity%20Report’%20%20order%20by%20a/context/start_time/value%20desc%20offset%200%20limit%201”
  },
  “aql”: “select     a/uid/value as uid_value,     a/context/start_time/value as context_start_time from EHR e[ehr_id/value=‘f3da219d-4ca6-461f-a9ea-a1ad5df069e9’] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1]  where a/name/value= ‘Visual Acuity Report’  order by a/context/start_time/value desc offset 0 limit 1”,
  “executedAql”: “select     a/uid/value as uid_value,     a/context/start_time/value as context_start_time from EHR e[ehr_id/value=‘f3da219d-4ca6-461f-a9ea-a1ad5df069e9’] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1]  where a/name/value= ‘Visual Acuity Report’  order by a/context/start_time/value desc offset 0 limit 1”,
  “resultSet”: [
    {
      “context_start_time”: “2015-02-21T23:11:02.518+01:00”,
      “uid_value”: “49f085f8-fa4f-4d5c-a2f2-f62a1f0af4ab::across.c4h.ehrscape.com::1”
    }
  ]
}
````

###D. Retrieve the Patient's most recent Visual Acuity Report composition

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
        “href”: “https://rest.ehrscape.com/rest/v1/composition/49f085f8-fa4f-4d5c-a2f2-f62a1f0af4ab::across.c4h.ehrscape.com::1”
    },
    “format”: “FLAT”,
    “templateId”: “Across - Visual Acuity Report”,
    “composition”: {
        “visual_acuity_report/_uid”: “49f085f8-fa4f-4d5c-a2f2-f62a1f0af4ab::across.c4h.ehrscape.com::1”,
        “visual_acuity_report/context/start_time”: “2015-02-21T22:11:02.518Z”,
        “visual_acuity_report/context/setting|238”: true,
        “visual_acuity_report/context/setting|code”: “238”,
        “visual_acuity_report/context/setting|value”: “other care”,
        “visual_acuity_report/context/setting|terminology”: “openehr”,
        “visual_acuity_report/visual_acuity:0/any_event:0/test_name|at0139”: true,
        “visual_acuity_report/visual_acuity:0/any_event:0/test_name|code”: “at0139”,
        “visual_acuity_report/visual_acuity:0/any_event:0/test_name|value”: “Unaided visual acuity”,
        “visual_acuity_report/visual_acuity:0/any_event:0/test_name|terminology”: “external”,
        “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/eye_examined|at0012”: true,
        “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/eye_examined|code”: “at0012”,
        “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/eye_examined|value”: “Left eye”,
        “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/eye_examined|terminology”: “local”,
        “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/notation/low_vision_score|code”: “at0017”,
        “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/notation/low_vision_score|value”: “PL -  Perception of light”,
        “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/notation/low_vision_score|ordinal”: 2,
        “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/eye_examined|at0013”: true,
        “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/eye_examined|code”: “at0013”,
        “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/eye_examined|value”: “Right eye”,
        “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/eye_examined|terminology”: “local”,
        “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/notation/low_vision_score|code”: “at0018”,
        “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/notation/low_vision_score|value”: “HM - Hand movement”,
        “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/notation/low_vision_score|ordinal”: 3,
        “visual_acuity_report/visual_acuity:0/any_event:0/time”: “2015-02-21T22:11:02.518Z”,
        “visual_acuity_report/visual_acuity:0/any_event:1/test_name|at0137”: true,
        “visual_acuity_report/visual_acuity:0/any_event:1/test_name|code”: “at0137”,
        “visual_acuity_report/visual_acuity:0/any_event:1/test_name|value”: “Best corrected visual acuity”,
        “visual_acuity_report/visual_acuity:0/any_event:1/test_name|terminology”: “external”,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/eye_examined|at0012”: true,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/eye_examined|code”: “at0012”,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/eye_examined|value”: “Left eye”,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/eye_examined|terminology”: “local”,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/notation/low_vision_score|code”: “at0019”,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/notation/low_vision_score|value”: “CF - Count fingers”,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/notation/low_vision_score|ordinal”: 4,
        “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/interpretation:0”: “Poor vision.”,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/eye_examined|at0013”: true,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/eye_examined|code”: “at0013”,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/eye_examined|value”: “Right eye”,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/eye_examined|terminology”: “local”,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/notation/low_vision_score|code”: “at0018”,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/notation/low_vision_score|value”: “HM - Hand movement”,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/notation/low_vision_score|ordinal”: 3,
        “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/interpretation:0”: “Corrected vision in right eye pretty still pretty poor”,
        “visual_acuity_report/visual_acuity:0/any_event:1/time”: “2015-02-21T22:11:02.518Z”
    },
    “deleted”: false,
    “lastVersion”: true
}
````


###E. Persist a new instance of the Visual Acuity Report Composition as a POST

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  'target composition' data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example Ehrscape Flat JSON Composition](/technical/instances/visual_acuity/Visual acuity Report FLAT.json)  

In this scenario every Visual Acuity Report merits a separate instance of the document and a POST is used, with PUT updates only being used to correct errors or to perform 'sign-off'.

**When a PUT call is used the compositionId that is passed in must include the domain and version suffix  of the last version**

e.g. `798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2`

#####Call: POST a new openEhr composition and returns the new CompositionId
````
POST /rest/v1/composition?ehrId=f3da219d-4ca6-461f-a9ea-a1ad5df069e9&amp;templateId=Across - Visual Acuity Report&amp;committerName=handi&amp;format=FLAT
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId

{
  “ctx/composer_name”: “Dr Joyce Smith”,
  “ctx/health_care_facility|id”: “999999-345”,
  “ctx/health_care_facility|name”: “Northumbria Community NHS”,
  “ctx/id_namespace”: “NHS-UK”,
  “ctx/id_scheme”: “2.16.840.1.113883.2.1.4.3”,
  “ctx/language”: “en”,
  “ctx/territory”: “GB”,
	“ctx/time”: “2015-02-22T00:11:02.518+02:00”,
    “visual_acuity_report/visual_acuity:0/any_event:0/test_name|code”: “at0139”,
    “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/eye_examined|code”: “at0012”,
    “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/notation/low_vision_score|code”: “at0017”,
    “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/notation/low_vision_score|ordinal”: 2,
    “visual_acuity_report/visual_acuity:0/any_event:0/left_eye/notation/low_vision_score|value”: “PL - Perception of light”,
    “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/eye_examined|code”: “at0013”,
    “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/notation/low_vision_score|code”: “at0018”,
    “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/notation/low_vision_score|ordinal”: 3,
    “visual_acuity_report/visual_acuity:0/any_event:0/right_eye/notation/low_vision_score|value”: “HM - Hand movement”,  
    “visual_acuity_report/visual_acuity:0/any_event:1/test_name|code”: “at0137”,
    “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/eye_examined|code”: “at0012”,
    “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/notation/low_vision_score|code”: “at0019”,
    “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/notation/low_vision_score|ordinal”: 4,
    “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/notation/low_vision_score|value”: “CF - Count fingers”,
    “visual_acuity_report/visual_acuity:0/any_event:1/left_eye/interpretation:0”: “Poor vision.”,
    “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/eye_examined|code”: “at0013”,
    “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/notation/low_vision_score|code”: “at0018”,
    “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/notation/low_vision_score|ordinal”: 3,
		“visual_acuity_report/visual_acuity:0/any_event:1/right_eye/notation/low_vision_score|value”: “HM - Hand movement”,		
    “visual_acuity_report/visual_acuity:0/any_event:1/right_eye/interpretation:0”: “Corrected vision in right eye pretty still pretty poor”
	} 
````
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
    b_a/data[at0001]/events[at0134]/data[at0003]/items[at0138]/value as Test_Name,
    b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Left eye’]/items[at0028]/items[at0015]/value as Left_Low_Vision_Score,
    b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Right eye’]/items[at0028]/items[at0015]/value as Right_Low_Vision_Score,
    b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Right eye’]/items[at0066]/value as Right_Interpretation,
    b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Left eye’]/items[at0066]/value as Left_Interpretation
from EHR e
contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1]
contains OBSERVATION b_a[openEHR-EHR-OBSERVATION.visual_acuity.v1]
where a/name/value=‘Visual Acuity Report’
offset 0 limit 100
````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select     b_a/data[at0001]/events[at0134]/data[at0003]/items[at0138]/value as Test_Name,     b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Left eye’]/items[at0028]/items[at0015]/value as Left_Low_Vision_Score,     b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Right eye’]/items[at0028]/items[at0015]/value as Right_Low_Vision_Score,     b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Right eye’]/items[at0066]/value as Right_Interpretation,     b_a/data[at0001]/events[at0134]/data[at0003]/items[at0053, ‘Left eye’]/items[at0066]/value as Left_Interpretation from EHR e contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] contains OBSERVATION b_a[openEHR-EHR-OBSERVATION.visual_acuity.v1] where a/name/value=‘Visual Acuity Report’ offset 0 limit 100 HTTP/1.1
Host: rest.ehrscape.com
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

#####Return:
````json
   
 {“resultSet”: [
      “Test_Name”: {
        “@class”: “DV_CODED_TEXT”,
        “value”: “Unaided visual acuity”,
        “defining_code”: {
          “@class”: “CODE_PHRASE”,
          “terminology_id”: {
            “@class”: “TERMINOLOGY_ID”,
            “value”: “external”
          },
          “code_string”: “at0139”
        }
      },
      “Right_Low_Vision_Score”: {
        “@class”: “DV_ORDINAL”,
        “value”: 3,
        “symbol”: {
          “@class”: “DV_CODED_TEXT”,
          “value”: “HM - Hand movement”,
          “defining_code”: {
            “@class”: “CODE_PHRASE”,
            “terminology_id”: {
              “@class”: “TERMINOLOGY_ID”,
              “value”: “local”
            },
            “code_string”: “at0018”
          }
        }
      },
      “Left_Low_Vision_Score”: {
        “@class”: “DV_ORDINAL”,
        “value”: 2,
        “symbol”: {
          “@class”: “DV_CODED_TEXT”,
          “value”: “PL -  Perception of light”,
          “defining_code”: {
            “@class”: “CODE_PHRASE”,
            “terminology_id”: {
              “@class”: “TERMINOLOGY_ID”,
              “value”: “local”
            },
            “code_string”: “at0017”
          }
        }
      },
      “Left_Interpretation”: null,
      “Right_Interpretation”: null
    },
    {
      “Test_Name”: {
        “@class”: “DV_CODED_TEXT”,
        “value”: “Best corrected visual acuity”,
        “defining_code”: {
          “@class”: “CODE_PHRASE”,
          “terminology_id”: {
            “@class”: “TERMINOLOGY_ID”,
            “value”: “external”
          },
          “code_string”: “at0137”
        }
      },
      “Right_Low_Vision_Score”: {
        “@class”: “DV_ORDINAL”,
        “value”: 3,
        “symbol”: {
          “@class”: “DV_CODED_TEXT”,
          “value”: “HM - Hand movement”,
          “defining_code”: {
            “@class”: “CODE_PHRASE”,
            “terminology_id”: {
              “@class”: “TERMINOLOGY_ID”,
              “value”: “local”
            },
            “code_string”: “at0018”
          }
        }
      },
      “Left_Low_Vision_Score”: {
        “@class”: “DV_ORDINAL”,
        “value”: 4,
        “symbol”: {
          “@class”: “DV_CODED_TEXT”,
          “value”: “CF - Count fingers”,
          “defining_code”: {
            “@class”: “CODE_PHRASE”,
            “terminology_id”: {
              “@class”: “TERMINOLOGY_ID”,
              “value”: “local”
            },
            “code_string”: “at0019”
          }
        }
      },
      “Left_Interpretation”: {
        “@class”: “DV_TEXT”,
        “value”: “Poor vision.”
      },
      “Right_Interpretation”: {
        “@class”: “DV_TEXT”,
        “value”: “Corrected vision in right eye pretty still pretty poor”
      }
    ]
]
}
````

