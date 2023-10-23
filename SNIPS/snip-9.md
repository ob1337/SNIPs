
---
snip: 9
title: SNIP STRK fee token
authors: Evyatar Oster <@evyataro>, Ohad Barta <ohad@starkware.co>
status: Draft
type: Standarts Track
category: Core
created: 2023-10-23
---

## Simple Summary
This SNIP outlines the semantics of transaction V3, and the protocol and API changes required to facilitate STRK as a fee token. Older transaction versions will continue to be supported, thus maintaining ETH as a fee token. It also specifies the first STRK <> ETH price feed to be used, a trusted off-chain oracle provided by [Pragma](https://www.pragmaoracle.com/). 

## Motivation

One of the primary purposes of STRK, the native token of Starknet, is to serve as the fee payment token on Starknet. 

## Specification

The newly introduced [V3 transaction](https://community.starknet.io/t/transaction-v3-snip/98228) will pay the fee in STRK (that is, the account must have a positive STRK balance). Older transactions are still supported and continue to pay with ETH. 

Note that in Starknet v0.13.0, we continue to charge a transaction only for its marginal verification cost on L1 (including calldata costs induced by state changes). To denominate the proposed fee in STRK, V3 transactions specify `max_price_per_unit` and `max_amount` of L1 gas. The estimate_fee API endpoint will return the same object for both V1 and V3 transactions. For V3 transactions the `gas_price` will be denominated in STRK.


Some of the features of V3 transactions will not yet be supported in v0.13.0, and the corresponding fields will be hardcoded to zero:

Volition-related fields:
    
&nbsp;&nbsp;&nbsp;&nbsp; `nonce_data_availability_mode: u32`
 
&nbsp;&nbsp;&nbsp;&nbsp;`fee_data_availability_mode:  u32`

Fee-market-related fields:

&nbsp;&nbsp;&nbsp;&nbsp; `max_amount` and `max_price_per_unit` For L2_GAS 

&nbsp;&nbsp;&nbsp;&nbsp;`tip: u64`

Other fields:

&nbsp;&nbsp;&nbsp;&nbsp; `paymaster_data: List[felt]`

&nbsp;&nbsp;&nbsp;&nbsp;`account_deployment_data: List[felt]`

## Feeder Gateway API Changes:
Two `gas_price` fields in a block:

`eth_l1_gas_price` - The gas price of the block, specified in units of Wei and encoded as a hex string. Used for transaction version prior to V3.

`strk_l1_gas_price` - The gas price of the block, specified in units of 10^-18 STRK (corresponding to the STRK ERC20 [decimals](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#ERC20-decimals) value), encoded as a hex string. Used for transaction version V3.
The feeder gateway will return all the fields of v3 transactions, including those which currently have no semantics and are not mandatory within the gateway. Note that this affects the get_block and get_transaction methods. The precise details appear in the [v3 transaction SNIP](https://github.com/EvyatarO/SNIPs/blob/snip-8/SNIPS/snip-8.md#specification)

## Json-RPC Spec Changes: 
You can look at the [following PR](https://github.com/starkware-libs/starknet-specs/pull/149/files) to the Starknet JSON-RPC spec to see the exact changes in the API.

## Protocol Changes: 
Fee computation:

The upcoming version will continue the approach we’ve taken to date, of setting the fee so that it attempts to cover L1 gas costs, disregarding congestion fees. Instead of `max_fee`, which is no longer present in V3, the maximum fee payable by the account will be calculated as `max_fee = max_amount * max_price_per_unit`,  and the actual_fee will be actual_fee=used_amount  * gas_price. Consequently, the transactions will be charged for the current `gas_price` and not `max_price_per_unit`.
Furthermore, during the validation stage, the Sequencer will confirm that `max_price_per_unit` is greater than or equal to the current `gas_price`. This ensures that users are not charged for transactions with high `gas_price`, enhancing cost-effectiveness.
In Transaction V3, STRK is utilized as the payment token, whereas older transaction versions continue to use ETH. The exchange rate will be applied by the Sequencer in a trusted manner and will not be integrated into the protocol.

## Appendix: STRK<>ETH Exchange Rate

Note: This serves as a temporary solution for the centralized case; the long-term decentralized protocol proposal will be published at a later date.

The Starknet Sequencer will use an off-chain STRK <> ETH price-feed, provided by [Pragma Oracle](https://www.pragmaoracle.com/). This feed will use a median of various Starknet and non-Starknet sources to derive the reported result. Note that the accuracy of the price-feed has limited influence on the Sequencer and the users, as detailed below. 

## Security Considerations:
Security Risk Exposure: The Sequencer is exposed to potential attacks on the exchange rate. However, since the Sequencer does not engage in STRK <> ETH trading as part of the protocol, the potential damage for the Sequencer is limited to the L1 cost of a block and not more. Potential damage for users is also limited as they can revert to submit V1 transactions and not be exposed to the price feed. 

Risk Mitigation: Launching such an attack on the Sequencer is costly. It would require a substantial investment to manipulate the actual STRK <> ETH ratio in the various sources which the oracle samples. Moreover, any artificial shifts in the STRK price are temporary, and would likely be remediated quickly.

## Copyright

Copyright and related rights waived via [MIT](../LICENSE).
