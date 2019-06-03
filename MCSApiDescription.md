# Introduction
MCS (Meter Communications Service) has been designed as a standalone service
for providing meter data communications services to other applications (referred to as the master application hereafter).

The master application will send data collection requests to MCS via the "MCS Request API" and MCS will asynchronously send results to
the master application via the "MCS Result API".

# Basics
Each API is implemented as [Web API](https://en.wikipedia.org/wiki/Web_API) and will use JSON as the data representation format.

## Security and Authentication
To be decided.

It is suggested HTTPS is used in both directions. Authentication (i.e. securing the API with password/tokens) may be omitted and instead client IP addresses could simply be white-listed on the receiving server in each direction. 

## Response codes
Standard HTTP response codes are used in the response to all the requests in the APIs.
The following codes are used

Response code | Meaning
--------------|--------
200 OK        | Indicates that the request was successful and the response body will contain a Json object (or array of objects) with the requested data. 
400 Bad Request | Indicates the request is not valid or understood by the server. The body of the response will provide more details of why the request is considered bad.
429 Too Many Requests | Can be sent by the server to throttle inputs. Exact usage to be decided. 
500 Internal Server Error | Indicates a fault on the server. The body of the response will provide more details about the error.

## Error handling
To be decided.

# MCS Request API
## General
The master application can request for data from a meter to be collected by MCS by sending a Data Collection Request and MCS will send a response acknowledging the request or an error code if appropriate.

## Data Collection Request method

The Data Collection Request method will use a HTTP POST message and the parameters will be sent in the body of the message as a JSON object:
```
POST collection-request
```

### JSON Request Parameters

Name             | Type   | Value | Mandatory 
-----------------|--------|-------|-----------
requestReference | String | A unique ID for the request. The master application must ensure this id is unique (GUID?) | YES 
responseUrl     | String | The URL that MCS will send the results to. The URL must implement the MCS Result API | YES
priority        | Int    | An integer value (>= 1) assigning a relative priority to the request. The lower the number the higher the priority. (0 is reserved for interactive requests and will not be accepted here). | YES
delayUntil      | String | Can be used to indicate that the request shouldn't be processed until a time in the future. This can be used to indicate requests should be processed overnight. | NO
meterType       | String | Specifies the type of meter to be tested. To be specified. | YES
remoteAddress   | String | Specifies the remote address used to connect to the meter. </br>The remote address is mandatory and can take the form of a phone number (for a modem connection), an IP address and port number for a GPRS or TCP connection, or a PAKNET number.</br>A phone number must be a UK national phone number, e.g. 07711000001.</br>An IP address and port number be in the form x.x.x.x:portno, e.g. 10.2.34.4:3400.</br>The PAKNET address must be a 14 digit PAKNET number, e.g. 23000000123456 | YES 
comsSettings   | String | Normally this field should be ommitted but for cases where meters are configured in a non standard way this field can be used to override the default coms settings. This is only applicable for modem connections and can be used to specify the data bits, parity and stop bits in the form DPS, e.g. 7E1 to specify 7 stop bits, even parity and 1 stop bit. | NO 
outstationAddress | String | Specifies the outstation address/device id of the meter. | Meter dependent | NO
serialNumber   | String | The meter serial number. If included a check will be made to determine if the meter returns this serial number and an error will be reported if there is a mismatch | NO
password       | String | The meter password. | Meter dependent | NO
surveyDays     | Number | Specifies the number of days of survey data to read. If this field is missing or zero no survey data will be collected | NO
surveyDate     | String | Specifies the start date for reading survey data in the form yyyy-MM-dd. If this field is empty and surveyDays is > 0 then SURVEY_DATE will be assumed to by SURVEY_DAYS before the current day. | NO
adjustTime     | Boolean | Indicates that MCS should attempt to adjust the time of the meter. Note 1. Note 2. | NO

Notes:
1. The time for meters will only be adjusted if it varies from the MCS server time by more than a configured minimum threshold and less than or equal to a configured maximum threshold. If the time varies by less than the minimum threshold the adjustment is considered "NOT REQUIRED". If the time varies by more than the maximum threshold it is considered an error. 
2. Possibly adjusting the time should be move to a seperate method for extensibility

### JSON Response parameters
Name             | Type   | Value | Mandatory 
-----------------|--------|-------|-----------
requestReference | String | Unique ID contained in the request. | YES 
details         | String | Details relating to any errors. This parameter is only included if the response code does not equal 200. | NO

### Sample - successful case
HTTP request from master application to MCS:
```
POST https://www.coherent-research.co.uk/MCS/collection-request
content-type: application/json

{
  "requestReference": "0001",
  "responseUrl": "https://www.company.com/MCSresults",
  "priority": 0,
  "meterType": "ELSTER_A1700",
  "remoteAddress": "07777000000",
  "outstationAddress": "1",
  "serialNumber": "12345678",
  "password": "AAAA0000",
  "surveyDays": 10,
  "surveyDate": "2019-12-01",
  "adjustTime": true
}
```
Reponse from MCS:
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "requestReference": "0001"
}
```
### Sample - error case
HTTP request from master application to MCS:
```
POST https://www.coherent-research.co.uk/MCS/collection-request
content-type: application/json

