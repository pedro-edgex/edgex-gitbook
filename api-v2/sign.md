# L2 Signature Guide

This document explains how to create Layer 2 (L2) signatures for operations on the EdgeX platform, including orders, withdrawals, and transfers. EdgeX V2 uses **EIP-712 typed data signing** for L2 operations. These signatures are separate from the HMAC authentication used by private REST and WebSocket connections.

---

## Overview

### What is L2 Signature?

Layer 2 signature is a cryptographic signature that authorizes operations on EdgeX's Layer 2 scaling solution. These signatures:

- **Authorize trading operations** (orders, cancellations)
- **Approve asset movements** (withdrawals, transfers)
- **Ensure security** without requiring on-chain gas fees for every action
- **Enable fast execution** while maintaining cryptographic security

### Signature Algorithm: EIP-712

EdgeX V2 uses **EIP-712** (Ethereum Improvement Proposal 712) for structured typed data hashing and signing. EIP-712 provides:

- **Human-readable** structured data
- **Type-safe** signing
- **Domain separation** to prevent signature reuse across different contracts
- **Standard** supported by Ethereum wallets like MetaMask

> **Note**: This is different from V1 which used StarkEx's Pedersen hash-based signatures. V2 simplifies integration by using standard Ethereum signing.

---

## Getting the Signer Key

The Signer Key is obtained from the same **API Management -> Perps V2 -> SDK Signer** dialog used for API credentials.

For the full retrieval flow, screenshots, and security notes, see [Authentication](authentication.md#how-to-get-api-credentials).

Use the **Private Key (Signer Key)** from that dialog for the L2 signatures described in this guide.

---

## EIP-712 Signature Structure

### Domain Separator

The domain separator ensures signatures are only valid for EdgeX platform:

```go
Domain {
    name:              "EdgeX"
    version:           "1"
    chainId:           "your-chain-id"        // From metadata.global.nativeChainId
    verifyingContract: "contract-address"     // From metadata.global.contractAddress
}
```

**How to get domain values**:

```go
// Get metadata first
metadata, err := client.GetMetaData(ctx)

chainID := metadata.Data.Global.NativeChainId
if chainID == "" {
    chainID = metadata.Data.Global.ChainId
}
verifyingContract := metadata.Data.Global.ContractAddress
```

### Common Type Definitions

All L2 signatures use the `OrderBase` struct:

```go
OrderBase {
    nonce:               uint256    // Unique identifier derived from clientOrderId
    signer:              address    // Your trading address
    accountId:           uint64     // Your EdgeX account ID
    expirationTimestamp: uint256    // When the signature expires (seconds)
}
```

---

## Set Margin Mode Signature

`setMarginMode` uses a dedicated EIP-712 payload. It is signed with the **trading private key** and then submitted together with the normal private REST authentication headers.

### EIP-712 Type Definition

```go
Types: {
    EIP712Domain: [
        { name: "name",              type: "string" },
        { name: "version",           type: "string" },
        { name: "chainId",           type: "uint256" },
        { name: "verifyingContract", type: "address" }
    ],
    SetMarginPreferenceParams: [
        { name: "accountId",  type: "uint64" },
        { name: "assetId",    type: "uint64" },
        { name: "marginMode", type: "uint8" },
        { name: "nonce",      type: "uint256" },
        { name: "signer",     type: "address" }
    ]
}
```

### Parameters Calculation

#### 1. accountId and assetId

```go
accountID := client.GetAccountID()
assetID := contractID // The REST request field is contractId; the typed-data field is assetId
```

#### 2. marginMode

```go
marginMode := strings.TrimSpace(params.MarginMode)
marginModeUint, err := strconv.ParseUint(marginMode, 10, 8)
if err != nil {
    return nil, fmt.Errorf("invalid marginMode: %w", err)
}
```

Only `0` and `1` are accepted by the backend.

#### 3. Nonce Calculation

Nonce is derived from `clientOrderId` in the SDK flow. If it is empty, the SDK generates one first:

```go
clientOrderID := strings.TrimSpace(params.ClientOrderID)
if clientOrderID == "" {
    clientOrderID = internal.GetRandomClientId()
}
nonce := internal.CalcNonce(clientOrderID)
```

#### 4. L2 Expire Time

The Go SDK rounds up to the next full hour first, then adds 14 days:

```go
func calcSetMarginModeL2ExpireTime(now time.Time) string {
    nowMillis := now.UnixMilli()
    nextHourMillis := ((nowMillis + 3600000 - 1) / 3600000) * 3600000
    return strconv.FormatInt(nextHourMillis+14*24*60*60*1000, 10)
}
```

#### 5. Signer Address

```go
tradingSigner, err := c.ResolveSignerAddress()
if err != nil {
    return nil, fmt.Errorf("trading private key is required for v2 EIP-712 margin mode signing: %w", err)
}
if !common.IsHexAddress(tradingSigner) {
    return nil, fmt.Errorf("invalid signer address: %s", tradingSigner)
}
tradingSigner = common.HexToAddress(tradingSigner).Hex()
```

### Complete Golang Example

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
    "time"

    "github.com/edgex-Tech/edgex-golang-sdk/sdk/internal"
    "github.com/edgex-Tech/edgex-golang-sdk/sdk/metadata"
    "github.com/ethereum/go-ethereum/common"
)

