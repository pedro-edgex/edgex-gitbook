# Quote Public API

## When to Use This Page

Use quote endpoints for market data reads such as order book depth, K-line history, ticker snapshots, open interest, and market status. These endpoints are public and do not require authentication.

## Minimal Calls

Get depth:

```bash
curl -X GET "https://<api-domain>/api/v2/public/quote/getDepth?contractId=10000001&level=15"
```

Get ticker summary:

```bash
curl -X GET "https://<api-domain>/api/v2/public/quote/getTickerSummary?contractId=10000001"
```

## Common Notes

- Load metadata first so you know which `contractId`, precision, and enums are valid.
- Quote endpoints are read-only; no authentication headers are required.
- For charting or polling clients, prefer a stable refresh cadence and switch to WebSocket when you need real-time updates.

<a id="opIdgetDepth"></a>

## GET Query Order Book Depth

GET /api/v2/public/quote/getDepth

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|contractId|string|No|Contract ID|
|level|integer|Yes|Depth level. Currently 15 and 200 levels available|

> Response Example

> 200 Response

```json
{
    "code": "SUCCESS",
    "data": [
        {
            "startVersion": "201223746",
            "endVersion": "201223747",
            "level": 15,
            "contractId": "10000001",
            "contractName": "BTCUSDT",
            "asks": [
                {
                    "price": "101695.9",
                    "size": "0.579"
                },
                {
                    "price": "101696.0",
                    "size": "0.923"
                },
                {
                    "price": "101703.0",
                    "size": "0.129"
                }
            ],
            "bids": [
                {
                    "price": "101695.5",
                    "size": "1.710"
                },
                {
                    "price": "101694.1",
                    "size": "0.189"
                },
                {
                    "price": "101692.9",
                    "size": "0.223"
                }
            ],
            "depthType": "SNAPSHOT"
        }
    ],
    "msg": null,
    "errorParam": null,
    "requestTime": "1734598036434",
    "responseTime": "1734598036435",
    "traceId": "99b69f04bac0df6e37961f249b9545e4"
}
```

### Response

