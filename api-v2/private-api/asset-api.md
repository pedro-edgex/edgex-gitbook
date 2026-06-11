# Asset Private API

## When to Use This Page

Use this page for the asset flows currently exposed by the V2 SDKs:

- Chain-to-spot deposit data retrieval
- Spot/perpv2 to chain withdraw flow
- Asset flow query

## Path Note

Although this page belongs to the V2 private API section, the asset endpoints currently used by both the Go SDK and Python SDK are under:

`/api/v1/private/unified-asset`

The legacy `/api/v2/private/assets/...` endpoints are no longer the SDK path.

## Domain Note

Requests on this page use the Asset API production domain:

`https://spot.edgex.exchange`

Do not combine these unified-asset paths with the default REST production domain `https://edgex-prod-v2.edgex.exchange`.

## Common Notes

- These endpoints use the same private REST authentication described in [Authentication](../authentication.md).
- Withdraw submission requires both private REST authentication and an EIP-712 user signature.
- Deposit data retrieval does not perform signing by itself; it returns chain transaction data for the client to use.
- `withdraw` in the SDK is a composed flow built from `getFeeByAssetFlow -> getEIP712Data -> submitAssetFlow`.

## Unified Attempt Object

Most unified-asset endpoints use an `attempt` object.

### Withdraw Attempt Example

```json
{
  "userAddress": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
  "privyAddress": "0x0000000000000000000000000000000000000000",
  "source": "spot",
  "sourceAccount": "12345",
  "tokenAddress": "0x98d2919b9A214E6Fa5384AC81E6864bA686Ad74c",
  "amount": "1000",
  "fee": "0",
  "destination": "chain-3343",
  "destinationAccount": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
  "clientWithdrawId": "849849126827855872",
  "expireTime": 123456
}
```

### Deposit Attempt Example

```json
{
  "userAddress": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
  "privyAddress": "0x0000000000000000000000000000000000000000",
  "source": "chain-3343",
  "sourceAccount": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
  "tokenAddress": "0x98d2919b9A214E6Fa5384AC81E6864bA686Ad74c",
  "amount": "1000",
  "fee": "0",
  "destination": "spot",
  "destinationAccount": "12345"
}
```

### Attempt Fields

|Name|Type|Required|Description|
|---|---|---|---|
|userAddress|string|Yes|Wallet address of the user.|
|privyAddress|string|No|Privy wallet address. SDK default is `0x0000000000000000000000000000000000000000`.|
|source|string|Yes|Asset source. For withdraw use `spot` or `perpv2` , and deposit uses `chain-<chainId>`.|
|sourceAccount|string|Yes|Source account identifier. For spot/perpv2 withdraw this is usually the EdgeX account ID; for chain deposit this is usually the user wallet address.|
|tokenAddress|string|Yes|Token contract address. Native token flows use `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`.|
|amount|string|Yes|Raw amount string.|
|fee|string|Yes|Fee string. Initial request commonly uses `0`, then withdraw flow updates it from `getFeeByAssetFlow`.|
|destination|string|Yes|Destination identifier. Withdraw uses `chain-<chainId>`; deposit uses `spot` or `perpv2`.|
|destinationAccount|string|Yes|Destination account identifier. Withdraw usually uses the same wallet address as `userAddress`; deposit usually uses the spot/perpv2 account ID.|
|clientWithdrawId|string|Withdraw only|Client-defined withdraw ID used for idempotency.|
|expireTime|integer|Withdraw only|Unix timestamp in seconds.|

<a id="opIdgetFeeByAssetFlow"></a>

## POST Get Fee By Asset Flow

POST `/api/v1/private/unified-asset/getFeeByAssetFlow`

Returns the fee for a unified asset flow attempt. In SDK withdraw flow, this is called before requesting EIP-712 data.

> Body Request Example

```json
{
  "attempt": {
    "userAddress": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "privyAddress": "0x0000000000000000000000000000000000000000",
    "source": "spot",
    "sourceAccount": "12345",
    "tokenAddress": "0x98d2919b9A214E6Fa5384AC81E6864bA686Ad74c",
    "amount": "1000",
    "fee": "0",
    "destination": "chain-3343",
    "destinationAccount": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "clientWithdrawId": "849849126827855872",
    "expireTime": 123456
  }
}
```

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|body|object|Yes|Request body.|
|body.attempt|object|Yes|Unified asset flow attempt.|

> Response Example

```json
{
  "code": "SUCCESS",
  "data": {
    "fee": "10"
  },
  "msg": null,
  "errorParam": null,
  "requestTime": "1234567890123",
  "responseTime": "1234567890124",
  "traceId": "abc123"
}
```

<a id="opIdgetEIP712Data"></a>

## POST Get EIP-712 Data

POST `/api/v1/private/unified-asset/getEIP712Data`

Returns the EIP-712 typed-data payload that must be signed for withdraw submission.

> Body Request Example