{
  "requestReference": "0001",
  "responseUrl": "https://www.company.com/MCSresults",
  "meterType": "ELSTER_A1700",
  "remoteAddress": "abc",
  "outstationAddress": "1",
  "serialNumber": "12345678",
  "password": "AAAA0000",
  "surveyDays": 10,
  "surveyDate": "2019-12-01",
  "adjustTime": true

}
```
HTTP response from MCS:
```
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8

{
  "requestReference": "0001".
  "details": "Remote address is not in a recognised format"
}
```

## Data Collection Cancel method
The master application can request the cancellation of any previously requested collection. MCS will remove the matching request from the queue. If the request has already been processed (or is being processed) MCS will respond positively.

The Data Collection Cancel method will use a HTTP POST message and the parameters will be sent in the body of the message as a JSON object:
```
POST collection-cancel
```
### JSON Request Parameters

Name             | Type   | Value | Mandatory 
-----------------|--------|-------|-----------
requestReference | String | A unique ID for the request. The master application must ensure this id is unique | YES 

### JSON Response parameters
Name             | Type   | Value | Mandatory 
-----------------|--------|-------|-----------
requestReference | String | Unique ID contained in the request. | YES 
details          | String | Details relating to any errors. This parameter is only included if the response code does not equal 200. | NO

### Sample - successful case
HTTP request from master application to MCS:
```
POST https://www.coherent-research.co.uk/MCS/collection-cancel
content-type: application/json

{
  "requestReference": "0001"
}
```
HTTP response from MCS:
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "requestReference": "0001"
}
```

## Data Collection Status method
The master application can request the status of any previously requested collection. This method is not required to be used by the master application as results will be sent using the MCS Result API but it can be used by system that prefer to poll for the results instead.

The Data Collection Status method will use a HTTP GET message and the parameters will be sent as part of the URL and the response will contain the information in JSON format in the response body.
```
GET collection-status/requestReference
```
### URL Request Parameters

Name            | Type   | Value | Mandatory 
----------------|--------|-------|-----------
requestReference       | String | Unique ID contained in the request. | YES 