func calcSetMarginModeL2ExpireTime(now time.Time) string {
    nowMillis := now.UnixMilli()
    nextHourMillis := ((nowMillis + 3600000 - 1) / 3600000) * 3600000
    return strconv.FormatInt(nextHourMillis+14*24*60*60*1000, 10)
}

func SignSetMarginMode(
    signerPrivateKey string,
    accountID int64,
    params SetMarginModeParams,
    md *metadata.MetaData,
) (internal.TypedData, map[string]any, error) {
    if md == nil || md.Global == nil {
        return internal.TypedData{}, nil, fmt.Errorf("metadata.global is required")
    }

    chainID := strings.TrimSpace(md.Global.NativeChainId)
    if chainID == "" {
        chainID = strings.TrimSpace(md.Global.ChainId)
    }
    if chainID == "" {
        return internal.TypedData{}, nil, fmt.Errorf("metadata.global.nativeChainId/chainId is required")
    }
    verifyingContract := strings.TrimSpace(md.Global.ContractAddress)
    if verifyingContract == "" {
        return internal.TypedData{}, nil, fmt.Errorf("metadata.global.contractAddress is required")
    }

    clientOrderID := strings.TrimSpace(params.ClientOrderID)
    if clientOrderID == "" {
        clientOrderID = internal.GetRandomClientId()
    }
    l2Nonce := internal.CalcNonce(clientOrderID)
    l2ExpireTime := calcSetMarginModeL2ExpireTime(time.Now())

    marginMode := strings.TrimSpace(params.MarginMode)
    marginModeUint, err := strconv.ParseUint(marginMode, 10, 8)
    if err != nil {
        return internal.TypedData{}, nil, fmt.Errorf("invalid marginMode: %w", err)
    }

    tradingSigner, err := ResolveSignerAddressFromPrivateKey(signerPrivateKey)
    if err != nil {
        return internal.TypedData{}, nil, fmt.Errorf("trading private key is required for v2 EIP-712 margin mode signing: %w", err)
    }
    if !common.IsHexAddress(tradingSigner) {
        return internal.TypedData{}, nil, fmt.Errorf("invalid signer address: %s", tradingSigner)
    }
    tradingSigner = common.HexToAddress(tradingSigner).Hex()

    domain, err := internal.NewTypedDataDomain("EdgeX", "1", chainID, verifyingContract)
    if err != nil {
        return internal.TypedData{}, nil, fmt.Errorf("failed to build EIP-712 domain: %w", err)
    }

    typedData := internal.TypedData{
        Types: internal.TypedDataTypes{
            "EIP712Domain": {
                {Name: "name", Type: "string"},
                {Name: "version", Type: "string"},
                {Name: "chainId", Type: "uint256"},
                {Name: "verifyingContract", Type: "address"},
            },
            "SetMarginPreferenceParams": {
                {Name: "accountId", Type: "uint64"},
                {Name: "assetId", Type: "uint64"},
                {Name: "marginMode", Type: "uint8"},
                {Name: "nonce", Type: "uint256"},
                {Name: "signer", Type: "address"},
            },
        },
        PrimaryType: "SetMarginPreferenceParams",
        Domain:      domain,
        Message: internal.TypedDataMessage{
            "accountId":  strconv.FormatInt(accountID, 10),
            "assetId":    strings.TrimSpace(params.ContractID),
            "marginMode": strconv.FormatUint(marginModeUint, 10),
            "nonce":      strconv.FormatInt(l2Nonce, 10),
            "signer":     tradingSigner,
        },
    }

    l2Signature, err := internal.SignTypedDataWithPrivateKey(signerPrivateKey, typedData)
    if err != nil {
        return internal.TypedData{}, nil, fmt.Errorf("failed to sign margin mode payload: %w", err)
    }

    requestBody := map[string]any{
        "accountId":    strconv.FormatInt(accountID, 10),
        "contractId":   strings.TrimSpace(params.ContractID),
        "marginMode":   marginMode,
        "l2Nonce":      strconv.FormatInt(l2Nonce, 10),
        "l2ExpireTime": l2ExpireTime,
        "signer":       tradingSigner,
        "l2Signature":  l2Signature,
    }
    return typedData, requestBody, nil
}
```

### REST Request Example

```json
{
  "accountId": "543429922991899150",
  "contractId": "10000001",
  "marginMode": "1",
  "l2Nonce": "123456",
  "l2ExpireTime": "1719212400000",
  "signer": "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
  "l2Signature": "0x..."
}
```

---

## Order Signature (Limit Orders)

### EIP-712 Type Definition

```go
Types: {
    EIP712Domain: [
        { name: "name",              type: "string" },
        { name: "version",           type: "string" },
        { name: "chainId",           type: "uint256" },
        { name: "verifyingContract", type: "address" }
    ],
    OrderBase: [
        { name: "nonce",               type: "uint256" },
        { name: "signer",              type: "address" },
        { name: "accountId",           type: "uint64" },
        { name: "expirationTimestamp", type: "uint256" }
    ],
    LimitOrderParams: [
        { name: "base",              type: "OrderBase" },
        { name: "amountSynthetic",   type: "int256" },
        { name: "amountCollateral",  type: "int256" },
        { name: "amountFee",         type: "uint256" },
        { name: "assetIdSynthetic",  type: "uint64" },
        { name: "assetIdCollateral", type: "uint64" },
        { name: "isBuyingSynthetic", type: "bool" },
        { name: "parentOrderHash",   type: "bytes32" },
        { name: "subOrderIndex",     type: "uint256" }
    ]
}
```

### Parameters Calculation

#### 1. Amount Conversions

All amounts must be converted using their respective resolutions:

```go
// Get resolution values from metadata
contract := metadata.ContractList[...]
quoteCoin := metadata.CoinList[...]

