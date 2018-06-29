<pre>
  CIP: 
  Title: Non Resendable Asset
  Authors: Cryptcoin Junkey
  Discussions-To: 
  Status: Draft
  Type: Standards
  Created: 2018-04-05
</pre>

## Abstract ##

Any user should be able to create assets that can be sent only by owner.

## Motivation ##

Enables to provide owner controlled assets that can send/collect only by the asset owner.

## Rationale ##

In some applications, a asset owner will want the asset under his/her control.

For example, promoters will want to use concert ticket as a asset. But they may not want to be sold tickets to another by customers.
membership card as a asset are better to controlled by the owner of each asset.
Owners of online games may want to manage equipment items in their games by using assets without any real-money-trade.

[NEM](https://nem.io/) blockchain supports similar assets as 'untransable mosaic'.

## Specification ##

There will be a extended issuance message that contains `resendable` flag.
The default value of `resendable` is True. 
An asset can set `resendable` to `False` only the first issuance.
But any assets issued with `resendable == False` can be set to True later.

an non resentable asset can't move 

## Changes ##

### Database 
 
1. Add `resentable` field into `assets` table:

```
TODO:
```

### Dividened

1. When validating a dividened attempt (dividened.validate):
   - Check if (`resendable` on `dividended_asset` is `True`) or (`dividended_asset` is owned by `source`).

### Issuance

1. When validating a issuance attempt (issuance.validate):
   - Check if modifying `resendable==True` not to `False`.

### Order

1. When validating a order attempt (order.validate):
   - Check if (`resenable` on `give_asset` is `True`) or (`give_asset` is owned by `source`).

### Sends

#### collect message

Defines new `collect` message.
On the protocol level, `collect` message have a same packet as `extend_send` except follows.

* ID=3
* `address` means the source address that have the collected asset.

When `resenable` of `asset` is `False` and the owner of `asset` is `destination`, `create_send` API returns the transaction that includes `collect` message.

#### validation

1. When validating a send attempt (send.validate):
   - In case normal send messages and `resendable==False`,
     - Check if sender is the owner of the asset.
   - In case `collect` message,
     - Check if the message sender is the owner of the asset.
     - Check if `quantiry` is less than `amount` in `address`.

## Copyright ##

This document is placed in the public domain.