### JSON Response parameters - successful case
Name             | Type   | Value | Mandatory 
-----------------|--------|-------|-----------
requestReference | String | Note 1 | YES 
meterType       | String | Note 1 | YES 
remoteAddress   | String | Note 1 | YES 
comsSettings   | String | Note 1 | YES 
outstationAddress | String | Note 1 | YES 
password       | String |  Note 1 | YES 
adjustTime     | Boolean | Note 1 | YES 
surveyDays     | Number | Note 1 | YES 
surveyDate     | String | Note 1 | YES 
result         | String | Indicates the overall result of the collection request. Values are "PENDING" (i.e. the collection  has not been performed yet), "SUCCESS", "PARTIAL SUCCESS" (note 3) or "ERROR: details" where details describe the problem that caused the test to fail. | YES
collectionStartTime  | String | Time time the collection started. In the case of multiple retries this will be the start of the first attempt. Note 2 | NO
collectionEndTime  | String | The time the collection ended. In the case of multiple retries this will be the end of the last attempt. Note 2 | NO. 
connectionStartTime | String | The time that the communication channel was opened. In the case of multiple retries this will be the start of the last attempt. Note 2 | NO
connectionEndTime | String | The time that the communication channel was closed. In the case of multiple retries this will be the  of the last attempt. Note 2 in the same format as above. | NO
serialNumber   | String |  The serial number received from the meter, or "ERROR: details" if it was not possible to fetch the serial number from the meter or if the serial number from the request is included and does not match the serial number read from the meter. | NO 
meterTime      | String | The meter date/time received from the meter with an offset in seconds from the time the test took place in the format YYYY-MM-DDTHH:mm:ss +/- Ns, e.g. 2014-10-31T23:33:32 -203s (indicates that the time was 203 seconds behind the real time when it was read). If no time was read this will contain "ERROR: details" where details describe the problem. | NO
timeAdjustmentResult | String | "NOT REQUIRED", "SUCCESS", "ERROR: details". The timeAdjustmentResult can be an ERROR if it was not possible to adjust the time or the variation was above the maximum threshold. | NO
registerValues | Array | An array of Register Value objects. See below | NO
surveyData | Array | An array of Register Survey Data objects. See below | NO

Notes: 
1. The parameters from the original Data Collection Request (or their interpretation in the case where their values were implicit) are repeated here. 
2. All times are in UTC and in the format YYYY-MM-DDTHH:mm:ssZ
3. A collection will be considered partially successful if some, but not all, of the data was collected.

### Register Value Object
A Register Value Object contains an instantaneous value read from the meter.

Name            | Type   | Value | Mandatory 
----------------|--------|-------|-----------
name            | String | The register name (depending on the meter type) | YES
value           | Number | The value of the register
units           | String | The units of the register

### Register Survey Data Object
A Register Survey Value Object contains survey data for an individual register/channel from the meter:

Name            | Type   | Value | Mandatory 
----------------|--------|-------|-----------
name            | String | The register name (depending on the meter type) | YES
units           | String | The units of the register | YES
readings        | JSON object | An array of readings | YES

### Survey Reading Object
Name            | Type   | Value | Mandatory 
----------------|--------|-------|-----------
timestamp       | String | The time of the reading | YES
value           | Number | The value of the register | YES

### Sample - successfully completed collection
HTTP request from master application to MCS:
```
GET https://www.coherent-research.co.uk/MCS/collection-status/0001
```
HTTP response from MCS:
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "requestReference": "0001",
  "meterType": "ELSTER_A1700",
  "remoteAddress": "07777000000",
  "outstationAddress": "1",
  "serialNumber": "12345678",
  "password": "AAAA0000",
  "surveyDays": 1,
  "surveyDate": "2019-01-01",
  "resultSummary": "SUCCESS",
  "collectionStartTime": "2019-01-02T04:00:00",
  "collectionEndTime": "2019-01-02T04:01:30",
  "connectionStartTime": "2019-01-02T04:02:00",
  "connectionEndTime": "2019-01-02T04:01:28",
  "serialNumber": "12345678",
  "meterTime": "2019-01-02T03T04:02:15 +10s",
  "registerValues": [
    {
      "name": "kWh Import",
      "timestamp": "2019-01-02T04:02:10Z",
      "value": "758",
      "units": "kWh"
    },
    {
      "name": "kvarh Q1",
      "timestamp": "2019-01-02T04:02:11Z",
      "value": "1190",
      "units": "kvarh"
    }
  ],
  "surveyData": [
      {
        "name": "kWh Import",
        "units": "kWh",
        [
          {
            "timestamp": "2019-01-01T00:00:00Z",
            "value": "640"
          },
          {
            "timestamp": "2019-01-01T00:30:00Z",
            "value": "645"
          },
          ...
          {
            "timestamp": "2019-01-01T23:30:00Z",
            "value": "700"
          }       
        ]
      },
      {
        "name": "kvarh Q1",
        "units": "kvarh",
        [
          {
            "timestamp": "2019-01-01T00:00:00Z",
            "value": "900"
          },
          {
            "timestamp": "2019-01-01T00:30:00Z",
            "value": "910"
          },
          ...
          {
            "timestamp": "2019-01-01T23:30:00Z",
            "value": "1000"
          }       
        ]
      },
  ]
}
```

### Sample - successfully completed time adjustment (no survey data)
From master application to MCS:
```
GET https://www.coherent-research.co.uk/MCS/collection-status/0001
```
From MCS to master application:
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "requestReference": "0001",
  "meterType": "ELSTER_A1700",
  "remoteAddress": "07777000000",
  "outstationAddress": "1",
  "serialNumber": "12345678",
  "password": "AAAA0000",
  "surveyDays": 1,
  "surveyDate": "2019-01-01",
  "resultSummary": "SUCCESS",
  "collectionStartTime": "2019-01-02T04:00:00",
  "collectionEndTime": "2019-01-02T04:01:30",
  "connectionStartTime": "2019-01-02T04:02:00",
  "connectionEndTime": "2019-01-02T04:01:28",
  "serialNumber": "12345678",
  "meterTime": "2019-01-02T03T04:02:15 +500s",
  "timeAdjustmentResult": "SUCCESS"
}
```
# MCS Result API
## General
MCS will send the results of any data collection request to the provided URL using the MCS Result API. 

