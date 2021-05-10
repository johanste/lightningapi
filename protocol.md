# Notification channel protocols (the last mile)

The Microsoft REST API guidelines outlines how to expose push notifications via webhooks. However, it does not provide a mechanism by which a service can push notifications to clients that are unable to receive HTTP requests/have an open port to receive incoming connections.

### Requirements

- Only require a single (possibly multiplexed) connection between the client and the service.
- Non-breaking (polling must still be possible)
- No race conditions where notifications are lost due to the client not having set up a notification channel yet.
- Networking stack support for tier 1 client languages (i.e. .NET, TypeScript/javascript, Java and Python)
- No information leak (receiving notifications for resources the client would not otherwise be allowed to read)

### Non goals

- Replace existing polling. The push notification protocol is additive to existing service API capabilities.
- Get asynchronous notifications for state changes not initiated by the client (e.g. continously listen for power state changes for a VM)

## Protocol

The client initiates a session by opening a [websocket](https://tools.ietf.org/html/rfc6455) connection to the service, providing a unique `sessionId` and `subscriptionId`.

> Note: The `subscriptionId` is the Azure Subscription Id in the case of Azure Resource Manager, but could be a different identifier for other services - e.g. the resource id for a cognitive service account.

The client includes the same `sessionId` along with a new, unique `notificationId` value in a header for each request that the client wants to receive notifications for.

> Note: While the `sessionId` could be server generated, it is preferable for the client to generate - or at least be able to provide - the `sessionId` in order to allow for multiple clients to connect to the same notification channel or addition to allowing the client to re-connect to the same channel in case of lost connections. Similarly, allowing the client to generate the `notificationId` avoids any race conditions that may otherwise occur if the operation completed before the response for the initial request reached the client. 

The service acknowledges the request to send push notifications by sending a `notification-session-result` header back to the client with the value `'accepted'`. The service MUST only acknowledge push notifications for asynchronous/long running operations. For synchronous operations, the service MUST NOT include the `notification-session-result` header in the response.

The client MUST assume that the server did not honor the request unless the server expclicitly stated that it had understood the request by responding with a `notification-session-result: accepted` header. 

The server notifies the client that a long running operation has completed by sending an `OperationCompleted` message containing the corresponding `notificationId` value back on the notification channel. 

### Service behavior

A push-notificaton enabled service MUST support previous flavors of HTTP polling for completion of long running operations. It MUST NOT assume that a client will receive the push notification even if it signalled support for the push notification pattern. The push notification is an optional optimization for clients.

### Notification Service behavior

The notification service MAY redirect a connection request to another node by responding with a standard HTTP `307 - Temporary Redirect` response with a `Location` header with a new url. This allows for load balancing or sharding of traffic between nodes.

The notification service MUST NOT redirect clients more than 10 times since clients are generally expected to not allow infinite redirect loops.

The notification service MUST support multiple listeners for the same session.

The notification service handler MAY close the connection after a no notification completed messages have been sent for a service defined amount of time. The idle timeout SHOULD be greater than 10 minutes. 

For `WebSocket` clients, the notification service handler MAY also close the connection if it has not received a [Ping frame](https://tools.ietf.org/html/rfc6455#section-5.5.2) in over 1 minute.

The notification service MAY choose temporarily cache notifications for unknown `sessionId`s, but it is not under any obligation to do so. 

### Client behavior

As the notification channel does not guarantee exactly-once delivery of messages, the client is required to handle both lost and duplicated notifications.

A client MUST generate a new, within the session, unique `notificationId` for each request it wants to subscribe to notifications for. The `notificationId` MUST NOT contain sensitive or secure information.

> Note (non-normative): The easiest implementation is to generate a new GUID for each request, but a client MAY also use a monotonically increasing sequence number or timestamps as long as they are unique.

A client MUST ignore to it unknown `notificationId`s. A client MUST ignore a `notificationId` that is has previously received/handled. 

The client MUST follow HTTP `307 - Temporary Redirect` responses when trying to connect. It SHOULD limit the number of followed redirects to a maximum of 10

While the session is active over a WebSocket connection (there are outstanding notifications), the client MUST send a [`Ping`](https://tools.ietf.org/html/rfc6455#section-5.5.2) every 20s in order to keep the notification channel open.

The client MUST close the notification channel once its session has completed.

If the notification channel connection is lost, the client MUST assume that all notifications associated with the session are lost and fall back to normal polling for operation completion The client MAY reconnect to the notification service using the same `sessionId` in order to short-circuit the HTTP polling procedure.

## Authentication and Authorization

TODO

### Message format

Messages uses the [websockets protocol binding](https://github.com/cloudevents/spec/blob/v1.0.1/websockets-protocol-binding.md) of [CloudEvents v1.0](https://cloudevents.io). 

#### ConnectToSessionMessage

```http
GET /notifications?subscriptionId=...&sessionId=...&api-version=1971-11-01
```

#### OperationCompletedMessage

The `com.azure.operationcompletion` message type includes the following .

```json
{
    "specversion": "1.0",
    "source": "com.management.azure",
    "id": <operationUrl>,
    "type": "com.azure.operationcompletion",
    "subject": <sessionNotificationId>,
    "datacontenttype": "application/json",
    "data": {
        "sessionId": <sessionId>
    }
}
```

## Versioning considerations

## Current technologies

- EventGrid
- EventHubs, Storage queues, Service Bus
- Azure relay
- SignalR
