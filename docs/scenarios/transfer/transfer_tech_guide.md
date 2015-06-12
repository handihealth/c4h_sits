##Code4Health Ehrscape ‘Transfer of Care Summary' Technical Guide

This document describes the series of [Ehrscape API](https://dev.ehrscape.com/api-explorer.html) calls required for the 'Handover Summary' scenario and assumes a base level of understanding of that API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

The steps covered are...  

* Open the Ehrscape Session

* Retrieve Patient's ehrId from Ehrscape, based on their subjectId (NHSNumber)
* If the patent does not have an EHR, create a new EHR in Ehrscape, based on their subjectId (NHSNumber)

* Retrieve the compositionID of Patient's most recent Allergy List composition
* Retrieve the Patient's most recent Allergy List composition
* Persist an updated AllergyList Composition as a PUT i.e update the same composition with a new version.
* Run an AQL query which returns a 'flat list' of key datapoints from the Allergy List for read-only display.

* Repeat the above for Problems and Relevant Contacts Lists

* Retrieve the Patient's most recent Medication List compositions - will be empty
* Persist an new Medication List Composition as a POST i.e create new Medication list instance.
* Persist an updated Medication List Composition as a PUT i.e update the same composition with a new version.

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

"subjectId": "7430555"  
"subjectNamespace": "uk.nhs.nhsnumber"  

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

If the patient already has an EHR and you have a patient's externalId e.g. their NHS Number, you may retrieve the patient's internal `ehrID` asociated with that subjectId. 
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

###B2. Create a new Patient 'EHR' in Ehrscape based on their subjectId and subjectNamespace

Now that an ehrscape session is available, the first task is to create an 'EHR' for the dummy patient. In openEHR systems like Ehrscape, the EHR refers to the complete set of records for a single pateient i.e the top-level container. When we create the EHR, we are returned a unique ehrId for this patient. The ehrID is a unique string which, for security reasons, cannot be assoicated with the patient, if for instance their openEHR records were leaked - the openEHR data does not contain references to any external identifiers such as an NHS or hospital number. All access to patient's EHR much be via their ehrId

#####Call: Create an openEHR 'EHR' for the specified subject ID and namespace.
````
POST /rest/v1/ehr/?subjectId=63436&subjectNamespace=ehrscape
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
#####Return:
````json
{
  "ehrId": "0da489ee-c0ae-4653-9074-57b7f63c9f16" // Note that the ehrId will be different in your domain.
}
````


###C. Retrieve the compositionId of Patient's most recent 'Allergies list' composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the containers for all stored openEHR patient data. i.e. an openEHR EHR is a colleciton of compositions belonging to a single patient.

The `name/value` of the Composition is the root name of the composition archetype `Allergies list` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

Creating AQL statements is outside of the scope of this document. You should be able to use or adapt those provided for this scenario.

#####AQL statement

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time
from EHR e[ehr_id/value={{ehrId}}]
contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary.v1]
where a/name/value='Allergies list'
order by a/context/start_time/value desc
offset 0 limit 1
````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `uid_value` element in the response is the unique identifier for the composition and `context_start_time` is the time that the document was authored.

Only a single row should be returned, as we are adopting a

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time from EHR e[ehr_id/value= '{{ehrId}}'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.care_summary_.v1] where a/name/value='Allergies list' order by a/context/start_time/value desc offset 0 limit 1
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
    "meta": {
        "href": "https://rest.ehrscape.com/rest/v1/composition/798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2",
        "precedingHref": "https://rest.ehrscape.com/rest/v1/composition/798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer::1"
    },
    "format": "FLAT",
    "templateId": "LCR Allergies List.v0",
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


###E. Persist an updated version of the Allergies List Composition as a PUT

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  'target composition' data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example Ehrscape Flat JSON Composition](/technical/instances/LCR Handover/AllergiesList_2FLAT.json)  

In this scenario we want to **maintain only a single instance of the Allergies List** and so will use a PUT call to update the previous version. The previous version remains avaialble for audit purpsoes but would not normally be found by routine querying.

This is in contrast to e.g a Nursing Observation Encouter, where every Encounter merits a separate instance of the document and a POST would norrmally be used, with PUT updates only being used to correct errors or to perform 'sign-off'.

**When a PUT call is used the compositionId that is passed in must include the domain and version suffix  of the last version**

e.g. `798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2`

#####Call: Updates a new openEhr composition and returns the new CompositionId
````
PUT /rest/v1/composition/{{compositionId}}?format=FLAT&templateId=LCR Allergies List.v0

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

###F. Run an AQL call displaying a flat list from the Allergies List composition

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

The `name/value` of the Composition is the root name of the composition archetype `Allergies list` (case-sensitive). In a real-world example we would query on other factors to ensure we had the 'correct' list.

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
where a/name/value='Allergies list'
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
where a/name/value='Allergies list'
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

###E. Repeat the steps above for Problem List

####Problem List


###F. Close the Ehrscape session

The last step in working with Ehrscape is to close the session.

#####Call: Create a new openEHR session:
 ````
 DELETE /rest/v1/session?sessionId={{sessionId}}
 ````
#####Returns:
````json
{
  "sessionId": "2dcd6528-0471-4950-82fa-a018272f1339"
}
````

##G. Other Services

These are other non-Ehrscape API services.

###1. Access the ALISS 'Local community resources' service

