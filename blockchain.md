# Blockchain Consensus Protocol

This document gives an overview of the low-level consensus protocol that is
implemented by the core Xyon daemon.

**NOTE:**  The final reference of the protocol is the implementation of
Xyon Core.  This document just tries to highlight the most important parts
in an easy-to-read form!

## Xyon Is Based on Namecoin <a name="names"></a>

Most parts of the consensus protocol of Xyon are inherited from
[Namecoin](https://www.namecoin.org/), which itself inherits most of the
[Bitcoin protocol](https://bitcoin.org/).

Just like Bitcoin, Namecoin and Xyon implement a distributed ledger by
tracking the current set of "unspent transaction outputs".  But in addition
to pure currency transactions, they also implement a **name-value database**.
This database is stored and updated similarly to the UTXO set.  Each entry
contains the following fields:

* **Name:**
  A byte array with [restrictions](#name-value-restrictions) that is the
  "key" into the name-value-database.  In Xyon, this is typically the
  account name of a player.
* **Value:**
  Another, typically longer, byte array (also with
  [restrictions](#name-value-restrictions)) that holds data associated
  with the name.  In Xyon, this is used to store the latest moves
  or other actions the given player did.
* **Address:**
  Similar to transaction outputs, each name is associated to a Xyon address
  (Bitcoin script) that "owns" it.  Only the owner has *write access* to this
  name's entry, while everyone can *read* it from the database.  The owner
  can send the name to a different address, either owned by her as well
  or by someone else.

Each transaction can optionally, in addition to spending and creating currency
outputs, also **register** or **update** a name in this database.  A single
transaction can at most touch one name, and each name can per block be updated
at most once.

We do not give specific details for the transaction format here, since this
works the same as in Namecoin.

**In contrast to Namecoin, which registers names in a two-step process
(`name_new` and `name_firstupdate`), Xyon's name registrations are always
done by a single transaction (called `name_register`).  This transaction is
similar to a `name_update` in Namecoin except that it does not consume a name
input.**

## Basic Differences to Bitcoin

Some of the basic chain parameters and properties of the genesis block
are changed in Xyon in contrast to Bitcoin and Namecoin.  In particular:

* PoW mining is done based on the **Neoscrypt** algorithm instead of using
  double SHA-256.
* The difficulty is updated on each block using the **DGW formula**.
* Blocks are scheduled to be produced every **30 seconds** instead of
  10 minutes.
* The initial block reward is set to **1 CHI**, and halves every **4.2M**
  blocks.  (This corresponds to approximately four years, just as in Bitcoin.)
  * This is the initial schedule, which is planned to be updated after the
    token sale to produce the correct total supply once it is determined.
* The genesis block's coinbase transaction pays to a multisig address owned
  by the Xyon team.  Unlike Bitcoin and Namecoin, it is actually spendable,
  and does not observe the usual "block maturity" rule.
  * These coins will be distributed to the community according to the token
    sale and Huntercoin snapshot.  Unsold coins will be destroyed by sending
    them to a provably unspendable script (`OP_RETURN`).
* The maximum block weight is **400k** instead of 4M, corresponding to a block
  size of **100 KB** instead of 1 MB.  The maximum number of sigops in a block
  is **8k**.
* The maximum size of a script element is **2048 bytes** instead of 520 bytes,
  primarily to allow for larger name values.

## Activation of Soft Forks

Xyon activates some of the soft forks introduced in Bitcoin over time
immediately (from the genesis block), since we start with a fresh chain:

* [BIP 16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki)
* [BIP 34](https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki)
* [BIP 65](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki)
* [BIP 66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki)
* CSV ([BIP 68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki),
  [BIP 112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) and
  [BIP 113](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki))
* Segwit ([BIP
  141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki),
  [BIP 143](https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki) and
  [BIP 147](https://github.com/bitcoin/bips/blob/master/bip-0147.mediawiki))

(Note that some of these activate later or never on the **regtest network**
to allow for certain tests to be possible.)

## Name and Value Restrictions <a name="name-value-restrictions"></a>

Valid names and values in Xyon must satisfy additional constraints
compared to Namecoin (which just enforces maximum lengths).  In particular,
these conditions must be met by names and values:

* Names can be up to **256 bytes** long and values up to **2048 bytes**.
* Names must have a namespace.  This means that they must start with
  one or more lower-case letters and `/`.  Expressed as a regexp, this
  means they must match: `[a-z]+\/.*`
* Names must be valid UTF-8 and must not contain unprintable ASCII characters
  (with a value less than 0x20).
* Values must be valid [JSON](https://json.org/) and parse to a JSON **object**.
