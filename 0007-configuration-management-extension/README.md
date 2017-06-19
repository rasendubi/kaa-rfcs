---
name: Configuration Management Extension
shortname: 7/CMX
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
---

## Introduction

The Configuration Management Extension is a [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is intended to manage endpoint configuration distribution.

## Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## Requirements and constraints
### Problems and possible solutions

1. The server should know if a configuration has been applied by the endpoint. That is needed for server to not resend the same configuration over and over again and is also good for device management, so the user can verify that all endpoints have applied the required configuration.

   Possible solutions:
   - The server initiates a configuration push and the client sends a publish message in response to the new configuration.

     This means the server is initiating a request. That does not work well for HTTP, CoAP, and other protocols that only support client-initiated requests, so it is not best solution.

   - Introduce `/applied` resource, so the client sends an additional request once configuration is applied.

2. An endpoint might be interested in a part of the general configuration only.

   Possible solutions:
   - Allow subscribing to a part of configuration. (e.g., using [JSONPath](http://goessner.net/articles/JsonPath/), or JSON Pointer defined in [RFC 6901](https://tools.ietf.org/html/rfc6901).)

3. Difference between subsequent configurations might be small, so sending the whole configuration is inefficient.

   Possible solutions:
   - Send only updates. (e.g., using JSON Patch format, defined in [RFC 6902](https://tools.ietf.org/html/rfc6902).)

4. The configuration should be delivered to the client as soon as it is changes on the server.

   Possible solutions:
   - Server pushes new configuration to the specified topic without client's previous request.

     This does not require client to subscribe to the response topic or manifest its interest in any way, so it is minimum network usage.

     On the other hand, it requires tracking the current endpoint version, which is non-local to extension and is hard to implement reliably. Furthermore, it disables endpoint's ability to operate with different application versions simultaneously (e.g., sending data points with old data format over 2/DCX).

     Also, this is server-initiated request which is not supported by CoAP and is not recommended by 1/KP.

   - Client should subscribe to the defined resource path to receive configurations.

     This has additional overhead of the initial subscription request but plays nicely with CoAP.

     For MQTT, implicit subscription may be allowed, which nulifies subscription overhead argument.

5. In cases when the configuration data changes more than once while the endpoint is not connected to the server, after reconnections it SHOULD receive only the latest data.

## Use cases

### UC1: Configuration request
Configuration delivery by request. The endpoint should be able to receive latest configuration from CMX by request.

### UC2: Configuration push
Configuration delivery is initiated by the server. The endpoint should receive latest configuration when it connects to the server if this configuration wasn't applied yet.

## Design
The extension follows client-initiated request/response pattern described in 1/KP, which nicely maps to both MQTT and CoAP.

There are two resources managed by this extension.

- `/<endpoint_token>/config/json` is used for subscribing to and receiving endpoint configuration.
- `/<endpoint_token>/applied/json` is used for notifiying server of the currently active configuration.

### Configuration identifier
Configuration identifier is an opaque string, which uniquely identifies the configuration. The client SHOULD NOT assume the nature of identifiers.

<!-- TODO(ashmalko): define maximum configuration identifier length, which can help clients to store them. -->

### Absent configuration
If an endpoint does not have a configuration assigned, its configuration is `null`, and the corresponding configuration identifier is an empty string (`""`).

### Request/response
The extension uses request/response pattern described in [1/KP](/0001-kaa-protocol/README.md) with observe extension. For MQTT, it is achieved by publishing multiple responses to the response path; for CoAP, it is achieved with the `Observe` option defined in [RFC 7641](https://tools.ietf.org/html/rfc7641).

### Config resource
`/<endpoint_token>/config/json` resource is used for distributing the latest endpoint configuration.

The request payload MUST be a JSON-encoded object with the folowing [JSON Schema] (defined in [config-request.schema.json](./config-request.schema.json)).

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "7/CMX configuration pull request schema",

  "type": "object",
  "properties": {
    "id": {
      "type": "number",
      "multipleOf": 1.0,
      "description": "ID of the message used to match server response to the request."
    },
    "configId": {
      "type": "string",
      "description": "Identifier of the currently applied configuration. Optional.",
      "default": ""
    },
    "observe": {
      "type": "boolean"
      "description": "Identifies if the endpoint is interested in observing the configuration changes. Optional."
    }
  },
  "required": [ "id" ],
  "additionalProperties": false
}

```

A request is a JSON object with the following fields:
- `id` (optional) - an opaque string used to match responses with requests. If absent, the client is not interested in matching responses with the requests.
- `configId` (optional) - the identifier of the currently applied configuration. If not present, the absent configuration identifier (`""`) is assumed.
- `observe` (optional) - identifies if the client is interested in observing the configuration.

A response is a new configuration along with needed meta information. It is a JSON object with the following fields:
- `id` (optional) - identifier of the corresponding request. MUST be the same as in the corresponding request.
- `configId` (optional) - identifier of the `config`.
- `config` (optional) - an arbitrary JSON type, representing the latest configuration available for the endpoint.

`configId` and `config` MUST come in a pair. If `configId` and `config` are absent in the response, that means configuration hasn't changed.

#### Semantics

If `configId` field is missing, the server MUST respond with the latest configuration for the given endpoint, or no configuration if none was assigned.

If `configId` field is present, the server MUST send new configuration only if `configId` differs from the currently assigned configuration identifier for the endpoint.

If `observe` field is present and is `true`, the server MUST send new configuration to the endpoint whenever configuration changes.

If `observe` field is present and is `false`, the server MUST NOT send new configurations to the endpoint without a further explicit request.

The server MAY send configurations to the endpoint when no request were done before, or `observe` was not present in a request.

If the current config matches one specified by `configId` in the request, the server MUST return response with both `configId` and `config` absent.

If an endpoint successfully applies provided configuration, it SHOULD notify the server via `/applied` resource.

### Applied resource
`/<endpoint_token>/applied/json` resource is used by the endpoint to notify the server of successfully applied configuration.

The request is a JSON object with the following fields:
- `configId` (required) - id of applied configuration.

The response is empty.

## Open questions
### Merge resources
The request to `/config` resource already includes all fields for `/applied` requests, so it might be possible to merge them in a single resource.

### Error handling
Whaw errors are possible here? Should endpoint care?

### Configuration rejection
Should it be possible for the endpoint to actively reject a configuration?

Examples are:
- the endpoint is unable to process the configuration.
- the configuration is ill-formated, or pre-conditions are not met.

### Subscribing to a part of configuration
This is still not addressed in this document.

### Only send configuration difference
Minimizing network traffic by only sending a configuration difference is not addressed.

On the other hand, we're sending JSONs, so there are other more efficient ways to minimize traffic by changing the format (e.g., use BSON, CBOR, MessagePack).
