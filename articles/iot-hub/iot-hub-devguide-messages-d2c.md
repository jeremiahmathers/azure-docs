---
title: Understand Azure IoT Hub message routing | Microsoft Docs
description: Developer guide - how to use message routing to send device-to-cloud messages. Includes information about sending both telemetry and non-telemetry data.
author: kgremban
manager: mehmet.kucukgoz
ms.service: iot-hub
services: iot-hub
ms.topic: conceptual
ms.date: 05/14/2021
ms.author: kgremban
ms.custom: ['Role: Cloud Development', devx-track-csharp]
---

# Use IoT Hub message routing to send device-to-cloud messages to different endpoints

[!INCLUDE [iot-hub-basic](../../includes/iot-hub-basic-partial.md)]

Message routing enables you to send messages from your devices to cloud services in an automated, scalable, and reliable manner. Message routing can be used for: 

* **Sending device telemetry messages as well as events** namely, device lifecycle events, device twin change events, digital twin change events, and device connection state events to the built-in-endpoint and custom endpoints. Learn about [routing endpoints](#routing-endpoints). To learn more about the events sent from IoT Plug and Play devices, see [Understand IoT Plug and Play digital twins](../iot-develop/concepts-digital-twin.md).

* **Filtering data before routing it to various endpoints** by applying rich queries. Message routing allows you to query on the message properties and message body as well as device twin tags and device twin properties. Learn more about using [queries in message routing](iot-hub-devguide-routing-query-syntax.md).

IoT Hub needs write access to these service endpoints for message routing to work. If you configure your endpoints through the Azure portal, the necessary permissions are added for you. Make sure you configure your services to support the expected throughput. For example, if you are using Event Hubs as a custom endpoint, you must configure the **throughput units** for that event hub so it can handle the ingress of events you plan to send via IoT Hub message routing. Similarly, when using a Service Bus Queue as an endpoint, you must configure the **maximum size** to ensure the queue can hold all the data ingressed, until it is egressed by consumers. When you first configure your IoT solution, you may need to monitor your other endpoints and make any necessary adjustments for the actual load.

The IoT Hub defines a [common format](iot-hub-devguide-messages-construct.md) for all device-to-cloud messaging for interoperability across protocols. If a message matches multiple routes that point to the same endpoint, IoT Hub delivers message to that endpoint only once. Therefore, you don't need to configure deduplication on your Service Bus queue or topic. Use this tutorial to learn how to [configure message routing](tutorial-routing.md).

## Routing endpoints

An IoT hub has a default built-in-endpoint (**messages/events**) that is compatible with Event Hubs. You can create [custom endpoints](iot-hub-devguide-endpoints.md#custom-endpoints) to route messages to by linking other services in your subscription to the IoT Hub. 

Each message is routed to all endpoints whose routing queries it matches. In other words, a message can be routed to multiple endpoints.

If your custom endpoint has firewall configurations, consider using the [Microsoft trusted first party exception.](./virtual-network-support.md#egress-connectivity-from-iot-hub-to-other-azure-resources)

IoT Hub currently supports the following endpoints:

 - Built-in endpoint
 - Azure Storage
 - Service Bus Queues and Service Bus Topics
 - Event Hubs

## Built-in endpoint as a routing endpoint

You can use standard [Event Hubs integration and SDKs](iot-hub-devguide-messages-read-builtin.md) to receive device-to-cloud messages from the built-in endpoint (**messages/events**). Once a Route is created, data stops flowing to the built-in-endpoint unless a Route is created to that endpoint. Even if no routes are created, a fallback route must be enabled to route messages to the built-in endpoint. The fallback is enabled by default if you create your hub using the portal or the CLI.

## Azure Storage as a routing endpoint

There are two storage services IoT Hub can route messages to: [Azure Blob Storage](../storage/blobs/storage-blobs-introduction.md) and [Azure Data Lake Storage Gen2](../storage/blobs/data-lake-storage-introduction.md) (ADLS Gen2) accounts. Azure Data Lake Storage accounts are [hierarchical namespace](../storage/blobs/data-lake-storage-namespace.md)-enabled storage accounts built on top of blob storage. Both of these use blobs for their storage.

IoT Hub supports writing data to Azure Storage in the [Apache Avro](https://avro.apache.org/) format and the JSON format. The default is AVRO. When using JSON encoding, you must set the contentType to **application/json** and contentEncoding to **UTF-8** in the message [system properties](iot-hub-devguide-routing-query-syntax.md#system-properties). Both of these values are case-insensitive. If the content encoding is not set, then IoT Hub will write the messages in base 64 encoded format.

The encoding format can be only set when the blob storage endpoint is configured; it can't be edited for an existing endpoint. To switch encoding formats for an existing endpoint, you'll need to delete the endpoint and re-create it with the format you want. One helpful strategy might be to create a new custom endpoint with your desired encoding format and add a parallel route to that endpoint. In this way, you can verify your data before deleting the existing endpoint.

You can select the encoding format using the IoT Hub Create or Update REST API, specifically the [RoutingStorageContainerProperties](/rest/api/iothub/iothubresource/createorupdate#routingstoragecontainerproperties), the [Azure portal](https://portal.azure.com), [Azure CLI](/cli/azure/iot/hub/routing-endpoint), or [Azure PowerShell](/powershell/module/az.iothub/add-aziothubroutingendpoint). The following image shows how to select the encoding format in the Azure portal.

![Blob storage endpoint encoding](./media/iot-hub-devguide-messages-d2c/blobencoding.png)

IoT Hub batches messages and writes data to storage whenever the batch reaches a certain size or a certain amount of time has elapsed. IoT Hub defaults to the following file naming convention:

```
{iothub}/{partition}/{YYYY}/{MM}/{DD}/{HH}/{mm}
```

You may use any file naming convention, however you must use all listed tokens. IoT Hub will write to an empty blob if there is no data to write.

We recommend listing the blobs or files and then iterating over them, to ensure all blobs or files are read without making any assumptions of partition. The partition range could potentially change during a [Microsoft-initiated failover](iot-hub-ha-dr.md#microsoft-initiated-failover) or IoT Hub [manual failover](iot-hub-ha-dr.md#manual-failover). You can use the [List Blobs API](/rest/api/storageservices/list-blobs) to enumerate the list of blobs or [List ADLS Gen2 API](/rest/api/storageservices/datalakestoragegen2/path) for the list of files. See the following sample as guidance.

```csharp
public void ListBlobsInContainer(string containerName, string iothub)
{
    var storageAccount = CloudStorageAccount.Parse(this.blobConnectionString);
    var cloudBlobContainer = storageAccount.CreateCloudBlobClient().GetContainerReference(containerName);
    if (cloudBlobContainer.Exists())
    {
        var results = cloudBlobContainer.ListBlobs(prefix: $"{iothub}/");
        foreach (IListBlobItem item in results)
        {
            Console.WriteLine(item.Uri);
        }
    }
}
```

To create an Azure Data Lake Gen2-compatible storage account, create a new V2 storage account and select *enabled* on the *Hierarchical namespace* field on the **Advanced** tab as shown in the following image:

![Select Azure Date Lake Gen2 storage](./media/iot-hub-devguide-messages-d2c/selectadls2storage.png)

## Service Bus Queues and Service Bus Topics as a routing endpoint

Service Bus queues and topics used as IoT Hub endpoints must not have **Sessions** or **Duplicate Detection** enabled. If either of those options are enabled, the endpoint appears as **Unreachable** in the Azure portal.

## Event Hubs as a routing endpoint

Apart from the built-in-Event Hubs compatible endpoint, you can also route data to custom endpoints of type Event Hubs. 

## Reading data that has been routed

You can configure a route by following this [tutorial](tutorial-routing.md).

Use the following tutorials to learn how to read messages from an endpoint.

* Reading from [Built-in-endpoint](../iot-develop/quickstart-send-telemetry-iot-hub.md?pivots=programming-language-nodejs)

* Reading from [Blob storage](../storage/blobs/storage-blob-event-quickstart.md)

* Reading from [Event Hubs](../event-hubs/event-hubs-dotnet-standard-getstarted-send.md)

* Reading from [Service Bus Queues](../service-bus-messaging/service-bus-dotnet-get-started-with-queues.md)

* Read from [Service Bus Topics](../service-bus-messaging/service-bus-dotnet-how-to-use-topics-subscriptions.md)


## Fallback route

The fallback route sends all the messages that don't satisfy query conditions on any of the existing routes to the built-in-Event Hubs (**messages/events**), that is compatible with [Event Hubs](../event-hubs/index.yml). If message routing is turned on, you can enable the fallback route capability. Once a route is created, data stops flowing to the built-in-endpoint, unless a route is created to that endpoint. If there are no routes to the built-in-endpoint and a fallback route is enabled, only messages that don't match any query conditions on routes will be sent to the built-in-endpoint. Also, if all existing routes are deleted, fallback route must be enabled to receive all data at the built-in-endpoint. 

You can enable/disable the fallback route in the Azure portal->Message Routing blade. You can also use Azure Resource Manager for [FallbackRouteProperties](/rest/api/iothub/iothubresource/createorupdate#fallbackrouteproperties) to use a custom endpoint for fallback route.

## Non-telemetry events

In addition to device telemetry, message routing also enables sending device twin change events, device lifecycle events, digital twin change events, and device connection state events. For example, if a route is created with data source set to **device twin change events**, IoT Hub sends messages to the endpoint that contain the change in the device twin. Similarly, if a route is created with data source set to **device lifecycle events**, IoT Hub sends a message indicating whether the device or module was deleted or created. For more information about device lifecycle events, see [Device and module lifecycle notifications](./iot-hub-devguide-identity-registry.md#device-and-module-lifecycle-notifications). When using [Azure IoT Plug and Play](../iot-develop/overview-iot-plug-and-play.md), a developer can create routes with data source set to **digital twin change events** and IoT Hub sends messages whenever a digital twin property is set or changed, a digital twin is replaced, or when a change event happens for the underlying device twin. Finally, if a route is created with data source set to **device connection state events**, IoT Hub sends a message indicating whether the device was connected or disconnected.


[IoT Hub also integrates with Azure Event Grid](iot-hub-event-grid.md) to publish device events to support real-time integrations and automation of workflows based on these events. See key [differences between message routing and Event Grid](iot-hub-event-grid-routing-comparison.md) to learn which works best for your scenario.

## Limitations for device connection state events

Device connection state events are available for devices connecting using either the MQTT or AMQP protocol, or using either of these protocols over WebSockets. Requests made only with HTTPS won't trigger device connection state notifications. For IoT Hub to start sending device connection state events, after opening a connection a device must call either the *cloud-to-device receive message* operation or the *device-to-cloud send telemetry* operation. Outside of the Azure IoT SDKs, in MQTT these operations equate to SUBSCRIBE or PUBLISH operations on the appropriate messaging [topics](iot-hub-mqtt-support.md). Over AMQP these equate to attaching or transferring a message on the [appropriate link paths](iot-hub-amqp-support.md).

IoT Hub does not report each individual device connect and disconnect, but rather publishes the current connection state taken at a periodic 60 second snapshot. Receiving either the same connection state event with different sequence numbers or different connection state events both mean that there was a change in the device connection state during the 60 second window.

## Testing routes

When you create a new route or edit an existing route, you should test the route query with a sample message. You can test individual routes or test all routes at once and no messages are routed to the endpoints during the test. Azure portal, Azure Resource Manager, Azure PowerShell, and Azure CLI can be used for testing. Outcomes help identify whether the sample message matched the query, message did not match the query, or test couldn't run because the sample message or query syntax are incorrect. To learn more, see [Test Route](/rest/api/iothub/iothubresource/testroute) and [Test all routes](/rest/api/iothub/iothubresource/testallroutes).

## Latency

When you route device-to-cloud telemetry messages using built-in endpoints, there is a slight increase in the end-to-end latency after the creation of the first route.

In most cases, the average increase in latency is less than 500 ms. However, the latency you experience can vary and can be higher depending on the tier of your IoT hub and your solution architecture. You can monitor the latency using **Routing: message latency for messages/events** or **d2c.endpoints.latency.builtIn.events** IoT Hub metric. Creating or deleting any route after the first one does not impact the end-to-end latency.

## Monitoring and troubleshooting

IoT Hub provides several metrics related to routing and endpoints to give you an overview of the health of your hub and messages sent. For a list of all of the IoT Hub metrics broken out by functional category, see [Metrics in the Monitoring data reference](monitor-iot-hub-reference.md#metrics). You can track errors that occur during evaluation of a routing query and endpoint health as perceived by IoT Hub with the [**routes** category in IoT Hub resource logs](monitor-iot-hub-reference.md#routes). To learn more about using metrics and resource logs with IoT Hub, see [Monitor IoT Hub](monitor-iot-hub.md).

You can use the REST API [Get Endpoint Health](/rest/api/iothub/iothubresource/getendpointhealth#iothubresource_getendpointhealth) to get [health status](iot-hub-devguide-endpoints.md#custom-endpoints) of the endpoints.

Use the [troubleshooting guide for routing](troubleshoot-message-routing.md) for more details and support for troubleshooting routing.

## Next steps

* To learn how to create Message Routes, see [Process IoT Hub device-to-cloud messages using routes](tutorial-routing.md).

* [How to send device-to-cloud messages](../iot-develop/quickstart-send-telemetry-iot-hub.md?pivots=programming-language-nodejs)

* For information about the SDKs you can use to send device-to-cloud messages, see [Azure IoT SDKs](iot-hub-devguide-sdks.md).
