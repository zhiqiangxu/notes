# Introduction

In solana, every program(or contract) state is stored within an account.

The account model is defined like this:

```rust
/// An Account with data that is stored on chain
/// This will be the in-memory representation of the 'Account' struct data.
/// The existing 'Account' structure cannot easily change due to downstream projects.
#[derive(PartialEq, Eq, Clone, Default, AbiExample, Deserialize)]
#[serde(from = "Account")]
pub struct AccountSharedData {
    /// lamports in the account
    lamports: u64,
    /// data held in this account
    data: Arc<Vec<u8>>,
    /// the program that owns this account. If executable, the program that loads this account.
    owner: Pubkey,
    /// this account's data contains a loaded program (and is now read-only)
    executable: bool,
    /// the epoch at which this account will next owe rent
    rent_epoch: Epoch,
}
```

The `lamports` field represents the balance amount this account has.

What interested me most is that it's possible to write a program that can manipulate the account balance directly like this:

```rust
/// Transfers lamports from one account (must be program owned)
/// to another account. The recipient can be any account
fn transfer_service_fee_lamports(
    from_account: &AccountInfo,
    to_account: &AccountInfo,
    amount_of_lamports: u64,
) -> ProgramResult {
    // Does the from account have enough lamports to transfer?
    if **from_account.try_borrow_lamports()? < amount_of_lamports {
        return Err(CustomError::InsufficientFundsForTransaction.into());
    }
    // Debit from_account and credit to_account
    **from_account.try_borrow_mut_lamports()? -= amount_of_lamports;
    **to_account.try_borrow_mut_lamports()? += amount_of_lamports;
    Ok(())
}
```
([source](https://docs.staging.trustedtrucks.com/zh/developers/cookbook/programs/transfer-sol))

> The fundamental rule is that your program can transfer lamports from
> any account owned by your program to any account at all.

How does it work?

# Principle (or implementation)

Solana implements this feature by:

1. Ensuring the `from_account` has been signed by the owner
2. Ensuring the amount of lamports before is the same as after the execution

The 2nd check is done inside `execute_loaded_transaction` [here](https://github.com/anza-xyz/agave/blob/0b17e575f2a6fc7f6e32ed916affdc5714b64d3c/svm/src/transaction_processor.rs#L952).

The 1st check is much harder to find, it's done after the SVM has executed an instruction within a transaction and upon deserializing the result it calls `serialization::deserialize_parameters` > `deserialize_parameters_aligned` > which then checks the returned buffer([`parameter_bytes`](https://github.com/anza-xyz/agave/blob/4439a5eab2ac46b689de50e23b3f57abd913e61c/programs/bpf_loader/src/lib.rs#L1610) in code, which is updated in-place during execution) with the previously saved lamports balance and then calls [`borrowed_account.set_lamports(lamports)?;`](https://github.com/anza-xyz/agave/blob/1c776667cb7f9e4f20891c08c256e82136829465/program-runtime/src/serialization.rs#L389) which ensures only [account owner](https://github.com/anza-xyz/agave/blob/219f805fcb4e46dd97d1fa3d8d75147c188ed57a/transaction-context/src/lib.rs#L851) can deduct from the lamports.