contractResolution := parseResolution(contract.Resolution, contract.StarkExResolution)
quoteResolution := parseResolution(quoteCoin.Resolution, quoteCoin.StarkExResolution)

// Calculate amounts
amountSynthetic  = size × contractResolution
amountCollateral = (price × size) × quoteResolution
amountFee        = (price × size × feeRate) × quoteResolution
```

**Example**:
```go
// BTC-USDT perpetual
// size = 0.001 BTC
// price = 97463.4 USDT
// contractResolution = 10^10 (from contract.starkExResolution)
// quoteResolution = 10^10 (from quoteCoin.starkExResolution)

amountSynthetic  = 0.001 × 10^10 = 10000000
amountCollateral = (97463.4 × 0.001) × 10^10 = 974634000000
amountFee        = (97463.4 × 0.001 × 0.0005) × 10^10 = 487317000 (rounded up)
```

#### 2. Nonce Calculation

Nonce is deterministically derived from `clientOrderId`:

```go
import (
    "crypto/sha256"
    "math/big"
)

func CalcNonce(clientOrderID string) int64 {
    hash := sha256.Sum256([]byte(clientOrderID))
    hashBigInt := new(big.Int).SetBytes(hash[:])
    
    // Modulo 2^63 to fit in positive int64
    maxInt64 := new(big.Int).Lsh(big.NewInt(1), 63)
    nonce := new(big.Int).Mod(hashBigInt, maxInt64)
    
    return nonce.Int64()
}
```

#### 3. Expiration Timestamp

```go
nowMillis := time.Now().UnixMilli()

// Order expires in 30 days
orderExpireWindowMillis := int64(30 * 24 * 60 * 60 * 1000)
l2ExpireTime := nowMillis + orderExpireWindowMillis

// L1 offset (8 days)
l1OffsetMillis := int64(8 * 24 * 60 * 60 * 1000)
expireTime := l2ExpireTime - l1OffsetMillis

