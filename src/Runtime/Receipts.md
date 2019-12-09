# Receipt

All cross-contract (we assume that each account lives in it's own shard) communication in Near happens trough Receipts.
Receipts are stateful in a sense that they serve not only as messages between accounts but also can be stored in the account storage to await other receipts.

Each `Receipt` has the following fields:

#### predecessor_id

- **`type`**: `AccountId`

The account_id which issued a receipt.

#### receiver_id

- **`type`**: `AccountId`

The destination account_id.

#### receipt_id

- **`type`**: `AccountId`

An unique id for the receipt.

#### type

- **`type`**: [ActionReceipt](#actionreceipt) | [DataReceipt](#datareceipt)

There is a 2 types of Receipts in Near: [ActionReceipt](#actionreceipt) and [DataReceipt](#datareceipt). ActionReceipt is a request to apply Actions, while DataReceipt is a result of application of these actions.

## ActionReceipt

`ActionReceipt` represents a request to apply actions on the `receiver_id` side. It could be a derived as a result of a `Transaction` execution or a another `ActionReceipt` processing. `ActionReceipt` consists the following fields:

#### signer_id

- **`type`**: `AccountId`

An account_id which signed the original [transaction](Transaction.md).

#### signer_public_key

- **`type`**: `PublicKey`

An [AccessKey](../Primitives/AccessKey.md) which was used to sign the original transaction.

#### gas_price

- **`type`**: `u128`

Gas price is a gas price which was set in a block where original [transaction](Transaction.md) has been applied.

#### output_data_receivers

- **`type`**: `[(CryptoHash, AccountId)]`

Output data receivers will be converted to `DataReceipt`s when smart contract finish its execution with some value.

#### input_data_ids

- **`type`**: `[CryptoHash]_`

`input_data_ids` are the receipt data dependencies. We These IDs are `DataReceipt.data_id`.

## DataReceipt

DataReceipt represents a final result of some contract execution.

#### data_id

- **`type`**: `CryptoHash`

An a unique DataReceipt identifier.

#### data

- **`type`**: `[u8]`

An an associated data in bytes.

# Receipt Matching

Runtime doesn't expect that Receipts are coming in a particular order. Each Receipt is processed individually. The goal of the `Receipt Matching` process is to match all [`ActionReceipt`s](#actionreceipt) to the corresponding [`DataReceipt`s](#datareceipt).

## Processing ActionReceipt

For each incoming [`ActionReceipt`](#actionreceipt) runtime checks whether we have all the [`DataReceipt`s](#datareceipt) (defined as [`ActionsReceipt.input_data_ids`](#input_data_ids)) required for execution. If all the required [`DataReceipt`s](#datareceipt) are already in the [storage](#received-datareceipt), runtime can apply this `ActionReceipt` immediately. Otherwise we save this receipt as a [Postponed ActionReceipt](#postponed-actionreceipt). Also we save [Pending DataReceipts Count](#pending-datareceipt-count) and [a link from pending `DataReceipt` to the `Postponed ActionReceipt`](#pending-datareceipt-for-postponed-actionreceipt). Now runtime will wait all the missing `DataReceipt`s to apply the `Postponed ActionReceipt`.

#### Postponed ActionReceipt

A Receipt which runtime stores until all the designated [`DataReceipt`s](#datareceipt) arrive.

- **`key`** = `account_id`,`receipt_id`
- **`value`** = `[u8]`

_Where `account_id` is [`Receipt.receiver_id`](#receiver_id), `receipt_id` is [`Receipt.receiver_id`](#receipt_id) and value is a serialized [`Receipt`](#receipt) (which [type](#type) must be [ActionReceipt](#actionreceipt))._

#### Pending DataReceipt Count

A counter which counts pending [`DataReceipt`s](#DataReceipt) for a [Postponed Receipt](#postponed-receipt) initially set to the length of missing [`input_data_ids`](#input_data_ids) of the incoming `ActionReceipt`. It's decrementing with every new received [`DataReceipt`](#datareceipt):

- **`key`** = `account_id`,`receipt_id`
- **`value`** = `u32`

_Where `account_id` is AccountId, `receipt_id` is CryptoHash and value is an integer._

#### Pending DataReceipt for Postponed ActionReceipt

We index each pending `DataReceipt` so when a new [`DataReceipt`](#datareceipt) arrives we can find to which [Postponed Receipt](#postponed-receipt) it belongs.

- **`key`** = `account_id`,`data_id`
- **`value`** = `receipt_id`

## Processing DataReceipt

#### Received DataReceipt

First of all, runtime saves the incoming `DataReceipt` to the storage as:

- **`key`** = `account_id`,`data_id`
- **`value`** = `[u8]`

_Where `account_id` is [`Receipt.receiver_id`](#receiver_id), `data_id` is [`DataReceipt.data_id`](#data_id) and value is a [`DataReceipt.data`](#data) (which is typically a serialized result of the call to a particular contract)._

Next, runtime checks if there is any [Postponed ActionReceipt](#postponed-actionreceipt) awaits for this `DataReceipt` by querying [`Pending DataReceipt` to the Postponed Receipt](#pending-datareceipt-for-postponed-actionReceipt). If there is no postponed `receipt_id` yet, we do nothing else. If there is a postponed `receipt_id`, we do the following:

- decrement [`Pending Data Count`](#pending-datareceipt-count) for the postponed `receipt_id`
- remove found [`Pending DataReceipt` to the `Postponed ActionReceipt`](#pending-datareceipt-for-postponed-actionreceipt)

If [`Pending DataReceipt Count`](#pending-datareceipt-count) is now 0 that means all the [`Receipt.input_data_ids`](#input_data_ids) are in storage and runtime can safely apply the [Postponed Receipt](#postponed-receipt) and remove it from the store.

## Case 1: ActionReceipt comes first, DataReceipts follows

Suppose runtime got the following `ActionReceipt` (we use a python-like pseudo code):

```python
# Non-relevant fields are omitted.
Receipt{
    receiver_id: "alice",
    receipt_id: "5e73d4"
    type: ActionReceipt {
        input_data_ids: ["e5fa44", "7448d8"]
    }
}
```

We can't apply this receipt right away: there are missing DataReceipt'a with IDs: ["e5fa44", "7448d8"]. Runtime does the following:

```python
postponed_receipts["alice,5e73d4"] = borsh_serialize(
    Receipt{
        receiver_id: "alice",
        receipt_id: "5e73d4"
        type: ActionReceipt {
            input_data_ids: ["e5fa44", "7448d8"]
        }
    }
)
pending_data_receipt_store["alice,e5fa44"] = "5e73d4"
pending_data_receipt_store["alice,7448d8"] = "5e73d4"
pending_data_receipt_count = 2
```

_Note: the subsequent Receipts could arrived in the current block or next, that's why we save [Postponed ActionReceipt](#postponed-actionreceipt) in the storage_

Then the first pending `Pending DataReceipt` arrives:

```python
# Non-relevant fields are omitted.
Receipt {
    receiver_id: "alice",
    type: DataReceipt {
        data_id: "e5fa44",
        data: "some data for alice",
    }
}
```

```python
data_receipts["alice,e5fa44"] = borsh_serialize(Receipt{
    receiver_id: "alice",
    type: DataReceipt {
        data_id: "e5fa44",
        data: "some data for alice",
    }
};
pending_data_receipt_count["alice,5e73d4"] = 1`
del pending_data_receipt_store["alice,e5fa44"]
```

And finally the last `Pending DataReceipt` arrives:

```python
# Non-relevant fields are omitted.
Receipt{
    receiver_id: "alice",
    type: DataReceipt {
        data_id: "7448d8",
        data: "some more data for alice",
    }
}
```

```python
data_receipts["alice,7448d8"] = borsh_serialize(Receipt{
    receiver_id: "alice",
    type: DataReceipt {
        data_id: "7448d8",
        data: "some more data for alice",
    }
};
postponed_receipt_id = pending_data_receipt_store["alice,5e73d4"]
postponed_receipt = postponed_receipts[postponed_receipt_id]
del postponed_receipts[postponed_receipt_id]
del pending_data_receipt_count["alice,5e73d4"]
del pending_data_receipt_store["alice,7448d8"]
apply_receipt(postponed_receipt)
```