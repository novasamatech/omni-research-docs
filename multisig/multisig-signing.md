# Introduction

This document describes what [multisig pallet](https://docs.rs/pallet-multisig/latest/pallet_multisig/#dispatchable-functions) API should be used for the multisig transaction signing, final signing and cancelling.

# Multisig signing process
## Multisig with quorum > 1
If Multisig account quorum is greater than 1 then the extrinsic call flow should be:
1. `multisig_approveAsMulti` or `multisig_asMulti` - both may be used with or without `callData` and without `timepoint`
2. `multisig_approveAsMulti` or `multisig_asMulti` - both may be used with or without `callData` and with `timepoint`
3. `multisig_asMulti` - for the final approval (if `quroum - number of singings = 1`) with `callData` and with `timepoint`

It's better to use the `multisig_approveAsMulti` extrinsic if possible because the fee is lower comparing to `multisig_asMulti`.
The weight of extrinsic may be checked in the [multisig pallet source code](https://github.com/paritytech/substrate/blob/master/frame/multisig/src/weights.rs)

## Multisig with quroum == 1
If Multisig account quorum equals to 1 then the extrinsic call flow should be:
1. `multisig_asMultiThreshold1`

The table below describes which extrinsics and when should be used:

| Case                          | Quorum | Extrinsic to be used                            | `callData` | `timepoint` |
|-------------------------------|--------|-------------------------------------------------|------------|-------------|
| 1st signing                   | `1`    | `multisig_asMultiThreshold1`                    | mandatory  | optional    |
| 1st signing                   | `>1`   | `multisig_approveAsMulti` or `multisig_asMulti` | optional   | optional    |
| not 1st and not final signing | `>1`   | `multisig_approveAsMulti` or `multisig_asMulti` | optional   | mandatory   |
| final signing                 | `>1`   | `multisig_asMulti`                              | mandatory  | mandatory   |


# Cancel multisig transaction
The Signatory which made a first signing is able to cancel the Multisig transaction.
For that purpose the `multisig_cancelAsMulti` - should be called with provided `timepoint` and `call_hash` should be used.


# Links
1. Polkadot js library API [multisig_asMultiThreshold1](https://polkadot.js.org/docs/substrate/extrinsics/#asmultithreshold1other_signatories-vecaccountid32-call-call), [multisig_cancelAsMulti](https://polkadot.js.org/docs/substrate/extrinsics/#cancelasmultithreshold-u16-other_signatories-vecaccountid32-timepoint-palletmultisigtimepoint-call_hash-u832), [multisig_asMulti](https://polkadot.js.org/docs/substrate/extrinsics/#asmultithreshold-u16-other_signatories-vecaccountid32-maybe_timepoint-optionpalletmultisigtimepoint-call-wrapperkeepopaquecall-store_call-bool-max_weight-u64), [multisig_approveAsMulti](https://polkadot.js.org/docs/substrate/extrinsics/#approveasmultithreshold-u16-other_signatories-vecaccountid32-maybe_timepoint-optionpalletmultisigtimepoint-call_hash-u832-max_weight-u64)
2. Multisig Substrate [pallet](https://github.com/paritytech/substrate/blob/master/frame/multisig/src/lib.rs)
