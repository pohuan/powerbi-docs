---
title: Asynchronous refresh with the Power BI REST API 
description: Describes asynchronous refresh by using the Power BI REST API
author: minewiskan
ms.author: owend
ms.service: powerbi
ms.subservice: pbi-data-sources
ms.topic: conceptual
ms.date: 01/26/2022
ms.custom: contperf-fy21q4
LocalizationGroup: 
---
# Asynchronous refresh with the Power BI REST API (Preview)

By using any programming language that supports REST calls, you can perform asynchronous data-refresh operations on your Power BI datasets.

Dataset refresh operations can take some time depending on a number of factors including data volume, level of optimization using partitions, etc. Traditionally, optimizing refresh for large and complex datasets have been invoked with existing programming methods using TOM (Tabular Object Model), PowerShell cmdlets, or TMSL (Tabular Model Scripting Language). These methods can, however, require often unreliable, long-running HTTP connections.

The Power BI REST API enables dataset-refresh operations to be carried out asynchronously. By using the REST API, long-running HTTP connections from client applications aren't necessary. Asynchronous refresh also includes additional reliability features such as auto retries and batched commits.

> [!IMPORTANT]
> This feature is in **Preview**. When in preview, functionality and documentation are likely to change.

## Base URL

The base URL follows this format:

```http
https://api.powerbi.com/v1.0/myorg/groups/{groupId}/datasets/{datasetId}/refreshes 
```

By using the base URL, resources and operations can be appended based on parameters. Groups, Datasets, and Refreshes are *collections*. Group, Dataset, and Refresh are *objects*.

:::image type="content" source="media/asynchronous-refresh/pbi-async-refresh-flow.png" border="false" alt-text="Asynchronous refresh flow":::

## Requirements

- GroupId and DatasetId are required.

- The required permission scope is **Dataset.ReadWrite.All**. The number of refreshes is limited according to the general limitations for API-based refreshes for both Pro and Premium datasets.

## Authentication

All calls must be authenticated with a valid Azure Active Directory (OAuth 2) token in the Authorization header. The token must meet the following requirements:

- The token must be either a user token or an application service principal.
- The token must have the correct audience set to https://api.powerbi.com.
- The user or application must have sufficient permissions on the dataset.

REST API modifications do not change permissions as they are currently defined for dataset refreshes.

## POST /refreshes

To perform a refresh operation, use the POST verb on the /refreshes collection to add a new *refresh* object to the collection. The Location header in the response includes the **requestId**. Because the operation is asynchronous, a client application can disconnect and check the status later if required.

Sample request

```http
POST https://api.powerbi.com/v1.0/myorg/groups/f089354e-8366-4e18-aea3-4cb4a3a50b48/datasets/cfafbeb1-8037-4d0c-896e-a46fb27ff229/refreshes
```

The body may resemble the following:

```json
{
    "type": "Full",
    "commitMode": "transactional",
    "maxParallelism": 2,
    "retryCount": 2,
    "objects": [
        {
            "table": "DimCustomer",
            "partition": "DimCustomer"
        },
        {
            "table": "DimDate"
        }
    ]
}

```

Only one refresh operation at a time is accepted for a dataset. If there's a current running refresh operation and another is submitted, a 409 Conflict HTTP status code is returned.

### Parameters

Specifying parameters is not required. If not specified, the default is applied. Please note that asynchronous refresh would be triggered only if any request payload except notifyOption is set.

