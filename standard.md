# DMPsee: Environmentally Responsible Event Propagation Standard for Data Management Plans

**Authors:**
Andrea Davanzo [0009-0000-5170-1737](https://orcid.org/0009-0000-5170-1737), University of Edinburgh

Last revision: 2025-09-04

> [!NOTE]
> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119)

## Introduction

DMP Sent Events (DMPsee) is an **open standard** designed to propagate updates
from DMP platforms to other systems.
These events can then be consumed by any interested subscriber.

This publish/subscribe model enables a loosely coupled and scalable approach
to keeping dependent systems informed of DMP changes across platforms.

As an open standard, DMPsee promotes interoperability, avoids vendor lock-in,
and ensures that diverse systems can seamlessly exchange DMP updates.

### The Environmentally Responsible Principle

One of the guiding principles of the DMPsee standard is to minimize unnecessary
data transfer, thereby reducing environmental impact.
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

### Protocol

The API uses [HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc7231) as the primary transport protocol. For high-volume or low-latency scenarios, [HTTP/2](https://datatracker.ietf.org/doc/html/rfc7540) may be considered in future versions, but is not required for initial implementations.

All endpoints SHOULD be accessible over **HTTPS** to ensure data confidentiality and integrity.

### Slim HTTP headers

DMPsee uses JSON as data payload for API exchange.
For this reason, DMPsee explicitly avoids the use `Content-Type: application/json` HTTP headers.
Implementers MUST NOT include such header in their requests. This ensures a leaner and more efficient protocol design.

By removing such overhead, DMPsee not only simplifies implementation but also aligns with the The Environmentally Responsible Principle and broader sustainability goals in digital infrastructure.

### Authentication

API requests are authenticated using two distinct methods to balance security and ease of use for different client types.

#### Publisher and Subscriber Authentication
Publisher and Subscriber clients authenticate using a simple, static API key model.
API requests from these clients must include the following headers:

- **X-Api-Id**: A unique ID that identifies the client (e.g., a specific publisher or subscriber application).

- **X-Api-Key**: The long-lived, secret key associated with the X-Api-Id. This key is the primary credential for a client to publish or subscribe to events.

#### Admin Authentication
Admin clients, which are typically used by human operators to manage the system, use a more secure, token-based authentication method. This approach leverages short-lived Bearer tokens to protect sensitive administrative actions.
Requests to the Admin API must include a standard Authorization header with a valid token:

- Authorization: Bearer <token>: The access token provided after a successful login to the Admin service. These tokens are designed to be temporary and are associated with a specific user session, ensuring a high level of security and auditability for all administrative actions.

### API response

DMPsee uses the most essential HTTP status. Here’s how each one maps to a likely scenario in DMPsee:

- **200 OK**: The standard response for a successful operation.

- **201 Created**: The response when a command successfully creates a new resource on the server. For example, when a subscriber sets a new URL or publisher submit an event.

- **400 Bad Request**: This is for client-side errors. It would be used if the request JSON is malformed, a required field is missing (e.g., cmd), or an invalid command is sent.

- **401 Unauthorized**: This is for authentication errors. It's the correct response if the provided X-Api-Id or X-Api-Key is missing or invalid.

- **403 Forbidden**: This is for authorization errors. It would be used if a client is authenticated but does not have permission for the requested action.

- **404 Not Found**: The standard response when the requested resource does not exist. While the API uses a single endpoint, this could be used if a command references an id that cannot be found in the system.

- **500 Internal Server Error**: The catch-all for unexpected issues on the server side. This would be used for any failure that is not the client's fault.


### RDA DMP Common Standard

The DMPsee uses the RDA DMP Common Standard for machine-actionable Data Management Plans (maDMPs) as its core data model.
At the time of writing the current version is [Version 1.1](https://github.com/RDA-DMP-Common/RDA-DMP-Common-Standard/releases/tag/v1.1).
All event payloads transmitted by publishers and consumed by subscribers MUST conform to the structure defined in the official RDA-DMP Common Standard specification.
This ensures that the research planning updates exchanged between systems are universally understandable and actionable.


---

## Publisher API

The **Publisher** component of the DMPsee API is responsible for receiving events raised in a DMP platform.
Publishers SHOULD ensure that updates are consistently broadcast to the event stream so that subscribers can act on them.

### Submit an Event

#### Request

**Method:** POST

**Path:** `/publisher/submit`

**Headers:**
`X-api-id: <publisher-id>`
`X-api-key: <publisher-key>`

**Payload Structure**:
- `cmd` (string)  [required] - the command to execute. For publishing events, use `event-publish`.
- `data` (object) [required] - contains the actual event data:
  - `event` (string) [required] - the type of event i.e. `"plan-updated"`, `"plan-created"`, `"plan-deleted"`).
  - `id` (string) [required] - internal identifier of the plan.
  - `data` (object) [optional] - the details of the event. A JSON object contains a maDMP object or a part of it, validated against the RDA-DMP Common Standard schema.

> **[!NOTE]** The field `id` is an internal identifier specific for the DMP platform.
> The primary scope is to be a reference for the subscriber but also for the same publishers in case they need the value back from a subscriber.

While including the full DMP object in the `data` is the most convinient solution for the subscribers, DMPsee implementers MUST consider the environmental impact of data transfer over subsscribers' convinience. The primary goal must be to privilege the shortest content and avoid wasting bandwidth on unnecessary information. The data element MUST be used only when essential for the subscriber to maintain synchronization.


```json
{
  "cmd": "event-publish",
  "payload": {
    "event": "plan-updated",
    "id": "123",
    "data": {
      "dmp": {
        "dmp_id": {
          "identifier": "https://doi.org/xyz",
          "type": "doi"
        },
        "title": "My Updated Research Plan",
        "description": "...",
        "contact": {
          ...
        },
        "dataset": [
          ...
        ]
      }
    }
  }
}
```


**Example Request:**

```bash
curl -k -X POST https://api.dmpsee.org/publisher/submit \
  -H "X-api-id: pub-id-1" \
  -H "X-api-key: pub-key-1" \
  -d '{
    "cmd": "event-publish",
    "data": {
      "event": "plan-updated",
      "id": "123-abc"
    }
  }'
```


#### Response

##### Success

```
HTTP Status code: 201
Body:
{
  "event_id": "id of the event assigned by DMPsee"
}
```

##### Failure:
```
HTTP Status codes: 40x or 500
Body:
{
  "message": "Failure message"
}
```

---


## API Subscriber

The **Subscriber** component of the DMPsee API is responsible for managing the subscribers' request.
One they have obtained their API crendentials, subscribers can send to DMPsee the following commands:

- **url**: create/update the URL where to receive the events
- **event-subscribe**: subscribe to one or more events
- **event-unsubscribe**: unsubscribe to one or more events
- **event-list**: list of all subscribed events


All the commands will use:

**Method:** POST

**Path:** `/subscriber/submit`

**Headers:**
`X-api-id: <publisher-id>`
`X-api-key: <publisher-key>`

### API Response

For all the commands sent to the API the response will be the same in case of Success or Failure.

#### Success

There is no body on success response as the http statu code is sufficient

```
HTTP Status codes: 200
Body: None
```

#### Failure:

```
HTTP Status codes: 40x or 500
Body:
{
  "message": "Failure message"
}

### Command "url"

Set/Update the Subscriber URL

#### Request

```bash

curl -k -X POST https://api.dmpsee.org/subscriber/submit \
  -H "X-api-id: sub-id-1" \
  -H "X-api-key: sub-key-1" \
  -d '{
    "cmd": "url",
    "url": "https://mysubscriber.example.com/dmp-events"
  }'
```

### Command "sub"

Registers the subscriber's interest in one or more event types.

#### Request

```bash

curl -k -X POST https://api.dmpsee.org/subscriber/submit \
  -H "X-api-id: sub-id-1" \
  -H "X-api-key: sub-key-1" \
  -d '{
    "cmd": "sub",
    "events": ["plan-updated", "plan-created"]
  }'

```

### Command "unsub"

This command removes the subscriber's registration for a specific event type.

#### Request

```bash

curl -k -X POST https://api.dmpsee.org/subscriber/submit \
  -H "X-api-id: sub-id-1" \
  -H "X-api-key: sub-key-1" \
  -d '{
    "cmd": "unsub",
    "events": ["plan-created"]
  }'

```