```json
{
  "attempt": {
    "userAddress": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "privyAddress": "0x0000000000000000000000000000000000000000",
    "source": "spot",
    "sourceAccount": "12345",
    "tokenAddress": "0x98d2919b9A214E6Fa5384AC81E6864bA686Ad74c",
    "amount": "990",
    "fee": "10",
    "destination": "chain-3343",
    "destinationAccount": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "clientWithdrawId": "849849126827855872",
    "expireTime": 123456
  }
}
```

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|body|object|Yes|Request body.|
|body.attempt|object|Yes|Unified asset flow attempt after fee has been applied.|

> Response Example

```json
{
  "code": "SUCCESS",
  "data": {
    "types": {
      "AssetFlowAttempt": {
        "fields": [
          {
            "name": "amount",
            "type": "uint256"
          }
        ]
      }
    },
    "primaryType": "AssetFlowAttempt",
    "domain": {
      "name": "EdgeX",
      "version": "1"
    },
    "messageJson": "{\"amount\":\"990\"}"
  },
  "msg": null,
  "errorParam": null,
  "requestTime": "1234567890123",
  "responseTime": "1234567890124",
  "traceId": "abc123"
}
```

## SDK Withdraw Signing Flow

Both the Go SDK and Python SDK implement withdraw in the same order:

1. Build withdraw `attempt`.
2. Call `getFeeByAssetFlow`.
3. Replace `attempt.fee` and `attempt.amount` with the net amount.
4. Call `getEIP712Data`.
5. Sign the returned typed data with the wallet private key.
6. Submit the flow with `submitAssetFlow`.

The submitted `userSignature` is `0x`-prefixed in both SDKs.

<a id="opIdgetDepositData"></a>

## POST Get Deposit Data

POST `/api/v1/private/unified-asset/getDepositData`

Returns deposit transaction data for chain-to-spot asset flows.

> Body Request Example

```json
{
  "attempt": {
    "userAddress": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "privyAddress": "0x0000000000000000000000000000000000000000",
    "source": "chain-33431",
    "sourceAccount": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "tokenAddress": "0x98d2919b9A214E6Fa5384AC81E6864bA686Ad74c",
    "amount": "1000",
    "fee": "0",
    "destination": "spot",
    "destinationAccount": "12345"
  },
  "extraData": ""
}
```

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|body|object|Yes|Request body.|
|body.attempt|object|Yes|Unified asset deposit attempt.|
|body.extraData|string|No|Extra payload passed through to the backend.|

> Response Example

```json
{
  "code": "SUCCESS",
  "data": {
    "chainId": 33431,
    "to": "0x86245522276a9F845a65a591b485cd4AaA51D86c",
    "data": "0x1234",
    "value": "0"
  },
  "msg": null,
  "errorParam": null,
  "requestTime": "1234567890123",
  "responseTime": "1234567890124",
  "traceId": "abc123"
}
```

<a id="opIdsubmitAssetFlow"></a>

## POST Submit Asset Flow

POST `/api/v1/private/unified-asset/submitAssetFlow`

Submits a signed unified asset flow. In the SDK, this is used by the withdraw flow after EIP-712 signing.

> Body Request Example

```json
{
  "attempt": {
    "userAddress": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "privyAddress": "0x0000000000000000000000000000000000000000",
    "source": "spot",
    "sourceAccount": "12345",
    "tokenAddress": "0x98d2919b9A214E6Fa5384AC81E6864bA686Ad74c",
    "amount": "990",
    "fee": "10",
    "destination": "chain-3343",
    "destinationAccount": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
    "clientWithdrawId": "849849126827855872",
    "expireTime": 123456
  },
  "userSignature": "0x1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111",
  "extraData": "",
  "privyIdentityToken": ""
}
```

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|body|object|Yes|Request body.|
|body.attempt|object|Yes|Unified asset flow attempt.|
|body.userSignature|string|Yes|EIP-712 signature produced from `getEIP712Data`.|
|body.extraData|string|No|Extra payload passed through to the backend.|
|body.privyIdentityToken|string|No|Privy identity token when Privy flow is used.|

> Response Example

```json
{
  "code": "SUCCESS",
  "data": {
    "flowId": "1"
  },
  "msg": null,
  "errorParam": null,
  "requestTime": "1234567890123",
  "responseTime": "1234567890124",
  "traceId": "abc123"
}
```

<a id="opIdqueryAssetFlows"></a>

## GET Query Asset Flows

GET `/api/v1/private/unified-asset/queryAssetFlows`

Queries unified asset flow records.

### Request Parameters

|Name|Type|Required|Description|
|---|---|---|---|
|params|object|No|Query parameters passed directly to the backend query API. The current Go SDK accepts `map[string]string`; the current Python SDK accepts `Dict[str, Any]`.|

> Response Example

```json
{
  "code": "SUCCESS",
  "data": {},
  "msg": null,
  "errorParam": null,
  "requestTime": "1234567890123",
  "responseTime": "1234567890124",
  "traceId": "abc123"
}
```

## SDK Convenience Methods

The SDKs also provide the following convenience methods on top of the raw API endpoints above:

- `CreateWithdraw`: wraps `getFeeByAssetFlow -> getEIP712Data -> submitAssetFlow`
- `GetSpotDepositData`: builds a chain-to-spot deposit attempt, then calls `getDepositData`
