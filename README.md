# DMPsee: Environmentally Responsible Event Propagation Standard for Data Management Plans

**Authors:**

- Andrea Davanzo [0009-0000-5170-1737](https://orcid.org/0009-0000-5170-1737), University of Edinburgh
- Don Stuckey [0009-0008-1697-9432](https://orcid.org/0009-0008-1697-9432), University of Edinburgh

Last revision: 2026-01-15

> [!NOTE]
> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119)

## Introduction

DMP Sent Events (DMPsee) is an **open standard** designed to propagate updates
from DMP platforms to other systems via webhooks.
These events can then be consumed by any interested subscriber.

This publish/subscribe model enables a loosely coupled and scalable approach
to keeping dependent systems informed of DMP changes across platforms.

As an open standard, DMPsee promotes interoperability, avoids vendor lock-in,
and ensures that diverse systems can seamlessly exchange DMP updates.

### The Environmentally Responsible Principle

One of the guiding principles of the DMPsee standard is to minimize unnecessary
resources and data transfer, thereby reducing environmental impact.
Although each individual request may be small, the cumulative effect of redundant
information exchanged billions of times per year becomes significant in terms of
bandwidth consumption, energy usage, and associated carbon emissions.
For this reason, the standard is designed with environmental responsibility as
a primary concern, while still acknowledging the social and economic dimensions of sustainability.
By privileging efficiency and avoiding wasteful data exchange, DMPsee demonstrates
that digital infrastructures can be both technically robust and ecologically responsible.


## API Overview

The DMPsee API is organised into three main parts:

- **Publisher**: Responsible for emitting DMP related events when changes occur in a DMP platform.

- **Subscriber**:   Applications or services that listen for and consume the published events.

- **Admin**: Provides management and monitoring capabilities for the event system.

### Event Data Flow

```

+-----------+              +-------------+               +-------------+
| Publisher | -- event --> |   DMPsee    | -- event -->  | Subscribers |
+-----------+              +-------------+               +-------------+
                                  ^
                                  |
                            +-----------+
                            |   Admin   |
                            +-----------+

```

### HTTP Protocol

The API uses [HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc7231) as the primary transport protocol.
For high-volume or low-latency scenarios, [HTTP/2](https://datatracker.ietf.org/doc/html/rfc7540) may be considered in future versions, but is not required for initial implementations and its adoption is not recommended.

All endpoints MUST be accessible over **HTTPS** to ensure data confidentiality and integrity.

### Slim HTTP headers

DMPsee uses JSON as data payload for API exchange (request and response).
For this reason, DMPsee explicitly avoids the use `Content-Type: application/json` HTTP headers.
Implementers MUST NOT include such header in their requests. This ensures a leaner and more efficient protocol design.

By removing such overhead, DMPsee not only simplifies implementation but also aligns with the The Environmentally Responsible Principle and broader sustainability goals in digital infrastructure.

### Authentication and Authorisation

API requests are authenticated using a custom **AC** HTTP header in the format:

```
AC: <api-id>:<api-key>
```

Allowed characters for both `api-id` and `api-key`:

- Alphanumeric (A–Z, a–z, 0–9)
- Dashes (-) and underscores (_)

Implementers MUST ensure:
- both `api-id` and `api-key` are unique and securely generated.
- adequate authorisation process

### API Request

The DMPsee API uses only the HTTP method `POST` and a single path `/post`.
Requests are sent as JSON array where every indexed position has

| Index | Description | Notes |
| ----- | ----------- | ------|
| 0     | Command | (Alphanumeric 3 chars) the command the Publisher, or Subscriber or Admin wants to perform |
| 1     | Data | (Mixed) The data part of the command |


### List of API request commands

| Command | User | Description | Example  | Notes |
| ------- | -----|----- | -------- | ----- |
| `eva`   | admin | Event Allow Publisher | `["eva",["test-event","pub-1"]]` | Grants publisher `pub-1` access to publish to `test-event`. |
| `evd`   | admin | Event Delete | `["evd","test-event"]` | Deletes the specified event. |
| `evi`   | admin | Event Disallow Publisher | `["evi",["test-event","pub-1"]]` | Revokes publishing rights from `pub-1` for `test-event`. |
| `evw`   | admin | Event Write | `["evw","test-event"]` | Creates an event with the given name. |
| `usd`   | admin | User Delete | `["usd","adm-2"]` | Deletes user `adm-2`. |
| `usw`   | admin | User Write | `["usw",["adm-2","key-adm-2","adm"]]` | Creates a user with ID `adm-2`, key `key-adm-2`, and role `adm`. |
| `evp`   | publisher |Event Publish | `["evp",["pup","123-abc"]]` | Publishes payload `123-abc` under event `pup`. |
| `evs`   | Subscriber | Event Subscribe | `["evs","test-event"]` | Subscribes to `test-event`.<br>Fails if event doesn’t exist (`["evs","non-existing"]`). |
| `evu`   | Subscriber | Event Unsubscribe | `["evu","test-event"]` | Unsubscribes from `test-event`.<br>Fails if event doesn’t exist (`["evu","non-existing"]`). |
| `urr`   | Subscriber | URL Read | `["urr"]` | Read the URL of the webhook. |
| `urw`   | Subscriber | URL Write | `["urw","my-url"]` | Write the URL of the webhook. |

[Table 1: List of API Request Commands]

### API response

DMPsee uses the most essential HTTP status.
For the same principle mentioned in "Slim HTTP headers", the typical response is composed by

- **Status-Line**	without The Reason Phrase (or Status Text). For example `HTTP/1.1 200`
- **Content-Length** For example assuming 4-digit length `Content-Length: 3072`

The only status codes used on DMPsee are show below.

| Code | Description |
| ---- | ----------- |
| 200  | (OK) The standard response for a successful operation. |
| 201  | (Created) The response when a command successfully creates a new resource on the server. For example, a publisher submit an event. |
| 400  | (Bad Request) This is for client-side errors. It would be used if the request JSON is malformed, a required field is missing or an invalid command is sent. |
| 401  | (Unauthorized) This is for authentication errors. It's the correct response if the provided X-Api-Id or X-Api-Key is missing or invalid. |
| 403  | (Forbidden) This is for authorization errors. It would be used if a client is authenticated but does not have permission for the requested action. |
| 500  | (Internal Server Error) The catch-all for unexpected issues on the server side. This would be used for any failure that is not the client's fault. |




### Events

Events are trasmitted by publisher to DMPsee, which will post back to the relative subscribers.
All events have a three-character **Event code (EC)** composed of a two-character **Event Element Prefix (EEP)** and a one-character **Event Action Code (EAC)**.
However, it is also possible to use DMPsee beyond the RDA DMP Common Standard.
For this reason, the first character of the Element prefixes, when numeric, is reserved for custom events.
Elements prefixes and Actions code are visible below:

| Element              | EEP |
| -------------------- | ------ |
| DMP                  | dm   |
| Contact              | co   |
| Contributor          | ct   |
| Cost                 | cs   |
| Project              | pr   |
| Funding              | fu   |
| Dataset              | ds   |
| Distribution         | di   |
| License              | li   |
| Host                 | ho   |
| Security and Privacy | sp   |
| Technical Resource   | te   |
| Metadata             | mt   |
| Reserved for custom events | [0-9]* |

| Action     | EAC |
| ---------- | ---- |
| Create     | c |
| Read (Get) | r |
| Update     | u |
| Delete     | d |

DMPsee uses the RDA DMP Common Standard for machine-actionable Data Management Plans (maDMPs) as its core data model.
At the time of writing, the current version is [Version 1.1](https://github.com/RDA-DMP-Common/RDA-DMP-Common-Standard/releases/tag/v1.1).
All event payloads transmitted by publishers and consumed by subscribers MUST conform to the structure defined in the official RDA-DMP Common Standard specification.
This ensures that the research planning updates exchanged between systems are universally understandable and actionable.

Examples of RDA DMP Common Standard event codes:

`dmu` = `dm` DMP `u` update
`dmc` = `dm` DMP `c` create
`cod` = `co` Contact `c` delete

When a publisher submit an event, the `data` part witt index position `1` inside the Event array.
This is the part that is stored and reported back to the subscribers.



---

## API for Publisher

The **API for Publisher** component is responsible for receiving events raised in a platform.
Publishers SHOULD ensure that updates are consistently broadcast to the event stream so that subscribers can act on them.
The Payload Structure for transmittin an event is

```
[
  0 => (required) API Request Command (ARC)
  [
    0 => "(required) Event code (EC)
    1 => "my-id", // (required) Publisher Internal Identifier (PII)
    2 => [optional] - the RDA DMP Common Standard element (RCSE) of the event.
                      A JSON object contains a maDMP object or a part of it,
                      validated against the RDA-DMP Common Standard schema
  ]
]
```

> **[!NOTE]** The Publisher Internal Identifier (PII) is an internal identifier specific for the publisher.
> The primary scope is to be a reference for the subscriber but also for the same publishers in case they need the value back from a subscriber.

While including the full DMP object in the index 2 of the data array, is the most convenient solution for the subscribers, DMPsee implementers MUST consider the environmental impact of data transfer over subscribers' convenience.
The primary goal MUST be to privilege the shortest content and avoid wasting bandwidth on unnecessary information.

Below an example of a API request

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: pub-1:key-pub-1" \
  -d '["evp", ["dmu", "my-id"]]'
```


## API for Subscriber

The **API for Subscriber** component allows downstream systems, such as storage providers, institutional repositories, or funding bodies, to manage their event streams.
Subscribers do not need to poll the DMP platform; instead, they use this API to register their interest and define where they want to receive asynchronous webhook callbacks.
Once they have obtained their API credentials, subscribers can send to DMPsee the commands as per Table 1.

**Available Commands for Subscribers:**

* **evs (Event Subscribe):** Registers interest in a specific 3-character event code.
* **evu (Event Unsubscribe):** Removes interest in a specific event code to stop receiving data.
* **urr (URL Read):** Retrieves the currently configured webhook callback URL for the subscriber.
* **urw (URL Write):** Sets or updates the destination URL where DMPsee will post events.

### Event Subscribe

Request structure:
```
[
  0 => (required) API Request Command (ARC)
  1 => (required) Event code (EC)
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d '["evs","dmu"]'
```

### Event Unsubscribe

Request structure:
```
[
  0 => (required) API Request Command (ARC)
  1 => (required) Event code (EC)
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d '["evu","dmu"]'
```



### URL Read
Request structure:
```
[
  0 => (required) API Request Command (ARC)
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d `["urr"]`
```


### URL Write

Request structure:
```
[
  0 => (required) API Request Command (ARC)
  1 => (required) Url to write
]
```

Example:


```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d `["urw","my-url"]`
```

## API for Admin

The **API for Admin** provides the governance layer for the DMPsee ecosystem.
It is used to manage the lifecycle of events and ensure that only authorized entities can publish to specific streams.
This component is critical for maintaining the security and integrity of the research data propagation network.

**Available Commands for Admins:**

* **eva (Event Allow Publisher):** Authorizes a specific Publisher ID to send events for a code.
* **evi (Event Disallow Publisher):** Revokes publishing permissions for a specific entity.
* **evw (Event Write):** Registers a new valid Event Code within the system.
* **evd (Event Delete):** Removes an Event Code and all associated permissions from the system.
* **usw (User Write):** Creates or updates user credentials and assigns roles (`pub`, `sub`, `adm`).
* **usd (User Delete):** Deactivates a user and revokes their API access.

### Event Allow Publisher

Request structure:

```
[
  0 => (required) API Request Command (ARC)
  [
    0 => (required) Event code (EC)
    1 => (required) Publisher ID
  ]
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d '["eva",["dmu","pub-1"]]'
```

### Event Disallow Publisher

Request structure:

```
[
  0 => (required) API Request Command (ARC)
  [
    0 => (required) Event code (EC)
    1 => (required) Publisher ID
  ]
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d '["evi",["dmu","pub-1"]]'
```

### Event Delete

Request structure:

```
[
  0 => (required) API Request Command (ARC)
  1 => (required) Event code (EC)
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d `["evd","dmu"]`
```

### Event Write

Request structure:

```
[
  0 => (required) API Request Command (ARC)
  1 => (required) Event code (EC)
]
```

Example:


```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d `["evw","dmu"]`
```

### User Delete

Request structure:

```
[
  0 => (required) API Request Command (ARC)
  1 => (required) Publisher ID
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d `["usd","pub-1"]`
```

### User Write


Request structure:

```
[
  0 => (required) API Request Command (ARC)
  [
    0 => (required) User ID
    1 => (required) API ID
    2 => (required) Role [pub = publisher, sub = subscriber, adm = admin]
  ]
]
```

Example:

```bash
curl -k -X POST https://demo-api.dmpsee.org/post \
  -H "AC: sub-1:key-sub-1" \
  -d `["usw",["pub-1","key-pub-1","pub"]]`
```