|Name  |Type  |Default  |Description  |
|---------|---------|---------|---------|
|`type`    |      Enum    |    `automatic`      |    The type of processing to perform. Types are aligned with the TMSL refresh command types: `full`, `clearValues`, `calculate`, `dataOnly`, `automatic`, and `defragment`. <br>`Add` type is not supported.      |
|`commitMode`    |   Enum       |    `transactional`     |     Determines if objects will be committed in batches or only when complete. Modes include: `transactional`, `partialBatch`.     |
|`maxParallelism`     |   Int       |   `10`     |   Determines the maximum number of threads on which to run processing commands in parallel. This value aligned with the `MaxParallelism` property that can be set in the TMSL `Sequence` command or by using other methods.       |
|`retryCount`     |       Int   |    `0`     |    Number of times the operation will retry before failing.      |
|`objects`     |    Array      |    Process the entire dataset      |    An array of objects to be processed. Each object includes `table` when processing the entire table, or `table` and `partition` when processing a partition. If no objects are specified, the entire dataset is refreshed.      |
|`applyRefreshPolicy`    |    Boolean     |    `true`     |   If an incremental refresh policy is defined, `applyRefreshPolicy` will determine if the policy is applied or not. If the policy is not applied, a **process full** operation will leave partition definitions unchanged and all partitions in the table will be fully refreshed. Modes are `true` or `false`. <br><br>Supported behavior: <br>If `commitMode` = `transactional`, <br> then `applyRefreshPolicy` = `true` or `false`. <br>If `commitMode` = `partialBatch`, <br>then `applyRefreshPolicy` = `false`. <br><br>Unsupported behavior: <br>If `commitMode` = `partialBatch`, <br>then `applyRefreshPolicy` = `true`. |
|`effectiveDate`    |    Date     |    Current date     |   If an incremental refresh policy is applied, the `effectiveDate` parameter overrides the current date.       |

### Response

```json
202 Accepted
```

The response also includes a location response-header field to point the caller to the refresh operation that was just created/accepted. Location is that of the new resource which was created by the request, which includes the `requestId`, which is required for some asynchronous refresh operations. For example, in the following response, `requestId` is the last identifier identifier in the response, `87f31ef7-1e3a-4006-9b0b-191693e79e9e`.

```json
x-ms-request-id: 87f31ef7-1e3a-4006-9b0b-191693e79e9e
Location: https://api.powerbi.com/v1.0/myorg/groups/f089354e-8366-4e18-aea3-4cb4a3a50b48/datasets/cfafbeb1-8037-4d0c-896e-a46fb27ff229/refreshes/87f31ef7-1e3a-4006-9b0b-191693e79e9e
```

## GET /refreshes

Use the GET verb on the /refreshes collection to list historical, current, and pending refresh operations.

Here's an example of the response body:

```json
[
    {
        "requestId": "1344a272-7893-4afa-a4b3-3fb87222fdac",
        "refreshType": "ViaApi",
        "startTime": "2020-12-07T02:06:57.1838734Z",
        "endTime": "2020-12-07T02:07:00.4929675Z",
        "status": "succeeded"
    },
    {
        "requestId": "474fc5a0-3d69-4c5d-adb4-8a846fa5580b",
        "startTime": "2020-12-07T01:05:54.157324Z",
        "refreshType": "ViaApi",
        "endTime": "2020-12-07T01:05:57.353371Z",
        "status": "inProgress"
    }
    {
        "requestId": "85a82498-2209-428c-b273-f87b3a1eb905",
        "refreshType": "ViaApi",
        "startTime": "2020-12-07T01:05:54.157324Z",
        "endTime": "2020-12-07T01:05:57.353371Z",
        "status": "notStarted"
    }
]

```

> [!NOTE]
> Power BI might drop additional requests if too many requests are sent in a short period of time. Power BI will perform a refresh, queue the next request, and drop all others. You cannot query status on requests that have been dropped. This is by design.

### Response properties

|Name  |Type  |Description  |
|---------|---------|---------|
|`requestId`     |    Guid     |    The identifier of the refresh request. `requestId` is required to query for individual refresh operation status or delete (cancel) an in-progress refresh operation. |
|`refreshType`   |   RefreshType      |    `OnDemand` indicates refresh was triggered interactively through the Power BI portal. <br>`Scheduled` indicates refresh was triggered by the dataset refresh schedule setting. <br>`ViaApi` indicates refresh was triggered by an API call. <br>`ReliableProcessing` indicates an Asynchronous refresh triggered by an API call.   |
|`startTime`     |    string     |    DateTime of start.     |
|`endTime`     |   string      |    DateTime of end.     |
|`status`     |  string       |   `completed`\*  indicates the refresh operation completed successfully. <br>`failed` indicates the refresh operation failed - serviceExceptionJson will contain the error. <br>`unknown` indicates a completion state that cannot be determined. `endTime` will be empty with this status.   <br>`disabled` indicates refresh was disabled by selective refresh.  |
|`extendedStatus`     |    string     |   Augments the status property to provide additional information.     |