// Convert to seconds for signature
expirationTimestamp := expireTime / 1000
```

#### 4. isBuyingSynthetic Flag

```go
// For BUY orders (going long)
isBuyingSynthetic := true   // Buying BTC synthetic with USDT collateral

// For SELL orders (going short)
isBuyingSynthetic := false  // Selling BTC synthetic for USDT collateral
```

### Complete Golang Example

```go
package main

import (
    "context"
    "fmt"
    "strconv"
    "time"
    
    "github.com/shopspring/decimal"
)

// SignLimitOrder shows the calculation flow for an EIP-712 limit-order signature
func SignLimitOrder(
    signerPrivateKey string,
    accountID int64,
    signerAddress string,
    contract Contract,
    quoteCoin Coin,
    metadata Global,
    side string,        // "BUY" or "SELL"
    price decimal.Decimal,
    size decimal.Decimal,
    clientOrderID string,
) (string, error) {
    
    // 1. Calculate amounts with resolutions
    contractResolution, _ := decimal.NewFromString(contract.StarkExResolution)
    quoteResolution, _ := decimal.NewFromString(quoteCoin.StarkExResolution)
    
    amountSynthetic := size.Mul(contractResolution).BigInt()
    amountCollateral := price.Mul(size).Mul(quoteResolution).BigInt()
    
    // Calculate fee (assuming 0.05% taker fee)
    feeRate := decimal.NewFromFloat(0.0005)
    limitFee := price.Mul(size).Mul(feeRate).Ceil()
    amountFee := limitFee.Mul(quoteResolution).BigInt()
    
    // 2. Calculate nonce from client order ID
    nonce := CalcNonce(clientOrderID)
    
    // 3. Calculate expiration
    nowMillis := time.Now().UnixMilli()
    orderExpireWindowMillis := int64(30 * 24 * 60 * 60 * 1000) // 30 days
    l1OffsetMillis := int64(8 * 24 * 60 * 60 * 1000)          // 8 days
    l2ExpireTime := nowMillis + orderExpireWindowMillis
    expireTime := l2ExpireTime - l1OffsetMillis
    expirationTimestamp := expireTime / 1000 // Convert to seconds
    
    // 4. Build EIP-712 domain
    typedDomain, err := internal.NewTypedDataDomain(
        "EdgeX",
        "1",
        metadata.NativeChainId,
        metadata.ContractAddress,
    )
    if err != nil {
        return "", err
    }
    
    // 5. Build typed data structure
    typedData := internal.TypedData{
        Types: internal.TypedDataTypes{
            "EIP712Domain": []internal.TypedDataType{
                {Name: "name", Type: "string"},
                {Name: "version", Type: "string"},
                {Name: "chainId", Type: "uint256"},
                {Name: "verifyingContract", Type: "address"},
            },
            "OrderBase": []internal.TypedDataType{
                {Name: "nonce", Type: "uint256"},
                {Name: "signer", Type: "address"},
                {Name: "accountId", Type: "uint64"},
                {Name: "expirationTimestamp", Type: "uint256"},
            },
            "LimitOrderParams": []internal.TypedDataType{
                {Name: "base", Type: "OrderBase"},
                {Name: "amountSynthetic", Type: "int256"},
                {Name: "amountCollateral", Type: "int256"},
                {Name: "amountFee", Type: "uint256"},
                {Name: "assetIdSynthetic", Type: "uint64"},
                {Name: "assetIdCollateral", Type: "uint64"},
                {Name: "isBuyingSynthetic", Type: "bool"},
                {Name: "parentOrderHash", Type: "bytes32"},
                {Name: "subOrderIndex", Type: "uint256"},
            },
        },
        PrimaryType: "LimitOrderParams",
        Domain:      typedDomain,
        Message: internal.TypedDataMessage{
            "base": map[string]interface{}{
                "nonce":               strconv.FormatInt(nonce, 10),
                "signer":              signerAddress,
                "accountId":           strconv.FormatInt(accountID, 10),
                "expirationTimestamp": strconv.FormatInt(expirationTimestamp, 10),
            },
            "amountSynthetic":   amountSynthetic.String(),
            "amountCollateral":  amountCollateral.String(),
            "amountFee":         amountFee.String(),
            "assetIdSynthetic":  contract.ContractId,
            "assetIdCollateral": quoteCoin.CoinId,
            "isBuyingSynthetic": side == "BUY",
            "parentOrderHash":   "0x0000000000000000000000000000000000000000000000000000000000000000",
            "subOrderIndex":     "0",
        },
    }
    
    // 6. Sign the typed data
    signature, err := internal.SignTypedDataWithPrivateKey(signerPrivateKey, typedData)
    if err != nil {
        return "", fmt.Errorf("failed to sign typed data: %w", err)
    }
    
    return signature, nil
}
```

### Using EdgeX Golang SDK

The SDK handles all signature generation automatically. Replace `<api-domain>` with `edgex-prod-v2.edgex.exchange` for standard REST APIs, and use `https://spot.edgex.exchange` for Asset API calls:

