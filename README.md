# Hedgehog
An experiment in offline lightning payments

# Introduction

Hedgehog is a protocol for two party payment channels. Hedgehog channels are similar to lightning channels but with a few comparative benefits.

- Hedgehog channels are simpler than lightning channels
- State updates only require the sender to propose an update and the recipient to accept it
- The recipient can wait to accept a state change til they want to propose another one

The properties mentioned above allow for asynchronous payments. The user experience is similar to ecash protocols like cashu or fedimint, except with no server. If you have a channel with someone, you can -- without their assistance -- create a payment for them, embed it in a piece of text (think of it like a cheque), and send it to them via email or some other communication method. Then you can go offline. When they get online, they can either accept the state change (the cheque) and update their balance without your further assistance, or they can reject it. If they accept the state change (the cheque) they can even use their new balance to pay you back later by making another state change (another cheque) that builds on the previous state change (i.e. spends the cheque to make a new cheque). And they can send the new state change (the new cheque) to you even if you are still offline. Or, if they reject your state change, they can propose an alternative one and wait for you to accept that.

# How hedgehog works

Hedgehog is bult on a primitive in bitcoin script called "revocable connectors." To make revocable connectors, you need two even more primitive primitives: revocable scripts and connector outputs.

## First primitive: revocable scripts

A revocable script looks like this:

```
(Alice after 2016 blocks) OR (Bob && A_S) <-- this is revocable by Alice
or
(Bob after 2016 blocks) OR (Alice && B_S) <-- this is revocable by Bob
```

`A_S` is "Alice's Secret," a value Alice may reveal to Bob if she wants to revoke her ability to safely receive money in an address containing the top script. `B_S` is a similar secret Bob may reveal to Alice to revoke the bottom script.

## Second primitive: connector outputs

A connector is a primitive Buraq popularized by using it in Ark. Suppose you have a multisig address like this:

```
(Alice && Bob)
```

Bob can create a signature that sends money from that address to Alice, but, since signatures can commit to multiple inputs, he can make his signature conditional: the sig is only valid for a transaction that *also* consumes a second input. The second input is called a connector, because it "connects" utxo A to utxo B. Alice can only spend utxo B (a utxo locked to the above multisig) if utxo A exists (because it is used as an input to the transaction Bob signed).

# Combining them: hedgehog channels

