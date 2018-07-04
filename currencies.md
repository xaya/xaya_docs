# Xyon Currencies

**Currencies** (custom tokens issued on the Xyon blockchain)
are a special case of more general games, and can thus be
implemented and managed through a Xyon [game state](games.md).

This document defines a standard for how such "games" may be created
with certain parameters, what the game rules are for them and what
moves can be made in the game to transact the tokens.
All currencies following this standard can then be supported out-of-the-box
by wallets or other infrastructure.
(This is very similar to how
[ERC-20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md)
standardised tokens on Ethereum.)

## Creation of a New Currency

To create a new currency, a corresponding `g/` name must be registered.
**In the registration**, the name's **initial** value must specify the new
game to be a currency according to this standard and set the basic parameters.
Wallets or other tools that wish to support general currencies must watch
the blockchain for registrations of `g/` names specifying valid parameters
to include them in their system.  (Or at least list them as available
currencies and give the user a choice to enable them.)

**All name updates other than the initial registration are ignored, so the
parameters must be set during registration of the name and cannot be changed
afterwards anymore!**

The initial value for a currency `g/` name must be a JSON object of the
following form:

    {
      "type": "currency",
      "version": 1,
      "creator": CREATOR,
      "supply": SUPPLY,
      "fixed": FIXED,
    }

The `version` field must currently be set to `1` to indicate that the
currency follows the first version of this standard.  In the future, additional
versions may be defined.
The meanings of the placeholders are:

* **`CREATOR`:**
  The player account (without the `p/` prefix) who is considered creator
  of the currency.
* **`SUPPLY`:**
  The total (initial) supply of tokens for this currency (must be an integer).
  They are initially all assigned to the currency's creator.
* **`FIXED`:**
  Boolean value that indicates whether the token supply is fixed with the
  creation or not.  If it is not fixed, then the creator is allowed to
  issue new tokens (see [below](#transactions)).

For example, this creates a currency with a fixed supply of **1,000** tokens
that are all initially assigned to `p/alice`:

    {
      "type": "currency",
      "version": 1,
      "creator": "alice",
      "supply": 1000,
      "fixed": true
    }

## Game State of a Currency

The current state of a currency (its "game state") is fully defined by
a map of player names to the amount of tokens they hold.

**All token amounts are integers.  They are interpreted to be the 10e8-th
part of the token's basic unit, similar to satoshis and bitcoins.  Wallets
and other user interfaces must convert between the integer amounts and the
displayed "token" amounts.**

## Transactions <a name="transactions"></a>

[Moves](games.md#moves) referencing the currency's game ID are used to
*transfer tokens to someone else*.  The sender of the transaction (from whose
balance the amount is deducted) is the account that sent the move by updating
its `p/` name.
Tokens can also be burnt (provably destroyed).  If the token supply is
not fixed (as specified by the game's parameters), then **the creator
of the currency** can also issue new tokens through a move.

The JSON value defining a currency transaction looks like this:

    {
      "s":
        {
          RECIPIENT1: AMOUNT1,
          RECIPIENT2: AMOUNT2,
          ...
        },
      "b": BURN-AMOUNT,
      "c": CREATE-AMOUNT,
    }

Each of these three fields is optional.  The placeholders' meanings are:

* **`RECIPIENT`n**:
  The player name (without `p/` prefix) to whom tokens are sent.
* **`AMOUNT`n**:
  The amount of tokens (integer) to send to `RECIPIENT`n.
* **`BURN-AMOUNT`**:
  The amount of tokens to burn (destroy irrecoverably and provably).
* **`CREATE-AMOUNT`**:
  The amount of tokens to create.  They are added to the balance of the
  user sending the move.

A move is only valid if *both* of the following conditions are fulfilled:

1. If `c` is set, then the game must be specified with `fixed: false`
   and the user sending the move must be the game's creator.
2. The user's balance must be large enough.  More specifically, the value
   of `AMOUNT1 + AMOUNT2 + ... + BURN-AMOUNT` must be less than or equal
   to the user's current balance (per the game state) plus `CREATE-AMOUNT`.

If at least one of these conditions is not true, then the **whole move is
invalid and does not affect the game state *at all***!

For example, if a currency `g/gold` is defined and `p/bob` has at least
ten gold units, then the following update of the name `p/bob` is
a valid transaction.  It burns two gold units,
sends five units to `p/alice` and three to `p/charly`:

    {
      "g":
        {
          "gold":
            {
              "b": 200000000,
              "s":
                {
                  "alice": 500000000,
                  "charly": 300000000
                }
            }
        }
    }

## Implementation Notes

Wallets and other software handling currencies are effectively
"game engines".  As such, they must not only handle moves from new blocks,
but they must also be able to [undo](games.md#undoing) blocks that are
detached.

Transactions following the [specification above](#transactions) can be
undone easily, since they already contain all the information necessary
to restore the previous state:

* Sent amounts are deducted from the recipients' balances and added to
  the balance of the sender of the move.
* Burnt amounts are added to the balance of the sender.
* Created amounts are deducted from the sender's balance.

**However, game engines must know whether or not a given transaction
was actually valid, and only undo the valid ones!**
Otherwise, they might "undo" the sending of a large amount that never belonged
to the player that sent the move.

Thus, the **undo data that implementations should keep for every block is the
list of transactions that were either valid or invalid**.
(Which one does not matter and might be chosen depending on which list is
shorter.  That will typically be the list of invalid moves, but it can also
be the other way round especially if a deliberate DoS attack is attempted.)
This list should ideally be stored using transaction IDs, although it
is also possible to use the list of sending names (since each name can
only be updated in a single transaction for each block).

There may be further optimisations possible to reduce the storage requirement,
but since even the full list of transaction IDs is already much smaller than the
block data itself, any larger effort is unlikely to be worthwhile.
