> ## Author Notes
> - This is specific to the Zenoss RM event API and a separate doc will be created for the Zenoss Cloud events API once it's available to me. 
> - If copying this to a GitHub Wiki replace `[WIKIURL]` with the URL of the GitHub wiki page. If not using on a wiki the Table of Contents will need modified or deleted. 
> - To customize for your environment replace `[URL]` with your Zenoss URL i.e. `https://zenny.zenoss.io`
> - [Live GitHub Wiki version](https://github.com/dan-smalley/ZenossGuides/wiki/Zenoss-Event-API-Guide)

# Table of Contents
- [Intro]([WIKIURL]#intro)
- [Access]([WIKIURL]#access)
- [API Payload]([WIKIURL]#api-payload)
- [Examples]([WIKIURL]#examples)
- [Zenoss Event Flow]([WIKIURL]#zenoss-event-flow)
    - [Event Fields]([WIKIURL]#event-fields)
    - [Deduplication]([WIKIURL]#deduplication)
    - [Other Notes]([WIKIURL]#other-notes)
- [License]([WIKIURL]#License)

# Intro
Welcome to the Zenoss Event API Guide! 

This page provides a crash course in how to use the Event Console Router API to ingest events into Zenoss, as well as some background information on how Zenoss processes events that you will need to understand in order to use the API effectively. Currently examples are provided in cURL, Python(requests) and PowerShell. but can be easily translated into other languages. 

> Tip: If you're just getting started with APIs I highly recommend the excellent and free software [Postman](https://www.postman.com/) to fine tune your API calls before moving them into your code. 

# Access 

To generate an API key follow the Zenoss [Collection Zone API keys](https://docs.zenoss.io/admin/clients/cz-api.html) guide. 

> Note: Users will need the ZenOperator privilege to create or manipulate events.  It's important to understand that this will allow the user to manipulate **all** events and generate them as they please.  If you're concerned about users having UI access to do this you can maintain the accounts centrally and only share the API keys.  

Requests will always be sent to the event_console router using the URL `[URL]/cz0/zport/dmd/evconsole_router`.

> Note: The API specific URL can be used for certain on-prem installations and is preferred when available

# API Payload

### JSON Payload
Sending an event to the Zenoss API simply requires issuing a POST request to the desired input with the following JSON payload in the body.
```json
{
    "action": "EventsRouter",
    "method": "add_event",
    "data": [
        {
            "device": "Device Shortname",
            "component": "Sub component (optional)",
            "summary": "A short description (max 128 characters)",
            "message": "A long descriptive message (optional, max 4096 characters)",
            "severity": "Clear, Debug, Info, Warning, Error, Critical",
            "evclasskey": "required but ususally left blank",
            "evclass": "Consult with Zenoss admins for appropriate event class",
            "eventKey": "unique key, see 'Deduplication' below"
        }
    ],
    "tid": 1
}
```

The `action` and `method` should always remain `EventsRouter` and `add_event`. The `tid` is an arbitrary integer. 

### Successful Response
If event creation was successful Zenoss will respond with the following JSON
``` json
{
    "uuid": "7b29408b-fbcd-405c-8a96-51d25d442a91",
    "action": "EventsRouter",
    "result": {
        "msg": "Created event",
        "success": true
    },
    "tid": 1,
    "type": "rpc",
    "method": "add_event"
}
```

### Failure Response 
If event creation fails Zenoss will provide information on the failure (assuming the API call was not malformed). 
``` json
{
    "uuid": "078ea670-a732-48be-b099-6b630d7afb07",
    "action": "EventsRouter",
    "result": {
        "msg": "TypeError: add_event() takes at least 6 arguments (2 given)",
        "type": "exception",
        "success": false
    },
    "tid": 1,
    "type": "rpc",
    "method": "add_event"
}
```

> Tip: When incorporating Zenoss API calls into your code it is often a good idea to compact the JSON payload for easy viewing. A tool like [JSON formatter](https://jsonformatter.org/) can be very helpful for this. 

# Examples

## cURL
``` bash
curl --location --request POST '[URL]/cz0/zport/dmd/evconsole_router' \
--header 'cache: no-cache' \
--header 'Content-Type: application/json' \
--header 'z-api-key: [YOUR KEY]' \
--data-raw '{"action":"EventsRouter","method":"add_event","data":[{"summary":"Testing Zenoss API","message":"A long message","device":"zenossUtilityBox","component":"testComponent","severity":"Info","evclasskey":"","evclass":"/API/CMDB","eventKey":"unique key"}],"tid":1}'

```

## Python(Requests) 
``` python
import requests

url = "[URL]/cz0/zport/dmd/evconsole_router"

payload="{\"action\":\"EventsRouter\",\"method\":\"add_event\",\"data\":[{\"summary\":\"Testing Zenoss API\",\"message\":\"A long message\",\"device\":\"zenossUtilityBox\",\"component\":\"testComponent\",\"severity\":\"Info\",\"evclasskey\":\"\",\"evclass\":\"/API/CMDB\",\"eventKey\":\"unique key\"}],\"tid\":1}"
headers = {
  'cache': 'no-cache',
  'Content-Type': 'application/json',
  'z-api-key': '[YOUR KEY]'
}

response = requests.request("POST", url, headers=headers, data=payload)

print(response.text)

```

## PowerShell

```powershell
$headers = New-Object "System.Collections.Generic.Dictionary[[String],[String]]"
$headers.Add("cache", "no-cache")
$headers.Add("Content-Type", "application/json")
$headers.Add("z-api-key", "[YOUR KEY]")

$body = "{`"action`":`"EventsRouter`",`"method`":`"add_event`",`"data`":[{`"summary`":`"Testing Zenoss API`",`"message`":`"A long message`",`"device`":`"zenossUtilityBox`",`"component`":`"testComponent`",`"severity`":`"Info`",`"evclasskey`":`"`",`"evclass`":`"/API/CMDB`",`"eventKey`":`"unique key`"}],`"tid`":1}"

$response = Invoke-RestMethod '[URL]/cz0/zport/dmd/evconsole_router' -Method 'POST' -Headers $headers -Body $body
$response | ConvertTo-Json
```

> Tip: In all instances it's important to properly escape the double quotes in the JSON payload. This is the most common source of errors I see.


# Zenoss Event Flow
It's important to keep in mind these (grossly oversimplified) concepts while implementing your API calls into Zenoss in order for them to be effective and produce the results you want. 

## Event Fields
| Key | Description/Notes | Required | Max Chars |
|-|-|:-:|:-:|
| device | Can be short name or FQDN. Device must be in Zenoss | Y | -- |
| component | Required field but can be blank. The component can be a real component in Zenoss i.e. A filesystem, or the component can be arbitrary i.e. The name of the script. | Y | 255 |
| severity | Clear, Debug, Info, Warning, Error, Critical - Determines the severity of the ticket created (see matrix below) | Y | -- |
| summary | A short, pithy description of the event | Y | 255 |
| message | A longer free-form message with additional details or mitigation steps. If not sent the Summary field will be copied into the Message. Line breaks can be included using "`\n`". | N | 4096 |
| evclass | The Event Class used to organize events. Work with a Zenoss admin to determine which event class should be used or if a new one should be created. | Y | -- |
| eventKey | Primary field used for event deduplication. See 'Deduplication' section below for more information. | Y | 128 |
| evclasskey | A required field but can be sent in empty. Value only needed if you are using the event classification function. | Y | 128 |

In addition arbitrary fields can be sent in as well which will appear in the event and can be accessed in Transforms and Notifications using `evt.[KEY]`

## Deduplication
Zenoss concatenates several fields sent in with your event to create a field called the dedupid. This is the unique identifier of your event. **Events that come in with the same dedupid will increment the event count rather than create a new event.** This is useful when you want to send in an event periodically until the condition has cleared but don't want separate incidents. It's also useful for consolidating several unique events into one to avoid excess tickets. 

The dedupid consists of the following fields. If all fields match the event's count is increased. If the combination is unique a new event is generated 

`device|component|evclass|severity|eventkey OR summary`

Take special note of the last field. If the event key is sent in it is used in place of the Summary for deduplication. Practically this means if you use an event key you can update the Summary on subsequent events without generating a new incident. For example you could include the depth of a queue in the summary and the depth would update each time a new event was sent in.

## Event severity matrix

While event severity has likely already been determined by your organization I find this hierarchy helps make the most of the event severities available in Zenoss and some of the dashboarding features that go along with them. 
| Zenoss Severity | Notes: |
|:---:|---|
| Critical | Should be reserved for major system outages |
| Error | Severe problems that require paging on-call |
| Warning | Problems that require an incident and notification but not immediate attention |
| Info | Creates an event in Zenoss but does not generate a ticket. Ticket generation can be enabled if requested. |
| Debug | Creates an event in Zenoss that is not shown in the UI using default filters.  |
| Clear |  Clears the event in Zenoss and closes the corresponding incident. |

## Other notes
Zenoss processes events individually and in isolation. It can't look backward at previous events (except to clear them).

i.e. "if X event and Y event make Z event." is not possible 

## License
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>. 
Author: [Dan Smalley](https://github.com/dan-smalley) 