With these two primitives in hand, suppose Alice opens a channel with Bob by sending 10_000 sats into a standard multisig `(Alice && Bob)`. Alice can send Bob 2_000 sats "off chain" -- while Bob is offline -- by sending him two signatures. The first signature is valid for a transaction that (a) creates a 330 sat dust utxo, revocable by Bob (i.e. the dust utxo is locked to a `revocable script`) and (b) sends the change (9_670 sats, minus a fee, so more like 9_500 sats) back to the multisig `(Alice && Bob)`. The second signature is valid for a transaction that sends 7_000 sats to Bob from the multisig `(Alice && Bob)` AND the utxo locked to the `revocable script.` (So this second transaction's inputs contain 9_500 + 330 sats together, and the utxo locked to the `revocable script` serves as a `revocable connector`.) This second transaction sends the change, 2_830 sats (minus a fee, so more like 2_500 sats) to Alice.

Alice’s new, off-chain balance: 2_500 sats (down ~2000 sats & a fee)
Bob’s new, off-chain balance: 7_000 sats (up ~2000 sats)
This is "State 2." (State 1 was just Alice: 10_000 sats, Bob: 0 sats)

When Bob eventually gets online, he can accept Alice’s disbursement (State 2) like this: cosign and broadcast the first transaction, which creates the revocable utxo; wait 2016 blocks for the timelock on the revocable utxo to expire (relative timelocks start "counting" as soon as the utxo exists); then cosign and broadcast the second transaction. But there's a better option: Bob can keep the signatures on hand and wait because he might want to send some money back to Alice later and there's no reason to "close this channel" yet.

# Bob’s turn

Suppose Bob goes for the latter option and 5 days later he decides he wants to send 3_000 sats to Alice. He can revoke the dust utxo by sending Alice `B_S` and, along with it, a signature that is valid for a transaction that sends the entire channel balance to Alice, but only if Alice discloses `B_S`. (So this transaction spends two inputs, 9_500 + 330, and creates one output for Alice, 9_830, but more like 9_500 due to fees, and it is only valid if Alice knows `B_S` -- which she now does -- and if Bob tries to close the channel in State 1.)

With this pair of data, Bob also sends Alice two other signatures: the first is valid for a transaction that spends from the original multisig `(Alice && Bob)`, the “on-chain” one that still has 10_000 sats in it, to (a) create a *different* revocable connector with 330 sats in it, revocable by *Alice* this time and (b) send the change (9_670 sats, minus a fee, so more like 9_500 sats) back to the multisig `(Alice && Bob)`. The second signature is valid for a transaction that sends 6_000 sats to Alice from the multisig AND the revocable utxo (so its inputs contain 9_500 + 330 sats together) and sends the change, 3_830 sats (minus a fee, so more like 3_500 sats) to Bob.

Alice’s new, off-chain balance: 6_000 sats (up ~3000 sats from her previous position)
Bob’s new, off-chain balance: 3_500 sats (down ~3000 sats & a fee from his previous position)
This is "State 3."

Now Alice is in the exact same position Bob was. Like Bob, she can accept Bob's disbursement (State 3) like this: cosign and broadcast the first transaction, which creates the revocable utxo; wait 2016 blocks for the timelock on the revocable utxo to expire; then cosign and broadcast the second transaction. Like Bob, she may also keep the signatures on hand and wait because she might want to send some money back to Bob later.

Importantly, Alice *cannot* broadcast her *original* transaction, create her *original* dust utxo, and spend it immediately using the revocation secret Bob gave her. She cannot do this because any expenditures from the multisig require two signatures, and Bob never cosigned her original transaction. So she literally is in the exact same position Bob was in, and she and Bob can just keep doing this back and forth forever if they want to, or they can close the channel whenever either party wants to.

# A potential problem, solved

A problem arises if, once a party sends money to their counterparty, they give up their ability to force close the channel, with no guarantee their counterparty will ever do so. After sending, a sender cannot broadcast the current state without their counterparty’s signature, who may not give it. A sender also cannot safely broadcast any previous state because they revoked all previous states and broadcasting a revoked state would let their counterparty take all their money. Therefore a sender, once they send, cannot safely do anything, and can lose their money forever. Here is how to fix this: instead of revoking the previous state *plain and simple,* the sender should revoke it *conditionally,* meaning they can try to broadcast their previous state, but their counterparty has a certain number of blocks to override it with the latest state instead if they come back online in time. In technical terms, instead of a revocable script looking like this:

```
(sender after 2016 blocks) OR (counterparty && revocation_secret)
```

It should look like this:

```
(sender after 2016 blocks) OR (counterparty && sender)
```

Thus, instead of sending a revocation secret to revoke a prior state, the sender merely sends a signature authorizing their counterparty to consume the revocable connector and the rest of the funds so long as both inputs are consumed in a transaction that creates the latest state. For even better safety, the old revocation script – (counterparty && revocation_secret) – should still be kept as a third tapleaf in the new revocation taptree, so that a sender can “conditionally” revoke their most recent state using (counterparty && sender) but “absolutely” revoke the state before that using the old revocation script. (That way they can only roll back to the state prior to their counterparty’s disappearance, while leaving their counterparty plenty of time to stop the rollback if they haven’t really disappeared.)

# Another potential problem, solved

In the section on the signatures created for hedgehog channels, I said, “The first signature is valid for a transaction that...sends the change...back to the multisig `(Alice && Bob)`.”

This could present a problem if a party broadcasts a transaction with that signature and then dies before broadcasting the followup transaction that is supposed to distribute the money according to the latest state. Once the funds are in the multisig, the remaining party would be unable to access them, due to their counterparty’s death and their own meager ability to produce only one signature out of two needed. To remedy this, the script for that output is modified to this:

```
(Alice && Bob) OR (Alice after 4032 blocks)
and, on the other hand,
(Alice && Bob) OR (Bob after 4032 blocks)
```

That way the party thought to have died may, if they did not die, still use their counterparty’s pregenerated signature, plus one of their own making, to distribute the funds according to the latest state, but if they did die, the money is not stuck forever. After 4032 blocks, their counterparty can take it.