```go
package main

import (
    "context"
    "log"
    
    "github.com/edgex-Tech/edgex-golang-sdk/sdk"
    "github.com/edgex-Tech/edgex-golang-sdk/sdk/order"
    "github.com/shopspring/decimal"
)

func main() {
    // Create client with trading private key
    client, err := sdk.NewClient(&sdk.Config{
        BaseURL:       "https://<api-domain>",
        AccountID:     724625476626153743,
        APIKey:        "your-api-key",
        APIPassphrase: "your-api-passphrase",
        APISecret:     "your-api-secret",
        SignerPriKey:  "your-signer-private-key",  // Signer key for L2 signatures
    })
    if err != nil {
        log.Fatal(err)
    }
    
    ctx := context.Background()
    
    // SDK automatically signs the order with EIP-712
    result, err := client.CreateOrder(ctx, &order.CreateOrderParams{
        ContractId: "10000001",
        Side:       order.OrderSideBuy,
        Type:       order.OrderTypeLimit,
        Price:      "97463.4",
        Size:       "0.001",
    }, nil, decimal.NewFromFloat(97463.4))
    
    if err != nil {
        log.Fatal(err)
    }
    
    log.Printf("Order created: %s", result.Data.OrderId)
}
```

---

## Withdrawal Signature

The current V2 SDK withdrawal path uses the unified-asset API. It no longer builds a fixed local `WithdrawParams` payload like the legacy asset flow.

### Current Unified-Asset Flow

Unified-asset withdraw uses this sequence:

1. Build the withdraw `attempt`.
2. Call `getFeeByAssetFlow`.
3. Update `attempt.fee` and `attempt.amount` to the net amount.
4. Call `getEIP712Data`.
5. Sign the returned EIP-712 typed data with the **wallet private key**.
6. Submit the signed request through `submitAssetFlow`.

### Key Differences From Order / setMarginMode

- The typed-data schema is returned by the server from `getEIP712Data`; it is not hard-coded in the SDK.
- The signature key is the **wallet private key**.
- The submitted field name is `userSignature`.
- The submitted signature is `0x`-prefixed.

### Unified Attempt Example

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

After `getFeeByAssetFlow`, the SDK mutates the same payload before signing, for example:

```json
{
  "amount": "990",
  "fee": "10"
}
```

### EIP-712 Response Shape

The SDK signs the typed data returned by `getEIP712Data`. A typical response shape is:

```json
{
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
}
```

The SDK normalizes this server response into an EIP-712 typed-data object and then signs it directly.

### Complete Golang Example

The SDK already encapsulates the full withdraw signing flow:

```go
package main

import (
    "context"
    "log"

    "github.com/edgex-Tech/edgex-golang-sdk/sdk"
    "github.com/edgex-Tech/edgex-golang-sdk/sdk/unified_asset"
)

func main() {
    client, err := sdk.NewClient(&sdk.Config{
        BaseURL:        "https://<api-domain>",
        AccountID:      12345,
        APIKey:         "your-api-key",
        APIPassphrase:  "your-api-passphrase",
        APISecret:      "your-api-secret",
        WalletPriKey:   "your-wallet-private-key",
    })
    if err != nil {
        log.Fatal(err)
    }

    result, err := client.CreateWithdraw(context.Background(), unified_asset.CreateWithdrawParams{
        AmountRaw:   "1000",
        UserAddress: "0xFCAd0B19bB29D4674531d6f115237E16AfCE377c",
        TokenAddress:"0x98d2919b9A214E6Fa5384AC81E6864bA686Ad74c",
        ChainID:     3343,
    })
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("withdraw result: %#v", result)
}
```

Internally, the SDK performs:

