# <a name="documentation"></a>Saxo OpenApi documentation

This document describes how an application can subscribe to the Saxo Event Notification Service (ENS).

## Table of contents

[Get realtime data using the Saxo API](#realtime)\
[Step 1: Connect to the feed](#realtime1)\
[Step 2: Handle connection events](#realtime2)\
[Step 3: Subscribe to updates](#realtime3)\
[Step 4: Handling events](#realtime4)\
[Step 5: Extend the subscription before the token expires](#realtime5)\
[Step 6: Description of the data](#realtime6)\

## <a name="realtime"></a>Get realtime data using the Saxo API

This document describes the realtime feed available for customers. Assumption is that a token is already available.

Native web sockets are used. Examples in JavaScript.

> The WebSocket API is an advanced technology that makes it possible to open a two-way interactive communication session between the user's browser and a server. With this API, you can send messages to a server and receive event-driven responses without having to poll the server for a reply.

More info: <https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_client_applications>.

Prerequisites:

- A token retrieved using the OAuth2 Authentication flow, as described in the Saxo documentation: <>.

For this example we use the simulation (sandbox) environment, with predefined test users and passwords.

### <a name="realtime1"></a>Step 1: Connect to the feed

The WebSocket API can be used in JavaScript by any modern browser.

The following code creates and starts a connection:

```javascript
    var contextId = encodeURIComponent("MyApp" + Date.now());
    var streamerUrl = "wss://gateway.saxobank.com/sim/openapi/streamingws/connect?authorization=" + encodeURIComponent("BEARER " + accessToken) + "&contextId=" + contextId;
    var connection = new WebSocket(streamerUrl);
    console.log("Connection created. Status: " + connection.readyState);
```

**accessToken** – The Bearer token

The contextId uniquely defines the connection. A client application might have multiple connections.

More info about the setup at Saxo: <https://www.developer.saxo/openapi/learn/plain-websocket-streaming>.

### <a name="realtime2"></a>Step 2: Handle connection events

The user might stop the connection. Or something can go wrong with the server. Then the application might do a reconnect, or just show this to the user.

The following code configures the events:

```javascript
    connection.onopen = function () {
        console.log("Streaming connected");
    };
    connection.onclose = function () {
        console.log("Streaming disconnected");
    };
    connection.onerror = function (evt) {
        console.error(evt);
    };
    connection.onmessage = function (event) {
        console.log("Streaming message received: " + event.data);
    };
```

### <a name="realtime3"></a>Step 3: Subscribe to updates

In order to subscribe to events, you need to create a subscription with a POST request to the API. This is an example to subscribe to order events:

```javascript
    var data = {
        "ContextId": contextId,
        "ReferenceId": "orders",
        "Arguments": {
            "ClientKey": clientKey,
            "AccountKey": accountKey,
            "Activities": [
                "AccountFundings",
                "Orders",
                "Positions"
            ],
            "FieldGroups": [
                "DisplayAndFormat",
                "ExchangeInfo"
            ]
        }
    };

    $.ajax({
        "dataType": "json",
        "contentType": "application/json; charset=utf-8",
        "type": "POST",
        "url": "https://gateway.saxobank.com/sim/openapi/ens/v1/activities/subscriptions",
        "data": JSON.stringify(data),
        "headers": {
            "Access-Control-Allow-Origin": "*",
            "Content-Type": "application/json",
            "Authorization": "Bearer " + accessToken
        }
    });
```

**contextId** – The identifyer of the connection (so the server can determine the target connection)\
**referenceId** – This is a reference for the updates; every event has a referenceId, to identify them - system events are prefixed by an underscore \
**clientKey** – The key identifying the customer\
**accountKey** – The key identifying the account\
**accessToken** – The Bearer token

That’s it. The application is now ready to receive order update messages.

More info about this endpoint: <https://www.developer.saxo/openapi/referencedocs/service?apiVersion=v1&serviceGroup=ens&service=client%20activities>. There you'll read about setting up a refresh rate and how to get (older) snapshot data.

### <a name="realtime4"></a>Step 4: Handling events

#### Receiving order events

The order events are published when there is a change in portfolio positions, orders or cash balances.

As described above, events are received in the onmessage handler:

```javascript
    function getTimeStamp(event) {
        var time = startTime.valueOf() + event.timeStamp;
        var timeStamp = new Date();
        timeStamp.setTime(time);
        return timeStamp;
    }

    function parseStreamingMessage(data, timeStamp) {
        try {
            var message = new DataView(data);
            var bytes = new Uint8Array(data);
            var messageId = message.getInt8();
            var refBeginIndex = 10;
            var refIdLength = message.getInt8(refBeginIndex);
            var refId = String.fromCharCode.apply(String, bytes.slice(refBeginIndex + 1, refBeginIndex + 1 + refIdLength));
            var payloadBeginIndex = refBeginIndex + 1 + refIdLength;
            var payloadLength = message.getUint32(payloadBeginIndex + 1, true);
            var segmentEnd = payloadBeginIndex + 5 + payloadLength;
            var payload = String.fromCharCode.apply(String, bytes.slice(payloadBeginIndex + 5, segmentEnd));
            var block = JSON.parse(payload);
            console.log("Message " + messageId + " parsed with referenceId " + refId + " and payload: " + payload");
            block.ReferenceId = refId;
            block.MessageID = messageId;
            block.Timestamp = timeStamp;
            switch (refId) {
            case "orders":
                console.log("Order event to be processed:");
                console.log(block[0]);
                break;
            default:
                console.log("No processing implemented for message with reference " + refId);
            }
            return {
                "segmentEnd": segmentEnd,
                "messages": block
            };
        } catch (error) {
            console.error("Parse message failed: " + error);
        }
    }

    connection.onmessage = function (event) {
        var reader = new FileReader();
        var timeStamp = getTimeStamp(event);
        console.log("Streaming message received at: " + timeStamp);
        reader.readAsArrayBuffer(event.data);
        reader.onloadend = function () {
            var beginAt = 0;
            var data = reader.result;
            var parsedMessage;
            do {
                parsedMessage = parseStreamingMessage(data, timeStamp);
                beginAt = parsedMessage.segmentEnd;
                data = data.slice(beginAt);
            } while (data.byteLength > 0);
            console.log(parsedMessage);
        };
    };
```

### <a name="realtime5"></a>Step 5: Extend the subscription before the token expires

The realtime feed will stop after the token has been expired. When the application has refreshed the token, there is a need to extend the subscription.

Extend the subscription using this code:

```javascript
    $.ajax({
        "type": "PUT",
        "url": "https://gateway.saxobank.com/sim/openapi/streamingws/authorize?contextid=" + contextId,
        "timeout": 30 * 1000,  // Timeout after 30 seconds.
        "headers": {
            "Access-Control-Allow-Origin": "*",
            "Content-Type": "application/json",
            "Authorization": "Bearer " + accessToken
        }
    });
```

**contextId** – The identifyer of the connection (so the server can determine the target connection)\
**accessToken** – The Bearer token

### <a name="realtime6"></a>Step 6: Description of the data

#### Order-object

An order object is structured as following:

```javascript
{
    AccountId: "123",
    ActivityTime: "2020-01-27T14:57:03.917000Z",
    ActivityType: "Orders",
    Amount: 100,
    ​AssetType: "Stock",
    ​BuySell: "Buy",
    ​ClientId: "456",
    ​CorrelationKey: "b52fffbf-be1b-445a-a253-35360df2c31f",
    ​DisplayAndFormat: {
        Currency: "EUR",
        Decimals: 2,
        Description: "Danone",
        OrderDecimals: 2,
​​        Symbol: "BN:xpar"
    },
    ​Duration: {
        DurationType: "DayOrder"
    },
    ​ExchangeInfo: {
        ExchangeId: "PAR"
    },
    ​HandledBy: "789",
    ​OrderId: "1234",
    ​OrderRelation: "StandAlone",
    ​OrderType: "Limit",
    ​Price: 60,
    ​SequenceId: "5678",
    ​Status: "Placed",
    ​SubStatus: "Confirmed",
    ​Symbol: "BN:xpar",
    ​Uic: 112809
}
```
