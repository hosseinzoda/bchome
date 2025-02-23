** Warning ** The scripts in this repository has not been tested, Published only for studying the concept.

# BCHOME

A proof-of-concept implementation of a BCH order matching engine.


# Summary

Set limit orders to buy or sell a token, When the matching engine finds a match it'll fill the orders with the ability to partially fill orders and once orders are fully filled it deposits the payout with instantly finality. Users have the ability to withdraw their order at anytime even if it has been partially filled.

The exchange uses two sorted linked lists to manage the bids & asks of the orderbook. Bids are sorted in descending order where the head of the list has the highest bid and the head of asks has the lowest price.

There are two nested sorting applied to both lists. First is the ask/bid price and the second is the order in which the entry has been included in the blockchain to achieve first-come-first-served exchange. Meaning if two bid orders are set at the same price, The matching engine first fills the order that has came first.

`add-entry.locking` & `remove-entry.locking` programs verify the entries are inserted and removed in the right place at the sorted lists, Such that the sorting rules of the list will remain valid at all times.

# The orderbook category

Upon the creation of the orderbook a new category should be created, Multiple types of utxos exist in this category, Such as `utxo_program`, `orderbook`, `bid-entry`, `ask-entry`.
A predefined script is assigned to each of the utxo types. Such that all utxos with this category are bound to the rules of the orderbook to maintain the validity of the data in stores in the blockchain.
The matching engine can only function as expected if all inputs with category are valid.

## Programs

The category has a utxo type for utxo programs. Which serve as an overriding program for entries & orderbook to enable more functions other than matching orders.

| Fields            | Description                                                                                                           |
|-------------------|-----------------------------------------------------------------------------------------------------------------------|
| UTXO Type & Flags | (1 byte) Highest four bits represent the type of utxo (type = `0b01000000`), The remaining are the flags. None Exists |
| Program ID        | (1 byte) The program id.                                                                                              |

### `add-entry` program parameters

| Fields             | Description                                                                   |
|--------------------|-------------------------------------------------------------------------------|
| min_token_amount   | (4 bytes) The order's minimum amount of tokens to exchange.                   |
| trade_fee_fraction | (4 bytes) A fraction representing the fee to pay when the order to be filled. |

## Orderbook UTXO

Orderbook utxo is used as the main utxo of the matching engine with some shared state with `bid-entry` & `ask-entry` utxos, The following is the functions of the current implementation it serves.

1. It collects fees when it matches orders.
2. It stores the `empty-list` state for bids & asks. These bits are used to maintain the validity of the list, add/remove entry functions require these bits.
3. It pays the txfee for matching transactions.
4. The owner of the orderbook can withdraw or set the orderbook's parameters, Like txfee_per_io

Orderbook commitment data structure.

| Fields            | Description                                                                                               |
|-------------------|-----------------------------------------------------------------------------------------------------------|
| UTXO Type & Flags | (1 byte) Highest four bits represent the type of utxo (type = `0b00010000`), The remaining are the flags, |
|                   | `0b00000001` is bids are empty flag, `0b00000010` is asks are empty flag.                                 |
| Token category    | (32 bytes) The category of the token to exchange.                                                         |
| txfee_per_io      | (4 bytes) The txfee to be paid for each input/output on the matching transction                           |


## Entry UTXO

Entries collectively ensure the validity of the asks/bids list, The matching engine uses the orderbook as the first io and then followed by a series of entries where they collectively ensure orders are filled & fees are paid.

Every entry has a carry utxo which holds the bch amount & token amount of the order. 

Entry's commitment data structure.

| Fields                   | Description                                                                                                                                                   |
|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| UTXO Type & Flags        | (1 byte) Highest four bits represent the type of utxo (`bid-entry = 0b00100000`, `ask-entry = 0b00110000`), The remaining are the flags of the current entry, |
|                          | There are two flags, `0b00000001` flag indicates that entry is at the head of the list and `0b00000010` flag is for the tail.                                 |
| Block or Price           | (4 bytes) The value is the price of the limit order, And it also represents a block in a sorted linked list.                                                  |
| Block index              | (4 bytes) The index of the entry at the current block. (`block + index`) is used as the unique entry's reference.                                             |
| Previous entry reference | (8 bytes) (`block + index`) of the previous entry.                                                                                                            |
| Carry's bch amount       | (8 bytes) The bch amount in the carry utxo.                                                                                                                   |
| Carry's token amount     | (8 bytes) The token amount in the carry utxo.                                                                                                                 |
| Fee                      | (4 bytes) A fraction used to calculate the exchange fee.                                                                                                      |