[ALISS](http://www.aliss.org/) (A Local Information System for Scotland) is a search and collaboration tool for Health and Wellbeing resources. Originally developed in Scotland, it is now us used in regions across the UK.

Further search options e.g by locality are avaiable from the [ALISS API documentation site](http://aliss.readthedocs.org/en/latest/search_api/).

#####Call: Search the ALLIS database for resources related to asthma
````
GET http://www.aliss.org/api/v2/search/?q=asthma

Headers:
 None
````
#####Return:
````json
{
    "count": 91,
    "next": "http://www.aliss.org/api/v2/search/?page=2&q=asthma",
    "previous": null,
    "results": [
        {
   "id": 351,
   "title": "Asthma - a booklist to help you manage the condition - from Renfrewshire Libraries",
   "description": "Renfrewshire Libraries have developed a facility for creating booklists online via a simple search function. One can find out whether Renfrewshire Libraries has a particular title, whether it is available and from which library, etc.\r\n\r\nThis one is about Asthma.",
   "uri": "https://libcat.renfrewshire.gov.uk/vs/List.csp?SearchT1=asthma&Index1=Keywords&Database=1&PublicationType=NoPreference&Location=NoPreference&SearchMethod=Find_1&SearchTerm1=asthma&OpacLanguage=eng&Profile=Default&EncodedRequest=*D5*2C2*D8*EE*C4*D4*5B*B2*9C*00*26*D8*5Cb*8F&EncodedQuery=*D5*2C2*D8*EE*C4*D4*5B*B2*9C*00*26*D8*5Cb*8F&Source=SysQR&PageType=Start&PreviousList=RecordListFind&WebPageNr=1&NumberToRetrieve=20&WebAction=NewSearch&StartValue=0&RowRepeat=0&ExtraInfo=SearchFromList&SortIndex=Author&SortDirection=1",
   "locations": [
       {
           "lat": 55.8561,
           "lon": -4.4057,
           "formatted_address": "Renfrew South & Gallowhill ward, Renfrewshire, PA3 4SF"
       }
   ],
   "tags": [
       "books",
       "library",
       "Asthma",
       "asthma"
   ],
   "owner": "Living Well at the Library",
   "event_start": null,
   "event_end": null,
   "created_on": "2011-07-13T12:46:44.732000+00:00",
   "modified_on": "2014-04-22T07:15:37.249920+00:00"
},

````

###2. Access the NHS Choices Patient advice service - HTML

This API call retreives NHS Choices patient information page for Asthma in HTML format.

#####Call: Search the NHS Choices database for resources related to asthma, returning an HTML page
````
GET http://v1.syndication.nhschoices.nhs.uk/conditions/articles/asthma/introduction?apikey=GFENGBJA

Headers:
 None
````
#####Return:
````html
<html xmlns="http://www.w3.org/1999/xhtml" >
<head>
    <title>NHS Choices Syndication</title>
    <meta http-equiv="Content-Style-Type" content="text/css" />
.....
</head>
````

###3. Access the NHS Choices Patient advice service - XML

This API call retreives NHS Choices patient information for Asthma in XML format.

#####Call: Search the NHS Choices database for resources related to asthma, returning an XML document.
````
GET http://v1.syndication.nhschoices.nhs.uk/conditions/articles/asthma/introduction.xml?apikey=GFENGBJA

Headers:
 None
````
#####Return:
````xml
<?xml version="1.0" encoding="utf-8"?>
     <feed xmlns="http://www.w3.org/2005/Atom"><title type="text">NHS Choices - Introduction</title><id>uuid:6e093010-1013-46b4-b231-42795514008d;id=1950</id><rights type="text">© Crown Copyright 2009</rights><updated>2015-02-11T15:54:40Z</updated><category term="asthma" /><logo>http://www.nhs.uk/nhscwebservices/documents/logo1.jpg</logo>
	...
````

###4. Access the Indizen SNOMED CT Terminology browser service

This API call to the [Indizen](www.indizen.com/index.php/en/) Terminology serviceretreives SNOMED CT terms matching ``asthma`` in XML format.

The baseURL for Indizen ITSNode is http://www.itserver.es:9080/

#####Call: Search the Indizen terminology service database for terms matching asthma
````
GET /ITSNode/rest/snomed/descriptions?matchvalue=asthma&referencelanguage=en&fuzzy=false&numberOfElements=110&filtercomponent=all&spellingCorrection=true

Headers:
 Authorization: Basic aGFuZGlob3BkOmRmZzU4cGE=
````
#####Return:
````xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<sctDescriptionss>
	<description>
		<conceptid>195967001</conceptid>
		<descriptionid>301485011</descriptionid>
		<descriptionstatus>0</descriptionstatus>
		<descriptiontype>1</descriptiontype>
		<inititalcapitalstatus>0</inititalcapitalstatus>
		<languagecode>en</languagecode>
		<similarity>100.0</similarity>
		<sourceName/>
		<term>Asthma</term>
	</description>
	<description>
		<conceptid>161527007</conceptid>
		<descriptionid>251716013</descriptionid>
		<descriptionstatus>0</descriptionstatus>
		<descriptiontype>1</descriptiontype>
		<inititalcapitalstatus>1</inititalcapitalstatus>
		<languagecode>en</languagecode>
		<similarity>98.0</similarity>
		<sourceName/>
		<term>H/O: asthma</term>
	</description>
	<description>
		<conceptid>160377001</conceptid>
		<descriptionid>249995012</descriptionid>
		<descriptionstatus>0</descriptionstatus>
		<descriptiontype>1</descriptiontype>
		<inititalcapitalstatus>1</inititalcapitalstatus>
		<languagecode>en</languagecode>
		<similarity>98.0</similarity>
		<sourceName/>
		<term>FH: Asthma</term>
	</description>
   ...
</sctDescriptionss>
````
