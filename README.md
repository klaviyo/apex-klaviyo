# Klaviyo Apex Connector

## Overview
The Klaviyo Apex Connector (KAC) is a set of code files which can be used to connect Salesforce Dot Com's CRM (SFDC) to Klaviyo using a webhook-like integration. In its full functionality, KAC uses triggers for inserts/updates on records to detect changes in SFDC, sends the updated SObjects to a queueable processor class, collects any data needed for the payload (ie. the email address to send the request for and the watched fields on that SObject), and sends an HTTP callout via a class that wraps our Track and Identify APIs. It will be useful to [review how the Track and Identify APIs work](https://help.klaviyo.com/hc/en-us/articles/115000751052-Klaviyo-API-Reference-Guide) and the different types of reserved properties that can be sent with each payload before attempting to create a processor method.

## Components
KAC consists of 4 code components
1) KacLeadTrigger
2) KacProcessor
3) KacApiWrapper
4) KacTests

and 4 custom Metadata Types (MDTs)
1) KacDisableTrigger__mdt
  - KacDisableTriggerMap
2) KacLeadField__mdt
  - KacLeadFieldMap
3) KacApiKeys__mdt
  - TestAccount
  - ProductionAccount
4) KacCodeSetting__mdt
  - KacSettingsConfig

## Usage
There are some functionality and examples prebuilt in this code package that will allow you to slot in your own triggers and processor methods by copying the example for Lead records. There is also the code file for KacApiWrapper which can be used by itself to send Track and Identify requests.

### KAC Trigger Usage
In order to implement webhook-like functionality, you will need to set up some way to watch for updates in the SFDC environment. The easiest way to do this using only APEX is to:
1) Set up a trigger that watches a specific SObject for inserts/updated.
2) Check each SObject for updates to a set of "watched" fields specified by an MDT record (eg. `KacLeadFieldMap` record on `KacLeadField__mdt`).
3) Send the set of updated SObjects to a queueable (to run asynchronously) processor that will process the data for HTTP calouts.

### KacProcessor Usage
Using the processor is the more complicated part of the implementation of this connector, it is how you convert the data retrieved from the updated/inserted SObjects into a trackable format for our APIs. The `processLead()` example included in this package shows how to programmatically retrieve "watched" fields (eg. `KacLeadFieldMap` record on `KacLeadField__mdt`) from the set of Lead records and construct them into Maps for Identify API payloads but more advanced functionality can also be included (eg. loading in formula fields or using lookups and SOQL to retrieve data from other types of related SObject records.

### KacApiWrapper Usage
If you'd like to build your own connector or integration in some way just using our native Track and Identify APIs, the API wrapper can also be used directly. This library wraps our Track and Identify APIs in APEX methods and uses `Map<String, String>`'s as an APEX-native analog for JSON.

#### Initialize the wrapper
Initialize ApexKlaviyoAPI wrapper class. It takes a Map of public and private api keys.
```
Map<String, String> apiKeys = new map<String, String> {
    'PublicKey' => 'abc123',
    'PrivateKey' => 'pk_abcdefghijklmnopqrstuvwxyz12345678'
};
ApexKlaviyoAPI klaviyoClient = new ApexKlaviyoAPI(apiKeys);
```
#### Build and send Track & Identify requests
As an analog for JSON, this wrapper uses Maps which are a more native JSON-like data structure in APEX. Build the Map as you'd normally build a JSON payload fro Track/Identify then send the request using methods on the wrapper object initialized earlier.

##### Identify
Identify only has a single method that takes a map of customer properties
```
// Build Identify properties
Map<String, Object> customerProperties = new map<String, Object> {
    '$email' => 'walid.bendris+apex@klaviyo.com',
    '$first_name' => 'Apex',
    '$last_name' => 'Code',
    'ApexIdentify' => true,
    'ProfileApexBoolean' => true,
    'ProfileApexString' => 'test',
    'ProfileApexNumber' => 1,
    'ProfileApexDate' => '2020-01-01',
    'ProfileApexListOfBooleans' => new list<Boolean>{true,false,true},
    'ProfileApexListOfStrings' => new list<String>{'one','two','three'},
    'ProfileApexListOfNumbers' => new list<Integer>{1,2,3}
};
// Send Identify request
klaviyoClient.identify(customerProperties);
```
##### Track
Track has 4 different methods that allow for more flexibility when generating an event. The examples below review the 2 most common use-cases, tracking a real-time event and a historical event (ie. an event with a timestamp).

Track a real-time event using the metric name, customer properties Map, and an event properties Map
```
//Build Track properties
Map<String, Object> eventPayload = new map<String, Object> {
    '$value' => 100,
    'EventApexBoolean' => true,
    'EventApexString' => 'test',
    'EventApexNumber' => 1,
    'EventApexDate' => '2020-01-01',
    'EventApexListOfBooleans' => new list<Boolean>{true,false,true},
    'EventApexListOfStrings' => new list<String>{'one','two','three'},
    'EventApexListOfNumbers' => new list<Integer>{1,2,3}
};
// Send Track request
klaviyoClient.track('Test Event', customerProperties, eventPayload);
```

Track a historical event using the metric name, customer properties Map, an event properties Map, and an epoch timestamp
```
// Get time a week ago
Datetime dt = Datetime.now();
Long timestamp = dt.getTime() / 1000 - 604800;
// Send historical Track request
klaviyoClient.track('Past Test Event',customerProperties,eventPayload,timestamp);
```
