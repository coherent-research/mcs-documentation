# Introduction
DCE (Data Collection Engine) has been designed as a standalone service
for providing meter data collection services to other applications (referred to as the master application hereafter).

The master application will data collection requests to DCE via the "DCE Request API" and DCE will asynchronously send results to
the master application via the "DCE Result API".


# Basics
Each API is implemented as [Web API](https://en.wikipedia.org/wiki/Web_API) and will use JSON as the data representation format.


## Authentication
To be decided.

## DCE Request API
### General
The master application can request for meter to be collected by DCE by sending a Data Collection Request and DCE will send a response acknowledging the request or an error code if appropriate.

### Data Collection Request
The Data Collection Request will use a HTTP POST message and the parameters will be sent in the body of the message as a JSON object:
```
POST collection-request
```

### JSON Parameters

Name            | Type   | Value | Mandatory 
----------------|--------|-------|-----------
requestId       | String | A unique ID for the request. The master application must ensure this id is unique | YES 
meterType       | String | Specifies the type of meter to be tested. See XXX for allowed values.
remoteAddress   | String | Specifies the remote address used to connect to the meter. </br>The remote address is mandatory and can take the form of a phone number (for a modem connection), an IP address and port number for a GPRS or TCP connection, or a PAKNET number.</br>A phone number must be a UK national phone number, e.g. 07711000001.</br>An IP address and port number be in the form x.x.x.x:portno, e.g. 10.2.34.4:3400.</br>The PAKNET address must be a 14 digit PAKNET number, e.g. 23000000123456 | YES 
comsSettings   | String | Normally this field should be ommitted but for cases where meters are configured in a non standard way this field can be used to override the default coms settings. This is only applicable for modem connections and can be used to specify the data bits, parity and stop bits in the form DPS, e.g. 7E1 to specify 7 stop bits, even parity and 1 stop bit. | NO 
outstationAddress | String | Specifies the outstation address/device id of the meter. | Meter dependent
serialNumber   | String | The meter serial number. If included a check will be made to determine if the meter returns this serial number and an error will be reported if there is a mismatch | NO
password       | String | The meter password. | Meter dependent
surveyDays     | Number | Specifies the number of days of survey data to read. If this field is missing or zero no survey data will be collected | NO
surveyDate     | String | Specifies the start date for reading survey data in the form yyyy-MM-dd. If this field is empty and surveyDays is > 0 then SURVEY_DATE will be assumed to by SURVEY_DAYS before the current day. 
adjustTime     | Boolean | Indicates that DCE should attempt to adjust the time of the meter | NO

### Sample - successful case
From master application to DCE:
```
POST https://www.coherent-research.co.uk/DCE/collection-request
content-type: application/json

{
  "requestId": "0001",
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
From DCE to master application:
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Set-Cookie: COHERENT-DCSV3 ...

{
  "requestId": "0001"
}
```
### Sample - error case
From master application to DCE:
```
POST https://www.coherent-research.co.uk/DCE/collection-request
content-type: application/json

{
  "requestId": "0001",
  "requestId": "0001",
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
From DCE to master application:
```
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
Set-Cookie: COHERENT-DCSV3 ...

{
  "requestId": "0001".
  "details": "Remote address is not in a recognised format"
}
```

### Response codes
Standard HTTP response codes are used in the response to all the request.
The following codes are used

Response code | Meaning
--------------|--------
200 OK        | Indicates that the query was successful and the response body will contain a Json object (or array of objects) with the requested data. 
400 Bad Request | Indicates the request is not valid or understood by DCE. The body of the response will provide more details of why the request is considered bad.
401 Unauthorized | Indicates that the user has not provided a valid authentication token. See above
500 Internal Server Error | Indicates a fault on the server. The body of the response will provide more details about the error.
