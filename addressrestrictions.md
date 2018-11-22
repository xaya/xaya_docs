# Address Restrictions

This document describes a proposed change to the network consensus rules
leading to a *soft fork* of the Xaya blockchain.  The new rules allow
placing **restrictions on receiving CHI on certain addresses**.  This
facilitates building in-game market places.

## Overview

Xaya games [can react to voluntary transfers of CHI](games.md#currency), but
they cannot force transfers.  This still allows an individual game to
include a market place for trading assets with CHI within its rules:
Players can list items by specifying a price and their address, and then
the game rules specify that the item is transferred to a buyer if the price
is paid to the given address.  This is fully decentralised and trustless,
based on atomic transactions.

There is one potential issue with this scheme, though:  If a sold item
is unique, it could happen that two prospective buyers send CHI at roughly
the same time, but only one of them can get the item in the game state.
The other one would have to be refunded, but that can only be done
voluntarily by the seller and not in a trustless way.  A potential solution
here is to introduce *reservations* of items:  When an item is for sale
on the in-game market place and Alice wants to buy it, she first sends a
transaction that does not yet transfer any CHI but signals her intent to
buy the item.  After confirmation, the item is reserved for her for a certain
time (e.g. 100 blocks).  During that time, Alice can then transfer the CHI
and be sure that she will get the item in exchange.  Any other prospective
buyer would see that the item is reserved for Alice, and thus won't send any
CHI that would have to be refunded.

Reservations work, but they are cumbersome to implement, require unnecessary
transactions in the blockchain and open up a venue for potential abuse and
DoS attacks on a game's market place.  Thus, we propose an alternative solution:

**The consensus rules of the Xaya network should allow sellers of items to
specify that they want to receive only a single CHI transaction at their
payment address.  All further payments will be invalid transactions.
This will allow games to implement simple market places without having
to rely on reservations (or refunds).**

Such optional *address restrictions* can be imposed with a soft fork of
the Xaya network.  Before going into the detailed specification below,
here is a brief overview of how those restrictions will work:

* The chain state of Xaya Core will keep track of active restrictions.
* Each restriction consists of a script (address) on which it is placed,
  an expiration block height and the actual restriction data (e.g. how many
  transactions are allowed or how much CHI can be sent in total).
* All restrictions are required to expire with a certain maximum
  time-to-live.  This allows to remove them from the chain state again
  in a timely manner.
* Restrictions are placed by updating a name with certain JSON in its
  value.  The restriction is put on *the address that held the name
  before the update*.  This ensures that *only the owner of an address can
  restrict it*.
* Address restrictions *do not apply to name transactions* sent to
  the address.  In other words, the locked 0.01 CHI in a name's
  "coloured coin" are not counted towards the limits.
* Name updates with JSON for an invalid restriction or transfers of CHI
  that violate an active restriction are invalid transactions and thus not
  allowed to be confirmed in blocks or put into the mempool of nodes.