\* In Azure Analysis Services, this status result is 'succeeded'. If migrating an Azure Analysis Services solution to Power BI, you may have to modify your solutions.

### Limit the number of refresh operations returned

The Power BI REST API supports limiting the requested number of entries in the refresh history by using the optional **$top** parameter. If not specified, the default is all available entries.

```http
GET https://api.powerbi.com/v1.0/myorg/datasets/{datasetId}/refreshes?$top={$top}      
```

## GET /refreshes/\<requestId\>

To check the status of a refresh operation, use the GET verb on the refresh object by specifying the **requestId**. Here's an example of the response body. If the operation is in progress, `inProgress` is returned in status.

```json
{
    "startTime": "2020-12-07T02:06:57.1838734Z",
    "endTime": "2020-12-07T02:07:00.4929675Z",
    "type": "full",
    "status": "inProgress",
    "currentRefreshType": "full",
    "objects": [
        {
            "table": "DimCustomer",
            "partition": "DimCustomer",
            "status": "inProgress"
        },
        {
            "table": "DimDate",
            "partition": "DimDate",
            "status": "inProgress"
        }
    ]
}

```

## DELETE /refreshes/\<requestId\>

To cancel an in-progress refresh operation, use the DELETE verb on the refresh object by specifying the `requestId`.

For example,

```http
DELETE https://api.powerbi.com/v1.0/myorg/groups/f089354e-8366-4e18-aea3-4cb4a3a50b48/datasets/cfafbeb1-8037-4d0c-896e-a46fb27ff229/refreshes /1344a272-7893-4afa-a4b3-3fb87222fdac
``````

## Limitations

#### Non-asynchronous refresh operations

Scheduled and on-demand (manual) dataset refreshes cannot be cancelled by using `DELETE /refreshes/<requestId>`.

Scheduled and on-demand (manual) dataset refreshes do not support getting refresh operation details by using `GET /refreshes/<requestId>`.

Get Details and Cancel are new operations for asynchronous refresh only. They are not supported for non-asynchronous refresh operations.

#### Power BI Embedded

If a capacity is paused manually in the Portal or by using PowerShell, or a system outage occurs, if there is an on-going asynchronous refresh operation the status of the refresh will stay in `In-Progress` for the maximum of six hours. If the capacity is resumed within six hours, the asynchronous refresh operation will resume automatically. If the capacity is resumed after more than six hours, the asynchronous refresh operation may return a time out error. The asynchronous refresh operation must then be run again.

## Troubleshooting

#### Problem - Dataset eviction

Power BI uses dynamic memory management to optimize capacity memory. If during an asynchronous refresh operation the dataset is evicted from memory, the following error may be returned:

```json
{
    "messages": [
        {
            "code": "0xC11C0020",
            "message": "Session cancelled because it is connected to a database that has been evicted to free up memory for other operations",
            "type": "Error"
        }
    ]
}

```

#### Solution: Run the asynchronous refresh operation again

To learn more about Dynamic memory management and dataset eviction, see [What is Power BI Premium - How capacities function](../enterprise/service-premium-what-is.md#how-capacities-function).

## Code sample

Here's a C# code sample to get you started, [RestApiSample on GitHub](https://github.com/Microsoft/Analysis-Services/tree/master/RestApiSample).

### To use the code sample

1. Clone or download the repo. Open the RestApiSample solution.
1. Find the line **client.BaseAddress = …** and provide your [base URL](#base-url).

The code sample uses service principal authentication.

## See also

[Using the Power BI REST APIs](/rest/api/power-bi/)  