## Data Collect Result method

The Data Collection Result method will use a HTTP POST message and the parameters will be sent in the body of the message as a JSON object. The object format is exactlty as in the response to the MCS Collection Status request above.
```
POST collection-result
```
### URL Request Parameters
See MCS Collection Status request above.

### JSON Response parameters
Name            | Type   | Value | Mandatory 
----------------|--------|-------|-----------
requestReference       | String | Unique ID contained in the request. | YES 
details         | String | Details relating to any errors. This parameter is only included if the response code does not equal 200. | NO

### Sample - successfully completed collection
HTTP request from MCS to master application:
```
POST https://www.coherent-research.co.uk/MCS/collection-result
Content-Type: application/json; charset=utf-8

{
  "requestReference": "0001",
  "meterType": "ELSTER_A1700",
  "remoteAddress": "07777000000",
  "outstationAddress": "1",
  "serialNumber": "12345678",
  "password": "AAAA0000",
  "surveyDays": 1,
  "surveyDate": "2019-01-01",
  "resultSummary": "SUCCESS",
  "collectionStartTime": "2019-01-02T04:00:00",
  "collectionEndTime": "2019-01-02T04:01:30",
  "connectionStartTime": "2019-01-02T04:02:00",
  "connectionEndTime": "2019-01-02T04:01:28",
  "serialNumber": "12345678",
  "meterTime": "2019-01-02T03T04:02:15 +10s",
  "registerValues": [
    {
      "name": "kWh Import",
      "timestamp": "2019-01-02T04:02:10Z",
      "value": "758",
      "units": "kWh"
    },
    {
      "name": "kvarh Q1",
      "timestamp": "2019-01-02T04:02:11Z",
      "value": "1190",
      "units": "kvarh"
    }
  ],
  "surveyData": [
      {
        "name": "kWh Import",
        "units": "kWh",
        [
          {
            "timestamp": "2019-01-01T00:00:00Z",
            "value": "640"
          },
          {
            "timestamp": "2019-01-01T00:30:00Z",
            "value": "645"
          },
          ...
          {
            "timestamp": "2019-01-01T23:30:00Z",
            "value": "700"
          }       
        ]
      },
      {
        "name": "kvarh Q1",
        "units": "kvarh",
        [
          {
            "timestamp": "2019-01-01T00:00:00Z",
            "value": "900"
          },
          {
            "timestamp": "2019-01-01T00:30:00Z",
            "value": "910"
          },
          ...
          {
            "timestamp": "2019-01-01T23:30:00Z",
            "value": "1000"
          }       
        ]
      },
  ]
}
```
HTTP response from master application:
```
HTTP/1.1 200 OK
```
