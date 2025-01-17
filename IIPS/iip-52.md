---
iip: 52
title: ICON BTP Arbitrary Call Service Standard
author: Jaechang Namgoong (@sink772)
discussions-to: https://github.com/icon-project/IIPs/issues/52
status: Draft
type: Standards Track
category: IRC
created: 2022-07-13
requires: https://github.com/icon-project/IIPs/issues/25
---

## Simple Summary

A standard interface to make an arbitrary call service (a.k.a. `xcall`) between different blockchain networks via
ICON BTP protocols.

## Abstract

This document describes a new [BSH](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-25.md#btp-service-handler) interface
that can be used to invoke arbitrary cross-chain contract calls between different blockchains networks that support ICON BTP.
A new `xcall` service handler name will be used for the arbitrary call service.
The existing interfaces between BMC and BSH ([`sendMessage`](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-25.md#sendmessage)
and [`handleBTPMessage`](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-25.md#handlebtpmessage))
will be utilized for sending and receiving the requested arbitrary call payloads.

## Motivation

Adding a new service to an existing BTP infrastructure requires an audit of the code to ensure that the service
does not have any malicious behavior, because the Relay bears the gas cost of executing the service transaction.
To improve scalability and flexibility and to overcome the restriction mentioned above, we propose a new interface
for invoking arbitrary cross-chain contract calls that can appropriately charge the gas cost to users triggering
cross-chain transactions.

## Specification

### Encoding of the Calldata Payload

Encoding and decoding of the calldata are up to the DApps implementation.
`xcall` only passes the raw byte streams with the predefined method calls.
This is simple and easy to implement from the viewpoint of the `xcall`, and also the result can be sent back
to the caller using the same `sendCallMessage` interface.

### Basic Interfaces

#### sendCallMessage

DApps need to invoke the following method of `xcall` on the source chain to send a call message to the destination chain.
The `_data` is DApp-specific payload.

```java
/**
 * Sends a call message to the contract on the destination chain.
 * Only allowed to be called from the contract.
 *
 * @param _to The BTP address of the callee on the destination chain
 * @param _data The calldata specific to the target contract
 * @param _rollback (Optional) The data for restoring the caller state when an error occurred
 * @return The serial number of the request
 */
@Payable
@External
BigInteger sendCallMessage(String _to, byte[] _data, @Optional byte[] _rollback);
```

The `_rollback` parameter is for handling error cases, see [Error Handling](#error-handling) section below for details.

This method is payable, and DApps need to call this method with proper fees, see [Fees Handling](#fees-handling) section below for details.

When `xcall` on the source chain receives the call request message, it sends the `_data` to the destination chain via BMC.

#### CallMessage

When the `xcall` on the destination chain receives the call request through BMC, it emits the following event for notifying the user.

```java
/**
 * Notifies the user that a new call message has arrived.
 *
 * @param _from The BTP address of the caller on the source chain
 * @param _to A string representation of the callee address
 * @param _sn The serial number of the request from the source
 * @param _reqId The request id of the destination chain
 * @param _data The calldata
 */
@EventLog(indexed=3)
void CallMessage(String _from, String _to, BigInteger _sn, BigInteger _reqId, byte[] _data);
```

#### executeCall

The user on the destination chain recognizes the call request and invokes the following method on `xcall` with the given `_reqId`.

```java
/**
 * Executes the requested call.
 *
 * @param _reqId The request Id
 */
@External
void executeCall(BigInteger _reqId);
```

#### handleCallMessage

When the user calls `executeCall` method, the `xcall` invokes the following predefined method in the target DApp
with the calldata associated in `_reqId`.

```java
/**
 * Handles the call message received from the source chain.
 * Only called from the Call Message Service.
 *
 * @param _from The BTP address of the caller on the source chain
 * @param _data The calldata delivered from the caller
 */
@External
void handleCallMessage(String _from, byte[] _data);
```

If DApp on the destination chain needs to send back the result (or error), it may call the same method interface
(i.e. `sendCallMessage`) to send the result message to the caller.
Then the user on the source chain would be notified via `CallMessage` event, and call `executeCall`,
then DApp may process the result in the `handleCallMessage` method.

### Error Handling

In success cases, DApp users only need to invoke two transactions (one for the source chain, and the other for
the destination chain).  However, there might be some error situations such as the execution of the call request
has failed on the destination chain. In this case, we need to notify the user on the source chain to rollback to the state
before the call request.

#### RollbackMessage

If a DApp needs to handle a rollback operation, it would fill some data in the last `_rollback` parameter of the `sendCallMessage`,
otherwise it would have a null value which indicates no rollback handling is required.
When an error occurs and the `_rollback` is not null, `xcall` emits the following event for notifying the user.

```java
/**
 * Notifies the user that a rollback operation is required for the request '_sn'.
 *
 * @param _sn The serial number of the previous request
 * @param _rollback The data for recovering that was given by the caller
 * @param _reason The error message that caused this rollback
 */
@EventLog(indexed=1)
void RollbackMessage(BigInteger _sn, byte[] _rollback, String _reason);
```

#### executeRollback

The user on the source chain recognizes the rollback situation and invokes the following method on `xcall` with the given `_sn`.
Note that the `executeRollback` can be called only when the original call request has responded with a failure.
It should be reverted when there is no failure response with the call request.

```java
/**
 * Rollbacks the caller state of the request '_sn'.
 *
 * @param _sn The serial number of the previous request
 */
@External
void executeRollback(BigInteger _sn);
```

Then the `xcall` invokes the `handleCallMessage` in the source DApp with the given `_rollback` data.
At this time, the `_from` would be the BTP address of `xcall`.

### Fees Handling

The RI for ICON network has adopted a simple fixed fees handling mechanism. If a user wants to make a call from
ICON to the T1 network, he needs to pay X ICX, and for the T2 network he needs to pay Y ICX.  That is, the fees
depend on the destination network address.  Also, a `default` fee has been defined in case of there is no fee
mapping to the network address.

The fees are divided into two types, one is for relays and the other is for protocol itself.
For example, for a destination chain T1, the fees could be relayFee = 0.25 ICX and protocolFee = 0.01 ICX.
And relayFee goes to relays, protocolFee goes to BTP protocol
(eventually, [Fee Aggregator SCORE](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-35.md)).
In this document, we don't address how to deal with these accrued fees for distribution.
For the fees distribution, we need to introduce another follow-up specification.

Here are getter and setter methods for the fixed fees handling. DApps that want to make a call to `sendCallMessage`,
should query the total fixed fees for the destination network via `totalFixedFees` interface, and then enclose the
appropriate fees in the method call.

```java
/**
 * Gets the fixed fee for the given network address and type.
 * If there is no mapping to the network address, `default` fee is returned.
 *
 * @param _net The network address
 * @param _type The fee type ("relay" or "protocol")
 * @return The fee amount in loop
 */
@External(readonly=true)
BigInteger fixedFee(String _net, String _type);

/**
 * Gets the total fixed fees for the given network address.
 * If there is no mapping to the network address, `default` fee is returned.
 *
 * @param _net The network address
 * @return The total fees amount in loop
 */
@External(readonly=true)
BigInteger totalFixedFees(String _net);

/**
 * Sets the fixed fees for the given network address.
 * Only the admin wallet can invoke this.
 *
 * @param _net The destination network address
 * @param _relay The relay fee amount in loop
 * @param _protocol The protocol fee amount in loop
 */
@External
void setFixedFees(String _net, BigInteger _relay, BigInteger _protocol);

/**
 * Gets the total accrued fees for the given type.
 *
 * @param _type The fee type ("relay" or "protocol")
 * @return The total accrued fees in loop
 */
@External(readonly=true)
BigInteger accruedFees(String _type);

/**
 * Notifies the user that the fees have been successfully updated.
 *
 * @param _net The destination network address
 * @param _relay The relay fee amount in loop
 * @param _protocol The protocol fee amount in loop
 */
@EventLog(indexed=1)
void FixedFeesUpdated(String _net, BigInteger _relay, BigInteger _protocol);
```

## Implementation
* [The RI of the `xcall` between ICON to ICON](https://github.com/icon-project/btp/tree/iconloop-v2/javascore/xcall)

## References
* [ICON BTP Standard](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-25.md)

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