1. `GetFeeByAssetFlow`
2. `GetEIP712Data`
3. `SignTypedDataWithWalletKey`
4. `SubmitAssetFlow`

### Submit Request Example

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
  "userSignature": "0x...",
  "extraData": "",
  "privyIdentityToken": ""
}
```

---

## Transfer Signature

### EIP-712 Type Definition

```go
Types: {
    EIP712Domain: [...],  // Same as order
    OrderBase: [...],     // Same as order
    TransferParams: [
        { name: "base",                type: "OrderBase" },
        { name: "amount",              type: "uint256" },
        { name: "assetIdCollateral",   type: "uint64" },
        { name: "receiverAccountId",   type: "uint64" },
        { name: "receiverPublicKey",   type: "address" }
    ]
}
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `amount` | uint256 | Transfer amount in base units |
| `assetIdCollateral` | uint64 | Coin ID being transferred |
| `receiverAccountId` | uint64 | Recipient's EdgeX account ID |
| `receiverPublicKey` | address | Recipient's signer address |

### Golang Example

```go
func SignTransfer(
    signerPrivateKey string,
    accountID int64,
    signerAddress string,
    coin Coin,
    metadata Global,
    receiverAccountID int64,
    receiverPublicKey string,
    amount decimal.Decimal,
    clientOrderID string,
) (string, error) {
    
    // Calculate amount with resolution
    resolution, _ := decimal.NewFromString(coin.StarkExResolution)
    amountScaled := amount.Mul(resolution).BigInt()
    
    // Calculate nonce and expiration
    nonce := CalcNonce(clientOrderID)
    expirationTimestamp := time.Now().Unix() + (30 * 24 * 60 * 60)
    
    // Build domain
    typedDomain, _ := internal.NewTypedDataDomain(
        "EdgeX", "1",
        metadata.NativeChainId,
        metadata.ContractAddress,
    )
    
    // Build typed data
    typedData := internal.TypedData{
        Types: internal.TypedDataTypes{
            "EIP712Domain": [...],
            "OrderBase":    [...],
            "TransferParams": []internal.TypedDataType{
                {Name: "base", Type: "OrderBase"},
                {Name: "amount", Type: "uint256"},
                {Name: "assetIdCollateral", Type: "uint64"},
                {Name: "receiverAccountId", Type: "uint64"},
                {Name: "receiverPublicKey", Type: "address"},
            },
        },
        PrimaryType: "TransferParams",
        Domain:      typedDomain,
        Message: internal.TypedDataMessage{
            "base": map[string]interface{}{
                "nonce":               strconv.FormatInt(nonce, 10),
                "signer":              signerAddress,
                "accountId":           strconv.FormatInt(accountID, 10),
                "expirationTimestamp": strconv.FormatInt(expirationTimestamp, 10),
            },
            "amount":              amountScaled.String(),
            "assetIdCollateral":   coin.CoinId,
            "receiverAccountId":   strconv.FormatInt(receiverAccountID, 10),
            "receiverPublicKey":   receiverPublicKey,
        },
    }
    
    return internal.SignTypedDataWithPrivateKey(signerPrivateKey, typedData)
}
```

---

## Helper Functions

### Derive Signer Address from Private Key

```go
import (
    "github.com/edgex-Tech/edgex-golang-sdk/sdk/internal"
)

func DeriveAddress(privateKeyHex string) (string, error) {
    return internal.DeriveAddressFromPrivateKey(privateKeyHex)
}

// Example usage
signerAddress, err := DeriveAddress("0x1234...")
// Returns: "0xAbC123..."
```

### Parse Resolution

```go
import "github.com/shopspring/decimal"

func ParseResolution(resolutionStr, starkExResolutionStr string) (decimal.Decimal, error) {
    // Priority: use starkExResolution if available
    if starkExResolutionStr != "" && starkExResolutionStr != "0" {
        return decimal.NewFromString(starkExResolutionStr)
    }
    
    // Fallback to resolution
    resolution, err := decimal.NewFromString(resolutionStr)
    if err != nil {
        return decimal.Zero, err
    }
    
    // Convert 10^N format to actual number
    // e.g., "10" means 10^10 = 10000000000
    exp := resolution.IntPart()
    return decimal.NewFromInt(10).Pow(decimal.NewFromInt(exp)), nil
}
```

---

## Signature Verification

### How EdgeX Verifies Signatures

