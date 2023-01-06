---
snip: 5
title: Event subscription on starknet nodes using JSON-RPC notifications
status: Draft
type: Standards Track
author: Matthieu Auger <matthieu.auger@gmail.com>
created: 2023-01-05
---

## Abstract

This SNIP proposes the addition of event subcription on StarkNet nodes, allowing clients to subscribe to blockchain events and receive notifications in real-time.

## Motivation

Currently, StarkNet clients must poll the node for updates, resulting in increased network traffic and slower response times. Event subscription offer a more efficient means of receiving event notifications, reducing the burden on both clients and nodes.

## Specification

We will add a new method, `starknet_subscribe`, to the StarkNet JSON-RPC specification. This method will allow clients to subscribe to specific events or sets of events, and receive notifications when those events occur.

To create a subscription, the client will use the `starknet_subscribe` method:

```json
{"id": 1, "method": "starknet_subscribe", "params": ["event_name"]}
```

The `event_name` parameter specifies the event or set of events to subscribe to.

The node will return a subscription ID:

```json
{"jsonrpc":"2.0","id":1,"result":"0x1"}
```

Once subscribed, the API will publish notifications using the `starknet_subscription` method, including the subscription ID:

```json
{"jsonrpc":"2.0","method":"starknet_subscription","params":{"subscription":"0x1","result":{"startingBlock":"0x0","currentBlock":"0x50","highestBlock":"0x343c19"}}}
```

Subscriptions are coupled with connections. If a connection is closed, all subscriptions created over the connection are removed.

### Supported subscriptions

* [New headers](#newHeads)
* [Logs](#logs)
* [Pending transactions](#pending-transactions)
* [Synchronizing](#synchronizing)

#### newHeads
Fires a notification each time a new header is appended to the chain, including chain reorganizations. Users can use the bloom filter to determine if the block contains logs that are interested to them.

In case of a chain reorganization the subscription will emit all new headers for the new chain. Therefore the subscription can emit multiple headers on the same height.

##### Example

```json
{"id": 1, "method": "starknet_subscribe", "params": ["newHeads"]}
```

```json
{
  "jsonrpc": "2.0",
  "method": "starknet_subscription",
  "params": {
    "result": {
      "difficulty": "0x15d9223a23aa",
      "extraData": "0xd983010305844765746887676f312e342e328777696e646f7773",
      "gasLimit": "0x47e7c4",
      "gasUsed": "0x38658",
      "logsBloom": "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
      "miner": "0xf8b483dba2c3b7176a3da549ad41a48bb3121069",
      "nonce": "0x084149998194cc5f",
      "number": "0x1348c9",
      "parentHash": "0x7736fab79e05dc611604d22470dadad26f56fe494421b5b333de816ce1f25701",
      "receiptRoot": "0x2fab35823ad00c7bb388595cb46652fe7886e00660a01e867824d3dceb1c8d36",
      "sha3Uncles": "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
      "stateRoot": "0xb3346685172db67de536d8765c43c31009d0eb3bd9c501c9be3229203f15f378",
      "timestamp": "0x56ffeff8",
      "transactionsRoot": "0x0167ffa60e3ebc0b080cdb95f7c0087dd6c0e61413140e39d94d3468d7c9689f"
    },
    "subscription": "0x9ce59a13059e417087c02d3236a0b1cc"
  }
}
```

## logs
Returns logs that are included in new imported blocks and match the given filter criteria.

In case of a chain reorganization previous sent logs that are on the old chain will be resend with the `removed` property set to true. Logs from transactions that ended up in the new chain are emitted. Therefore a subscription can emit logs for the same transaction multiple times.

### Parameters
1. `object` with the following (optional) fields
    - **address**, either an address or an array of addresses. Only logs that are created from these addresses are returned (optional)
    - **topics**, only logs which match the specified topics (optional)


### Example

```json
{"id": 1, "method": "starknet_subscribe", "params": ["logs", {"address": "0x8320fe7702b96808f7bbc0d4a888ed1468216cfd", "topics": ["0xd78a0cb8bb633d06981248b816e7bd33c2a35a6089241d099fa519e361cab902"]}]}
```

```json
{"jsonrpc":"2.0","method":"starknet_subscription","params": {"subscription":"0x4a8a4c0517381924f9838102c5a4dcb7","result":{"address":"0x8320fe7702b96808f7bbc0d4a888ed1468216cfd","blockHash":"0x61cdb2a09ab99abf791d474f20c2ea89bf8de2923a2d42bb49944c8c993cbf04","blockNumber":"0x29e87","data":"0x00000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000003","logIndex":"0x0","topics":["0xd78a0cb8bb633d06981248b816e7bd33c2a35a6089241d099fa519e361cab902"],"transactionHash":"0xe044554a0a55067caafd07f8020ab9f2af60bdfe337e395ecd84b4877a3d1ab4","transactionIndex":"0x0"}}}
```

## newPendingTransactions
Returns the hash for all transactions that are added to the pending state and are signed with a key that is available in the node.

When a transaction that was previously part of the canonical chain isn't part of the new canonical chain after a reogranization its again emitted.

### Parameters
none

### Example
```json
{"id": 1, "method": "starknet_subscribe", "params": ["newPendingTransactions"]}
```

```json
{
    "jsonrpc":"2.0",
    "method":"starknet_subscription",
    "params":{
        "subscription":"0xc3b33aa549fb9a60e95d21862596617c",
        "result":"0xd6fdc5cc41a9959e922f30cb772a9aef46f4daea279307bc5f7024edc4ccd7fa"
    }
}
```

## syncing
Indicates when the node starts or stops synchronizing. The result can either be a boolean indicating that the synchronization has started (true), finished (false) or an object with various progress indicators.

### Parameters
none

### Example

```json
{"id": 1, "method": "starknet_subscribe", "params": ["syncing"]}
```

```json
{"subscription":"0xe2ffeb2703bcf602d42922385829ce96","result":{"syncing":true,"status":{"startingBlock":674427,"currentBlock":67400,"highestBlock":674432,"pulledStates":0,"knownStates":0}}}}
```

## Rationale

The `starknet_subscribe` method is based on the `eth_subscribe` method used in the Besu JSON-RPC specification, which has proven to be a reliable and efficient way to subscribe to blockchain events. By following this established standard, we can ensure interoperability with other Ethereum-based clients.

## Backwards Compatibility

Both options introduce new methods to the StarkNet JSON-RPC specification, but do not introduce any backwards compatibility issues. Clients that do not support the `starknet_subscribe` or event-specific methods will simply ignore them and continue to function as before.

## Test Cases

Test cases for these options can be found in `assets/snip-XXX/tests.json`.

## Reference Implementation

Reference implementations for the `starknet_subscribe` and event-specific methods can be found in `assets/snip-XXX/starknet_subscribe.py`.

## Security Considerations

Websocket connections introduce the potential for malicious clients to flood the node with subscription requests, potentially causing a denial of service. To mitigate this risk, we will implement rate limiting on the number of subscriptions allowed per connection.

## Copyright Waiver

Copyright and related rights waived via [MIT](../LICENSE).
