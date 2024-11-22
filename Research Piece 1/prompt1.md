# Solana Escrow Program

Create a Solana program that implements an escrow system for exchanging tokens between two parties. It allows a party to create an offer to exchange a specific amount of one token for another token. The other party can then accept the offer, and the program will transfer the tokens accordingly.

## The program's workflow is as follows:

## Create Offer:
Party one initiates an offer by providing the following information:
- The amount of tokens they are willing to send (offer_amount).
- The mint address of the token they are sending (offer_token).
- The amount of tokens they expect to receive in return (ask_amount).
- The mint address of the token they expect to receive (ask_token).
- The public key of the other party (party_two).
- The program creates an offer details account, which stores the offer information.
- The program transfers the offer_amount from party one's send_account to a temporary account (temp_account) controlled by a PDA (Program Derived Address).

## Accept Offer:
Party two accepts the offer by providing their public key and the necessary token accounts.
- The program verifies that the offer details account exists and that the provided token accounts are valid.
- The program transfers the ask_amount from party two's send_account to party one's receive_account.
- The program transfers the offer_amount from the temp_account to party two's receive_account.

## Close Offer:
Party one can close the offer if it is not accepted.
- The program transfers the offer_amount from the temp_account back to party one's receive_account.

The program uses a PDA to act as the authority for the temp_account, ensuring that only the program can transfer tokens from it. This prevents either party from stealing the tokens before the offer is accepted or closed.