1. **Reconstruct typed data** from request parameters
2. **Hash the typed data** using EIP-712 hashing algorithm
3. **Recover signer address** from signature
4. **Verify** recovered address matches the registered trading address for the account

### Manual Verification (Optional)

```go
import (
    "github.com/ethereum/go-ethereum/signer/core/apitypes"
    "github.com/ethereum/go-ethereum/crypto"
)

func VerifySignature(typedData apitypes.TypedData, signature string, expectedAddress string) bool {
    // Hash the typed data
    digest, _, err := apitypes.TypedDataAndHash(typedData)
    if err != nil {
        return false
    }
    
    // Decode signature (remove 0x prefix, decode hex)
    sigBytes, err := hex.DecodeString(signature[2:])
    if err != nil {
        return false
    }
    
    // Adjust v value (EIP-712 uses 27/28, ecrecover uses 0/1)
    if sigBytes[64] >= 27 {
        sigBytes[64] -= 27
    }
    
    // Recover public key
    pubKey, err := crypto.SigToPub(digest, sigBytes)
    if err != nil {
        return false
    }
    
    // Derive address and compare
    recoveredAddress := crypto.PubkeyToAddress(*pubKey).Hex()
    return strings.EqualFold(recoveredAddress, expectedAddress)
}
```

---

## Common Errors

### Error: Invalid L2 Signature

**Causes**:
1. Wrong Signer Key
2. Incorrect amount calculation (resolution not applied)
3. Wrong nonce calculation
4. Expired timestamp
5. Wrong domain parameters (chainId or contractAddress)

**Solutions**:
1. Verify Signer Key matches registered key
2. Check resolution is correctly applied to amounts
3. Ensure nonce is calculated from clientOrderId using SHA-256
4. Use current timestamp, not expired
5. Get domain from latest metadata

### Error: Signer Address Mismatch

**Cause**: The address derived from Signer Key doesn't match registered address

**Solution**:
```go
// Verify your key derives to correct address
derivedAddress, _ := internal.DeriveAddressFromPrivateKey(yourSignerKey)
fmt.Printf("Your signer address: %s\n", derivedAddress)

// This should match the address you registered on EdgeX platform
```

### Error: Amount Overflow

**Cause**: Amount × resolution exceeds maximum value

**Solution**: Use big.Int for all calculations:
```go
import "math/big"

amountBigInt := new(big.Int)
// Perform calculations using big.Int methods
```

---

## Migration from V1

### Key Differences: V1 vs V2

| Aspect | V1 | V2 |
|--------|----|----|
| **Algorithm** | Pedersen Hash + ECDSA | EIP-712 Typed Data |
| **Curve** | Legacy StarkEx curve | Standard secp256k1 |
| **Signature Format** | r, s, y | Standard Ethereum signature (0x...) |
| **Wallet Support** | Custom implementation | MetaMask compatible |
| **Hash Function** | Pedersen hash | Keccak256 (EIP-712) |

### Migration Checklist

- [ ] Replace StarkEx signature code with EIP-712 implementation
- [ ] Update signature format from `{r, s}` to `0x...` hex string
- [ ] Use standard Ethereum private keys (not StarkEx keys)
- [ ] Update amount calculations (check if resolutions changed)
- [ ] Test with V2 testnet before production

---

## Security Best Practices

1. **Protect Your Signer Key**
   - Store encrypted (e.g., AWS KMS, HashiCorp Vault)
   - Never log or expose in error messages
   - Rotate periodically

2. **Validate All Inputs**
   - Check amounts are within valid ranges
   - Verify addresses are valid Ethereum addresses
   - Ensure nonce is unique

3. **Use Official SDK**
   - Let SDK handle signature generation
   - Reduces risk of implementation bugs
   - Automatically stays updated

4. **Monitor Signature Usage**
   - Track orders signed by your key
   - Alert on unusual patterns
   - Implement rate limiting

---

## Additional Resources

- **EIP-712 Specification**: https://eips.ethereum.org/EIPS/eip-712
- **Go-Ethereum EIP-712 Implementation**: https://pkg.go.dev/github.com/ethereum/go-ethereum/signer/core/apitypes
- **EdgeX Python SDK**: https://github.com/edgex-Tech/edgex-python-sdk
- **EdgeX Golang SDK**: https://github.com/edgex-Tech/edgex-golang-sdk
- **EdgeX API Documentation**: https://edgex-1.gitbook.io/edgeX-documentation