| Status Code | Status Code Meaning     | Description       | Data Model |
|-------------|-------------------------|-------------------|------------|
| 200         | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1) | successful response | [Result](#schemaresultlistdepth) |

### Response Data Structure

<a id="opIdgetKline"></a>

## GET Query K-Line

GET /api/v2/public/quote/getKline

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|contractId|string|No|Contract ID|
|priceType|string|No|Price type|
|klineType|string|No|K-line type|
|size|integer|No|Number to retrieve. Must be greater than 0 and less than or equal to 1000. Default is 1000|
|offsetData|string|No|Pagination offset. If empty, get the first page|
|filterBeginKlineTimeInclusive|string|No|Query start time (if 0, means from current time). Returns in descending order by time|
|filterEndKlineTimeExclusive|string|No|Query end time|

> Request Example
```
https://edgex-prod-v2.edgex.exchange/api/v2/public/quote/getKline?contractId=10000002&klineType=HOUR_1&filterBeginKlineTimeInclusive=1733416860000&filterEndKlineTimeExclusive=1734601200000&priceType=LAST_PRICE
```

> Response Example

> 200 Response

```json
{
    "code": "SUCCESS",
    "data": {
        "dataList": [
            {
                "klineId": "687194918450962784",
                "contractId": "10000002",
                "contractName": "ETHUSDT",
                "klineType": "HOUR_1",
                "klineTime": "1734595200000",
                "priceType": "LAST_PRICE",
                "trades": "3142",
                "size": "111.96",
                "value": "412199.6286",
                "high": "3694.59",
                "low": "3667.42",
                "open": "3694.57",
                "close": "3670.42",
                "makerBuySize": "52.21",
                "makerBuyValue": "192147.4907"
            }
        ],
        "nextPageOffsetData": ""
    },
    "msg": null,
    "errorParam": null,
    "requestTime": "1734601267556",
    "responseTime": "1734601267581",
    "traceId": "72cfd2eeb27fc602aa64990ad84cd8dd"
}
```

### Response

| Status Code | Status Code Meaning     | Description       | Data Model      |
|-------------|-------------------------|-------------------|-----------------|
| 200         | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1) | successful response | [Result](#schemaresultpagedatakline) |

<a id="opIdgetMultiContractKline"></a>

## GET Query Multi-Contract Quantitative K-Line

GET /api/v2/public/quote/getMultiContractKline

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|contractIdList|array|No|Collection of Contract IDs|
|priceType|string|No|Price type|
|klineType|string|No|K-line type|
|size|integer|No|Number to retrieve. Must be greater than 0 and less than or equal to 200. Default is 200|
|filterBeginKlineTimeInclusive|string|No|Query start time (if 0, means from current time). Returns in descending order by time|
|filterEndKlineTimeExclusive|string|No|Query end time|

> Request Example
```text
https://edgex-prod-v2.edgex.exchange/api/v2/public/quote/getMultiContractKline?contractIdList=10000001&klineType=HOUR_1&filterBeginKlineTimeInclusive=1733416860000&filterEndKlineTimeExclusive=1734601200000&priceType=LAST_PRICE
```

> Response Example

> 200 Response

```json
{
    "code": "SUCCESS",
    "data": [
        {
            "contractId": "10000001",
            "klineList": [
                {
                    "klineId": "687194849731486048",
                    "contractId": "10000001",
                    "contractName": "BTCUSDT",
                    "klineType": "HOUR_1",
                    "klineTime": "1734595200000",
                    "priceType": "LAST_PRICE",
                    "trades": "3123",
                    "size": "7.947",
                    "value": "807240.1268",
                    "high": "101798.4",
                    "low": "101326.3",
                    "open": "101603.8",
                    "close": "101605.6",
                    "makerBuySize": "5.222",
                    "makerBuyValue": "530431.6634"
                }
            ]
        }
    ],
    "msg": null,
    "errorParam": null,
    "requestTime": "1734601896988",
    "responseTime": "1734601897009",
    "traceId": "7edd9609a0c5976c1cb58bdee3d08088"
}
```

### Response

| Status Code | Status Code Meaning     | Description       | Data Model |
|-------------|-------------------------|-------------------|------------|
| 200         | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1) | successful response | [Result](#schemaresultlistcontractkline) |

<a id="opIdgetTicker"></a>

## GET Query 24-Hour Ticker

GET /api/v2/public/quote/getTicker

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|contractId|string|No|Contract ID|

> Response Example

> 200 Response

```json
{
    "code": "SUCCESS",
    "data": [
        {
            "contractId": "10000001",
            "contractName": "BTCUSDT",
            "priceChange": "-2270.5",
            "priceChangePercent": "-0.021849",
            "trades": "79372",
            "size": "499.487",
            "value": "50821443.7464",
            "high": "105331.5",
            "low": "98755.0",
            "open": "103913.2",
            "close": "101642.7",
            "highTime": "1734524115631",
            "lowTime": "1734575388228",
            "startTime": "1734510600000",
            "endTime": "1734597000000",
            "lastPrice": "101642.7",
            "indexPrice": "101676.380723500",
            "oraclePrice": "101636.3750002346932888031005859375",
            "markPrice": "101636.3750002346932888031005859375",
            "openInterest": "0.105",
            "fundingRate": "-0.00012236",
            "fundingTime": "1734595200000",
            "nextFundingTime": "1734609600000"
        }
    ],
    "msg": null,
    "errorParam": null,
    "requestTime": "1734597508246",
    "responseTime": "1734597508250",
    "traceId": "a49014b0ad76a121193d4717294f85fc"
}
```

### Response

| Status Code | Status Code Meaning     | Description       | Data Model |
|-------------|-------------------------|-------------------|------------|
| 200         | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1) | successful response | [Result](#schemaresultlistticker) |

<a id="opIdgetMarketStatus"></a>

## GET Get Market Status

GET /api/v2/public/quote/getMarketStatus

Get stocks market status and limit price information.

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|contractId|integer|No|Contract ID; when specified, returns status and limit price for the contract|

> Response Example

> 200 Response

```json
{
    "code": "SUCCESS",
    "data": {
        "status": true,
        "force": false,
        "contractId": 10000001,
        "markPrice": "101642.7",
        "limitLowPrice": "91478.43",
        "limitHighPrice": "111806.97"
    },
    "msg": null,
    "errorParam": null,
    "requestTime": "1734597508246",
    "responseTime": "1734597508250",
    "traceId": "a49014b0ad76a121193d4717294f85fc"
}
```

### Response

| Status Code | Status Code Meaning     | Description       | Data Model |
|-------------|-------------------------|-------------------|------------|
| 200         | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1) | successful response | [Result](#getmarketstatusmodel) |

## Data Model Notes

- The models below are grouped by endpoint so you can find the payload shape quickly.
- For onboarding, focus first on `DepthPayload`, `KlineRecord`, `TickerRecord`, and `GetMarketStatusPayload`.
- If you only need a single widget or chart, you usually do not need every model on this page.

## Data Models

<a id="schemaresultlistdepth"></a>
### DepthResult

|Name|Type|Description|
|---|---|---|
|code|string|Status code. `SUCCESS` means the request succeeded; other values indicate failure.|
|data|[[Depth](#schemadepth)]|Order-book depth payload returned by the server.|
|errorParam|object|Structured error details returned by the server.|
|requestTime|string(timestamp)|Timestamp when the server received the request.|
|responseTime|string(timestamp)|Timestamp when the server returned the response.|
|traceId|string|Call trace ID|

<a id="schemadepth"></a>
### DepthPayload

|Name|Type|Description|
|---|---|---|
|startVersion|string(int64)|Start order book version number|
|endVersion|string(int64)|End order book version number|
|level|integer(int32)|Depth level|
|contractId|string(int64)|Contract ID|
|contractName|string|Contract name|
|asks|[[BookOrder](#schemabookorder)]|Ask list|
|bids|[[BookOrder](#schemabookorder)]|Bid list|
|depthType|string|Depth type|

#### Enum Values

| Property  | Value               |
|-----------|---------------------|
| depthType | UNKNOWN_DEPTH_TYPE  |
| depthType | SNAPSHOT            |
| depthType | CHANGED             |
| depthType | UNRECOGNIZED       |

<a id="schemabookorder"></a>
### BookOrder

|Name|Type|Description|
|---|---|---|
|price|string(decimal)|Price|
|size|string(decimal)|Quantity|

<a id="schemaresultpagedatakline"></a>
### KlinePageResult

|Name|Type|Description|
|---|---|---|
|code|string|Status code. `SUCCESS` means the request succeeded; other values indicate failure.|
|data|[PageDataKline](#schemapagedatakline)|Paginated K-line data.|
|errorParam|object|Structured error details returned by the server.|
|requestTime|string(timestamp)|Timestamp when the server received the request.|
|responseTime|string(timestamp)|Timestamp when the server returned the response.|
|traceId|string|Call trace ID|

<a id="schemapagedatakline"></a>
### KlinePage

|Name|Type|Description|
|---|---|---|
|dataList|[[Kline](#schemakline)]|K-line records for the current page.|
|nextPageOffsetData|string|Offset token for the next page. Empty when there is no next page.|

<a id="schemakline"></a>
### KlineRecord

|Name|Type|Description|
|---|---|---|
|klineId|string(int64)|K-Line ID|
|contractId|string(int64)|Perpetual contract ID|
|contractName|string|Perpetual contract name|
|klineType|string|K-Line type|
|klineTime|string(int64)|K-Line time|
|priceType|string|Price type of the K-line|
|trades|string(int64)|Number of trades|
|size|string(decimal)|Volume|
|value|string(decimal)|Value|
|high|string(decimal)|High price|
|low|string(decimal)|Low price|
|open|string(decimal)|Opening price|
|close|string(decimal)|Closing price|
|makerBuySize|string(decimal)|Maker buy volume|
|makerBuyValue|string(decimal)|Maker buy value|

#### Enum Values

| Property  | Value              |
|-----------|--------------------|
| klineType | UNKNOWN_KLINE_TYPE  |
| klineType | MINUTE_1            |
| klineType | MINUTE_5            |
| klineType | MINUTE_15           |
| klineType | MINUTE_30           |
| klineType | HOUR_1              |
| klineType | HOUR_2              |
| klineType | HOUR_4              |
| klineType | HOUR_6              |
| klineType | HOUR_8              |
| klineType | HOUR_12             |
| klineType | DAY_1               |
| klineType | WEEK_1              |
| klineType | MONTH_1             |
| klineType | UNRECOGNIZED       |
| priceType | UNKNOWN_PRICE_TYPE |
| priceType | ORACLE_PRICE       |
| priceType | INDEX_PRICE        |
| priceType | LAST_PRICE         |
| priceType | ASK1_PRICE        |
| priceType | BID1_PRICE        |
| priceType | OPEN_INTEREST        |
| priceType | UNRECOGNIZED       |

<a id="schemaresultlistcontractkline"></a>
### ContractMultiKlineResult

|Name|Type|Description|
|---|---|---|
|code|string|Status code. `SUCCESS` means the request succeeded; other values indicate failure.|
|data|[[ContractMultiKline](#schemacontractmultikline)]|Multi-contract K-line payload returned by the server.|
|errorParam|object|Structured error details returned by the server.|
|requestTime|string(timestamp)|Timestamp when the server received the request.|
|responseTime|string(timestamp)|Timestamp when the server returned the response.|
|traceId|string|Call trace ID|

<a id="schemacontractmultikline"></a>
### ContractMultiKline

|Name|Type|Description|
|---|---|---|
|contractId|string(int64)|Perpetual contract ID|
|klineList|[[Kline](#schemakline)]|Collection of kline data|

<a id="schemaresultlistticker"></a>
### TickerResult

|Name|Type|Description|
|---|---|---|
|code|string|Status code. `SUCCESS` means the request succeeded; other values indicate failure.|
|data|[[Ticker](#schematicker)]|24-hour ticker data returned by the server.|
|errorParam|object|Structured error details returned by the server.|
|requestTime|string(timestamp)|Timestamp when the server received the request.|
|responseTime|string(timestamp)|Timestamp when the server returned the response.|
|traceId|string|Call trace ID|

<a id="schematicker"></a>
### TickerRecord

|Name|Type|Description|
|---|---|---|
|contractId|string(int64)|Contract ID|
|contractName|string|Contract Name|
|priceChange|string(decimal)|Price change|
|priceChangePercent|string(decimal)|Price change percentage|
|trades|string(int64)|24-hour number of trades|
|size|string(decimal)|24-hour trading volume|
|value|string(decimal)|24-hour trading value|
|high|string(decimal)|24-hour high price|
|low|string(decimal)|24-hour low price|
|open|string(decimal)|24-hour opening price|
|close|string(decimal)|24-hour closing price|
|highTime|string(int64)|24-hour high price time|
|lowTime|string(int64)|24-hour low price time|
|startTime|string(int64)|24-hour quote start time|
|endTime|string(int64)|24-hour quote end time|
|lastPrice|string(decimal)|Latest trade price|
|indexPrice|string(decimal)|Current index price|
|oraclePrice|string(decimal)|Current oracle price|
|markPrice|string(decimal)|Current mark price|
|openInterest|string(decimal)|Open Interest|
|fundingRate|string|Latest settled funding rate in the current ticker window.|
|fundingTime|string(int64)|Timestamp of the latest funding settlement.|
|nextFundingTime|string(int64)|Timestamp of the next scheduled funding settlement.|

<a id="getmarketstatusmodel"></a>
### GetMarketStatusResult

|Name|Type|Description|
|---|---|---|
|code|string|Status code. `SUCCESS` means the request succeeded; other values indicate failure.|
|data|[GetMarketStatus](#schemagetmarketstatus)|Market-status payload returned by the server.|
|errorParam|object|Structured error details returned by the server.|
|requestTime|string(timestamp)|Timestamp when the server received the request.|
|responseTime|string(timestamp)|Timestamp when the server returned the response.|
|traceId|string|Call trace ID|

<a id="schemagetmarketstatus"></a>
### GetMarketStatusPayload

|Name|Type|Description|
|---|---|---|
|status|boolean|Whether the market is currently open. `true` means open.|
|force|boolean|Whether the status was manually forced by backend control.|
|contractId|integer(int64)|Contract ID included in the market-status response.|
|markPrice|string|Latest mark price for the contract.|
|limitLowPrice|string|Lower limit price|
|limitHighPrice|string|Upper limit price|
