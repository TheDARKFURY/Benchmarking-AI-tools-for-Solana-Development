Here's a basic implementation of the Solana program using the **Anchor framework**. This code creates an escrow program that handles the described token exchange workflow.

## Prerequisites
To implement the program, you'll need the Anchor framework and Solana tool suite installed. You can follow the official Anchor documentation for setup instructions.

## Program Code
Create a new Anchor program:

```bash
anchor init solana_escrow
cd solana_escrow
```

**Replace lib.rs inside the programs/solana_escrow/src/ folder:**
    
```rust
    use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount, Transfer};

declare_id!("YourProgramIDHere...");

#[program]
pub mod solana_escrow {
    use super::*;

    // Create Offer
    pub fn create_offer(
        ctx: Context<CreateOffer>,
        offer_amount: u64,
        ask_amount: u64,
    ) -> Result<()> {
        let offer_details = &mut ctx.accounts.offer_details;
        offer_details.offer_amount = offer_amount;
        offer_details.ask_amount = ask_amount;
        offer_details.party_one = *ctx.accounts.party_one.key;
        offer_details.party_two = *ctx.accounts.party_two.key;
        offer_details.offer_token = ctx.accounts.offer_token_mint.key();
        offer_details.ask_token = ctx.accounts.ask_token_mint.key();

        // Transfer tokens from party one's account to the temporary PDA account
        let cpi_accounts = Transfer {
            from: ctx.accounts.party_one_send_account.to_account_info().clone(),
            to: ctx.accounts.temp_token_account.to_account_info().clone(),
            authority: ctx.accounts.party_one.to_account_info().clone(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        token::transfer(cpi_ctx, offer_amount)?;

        Ok(())
    }

    // Accept Offer
    pub fn accept_offer(ctx: Context<AcceptOffer>) -> Result<()> {
        let offer_details = &mut ctx.accounts.offer_details;

        // Ensure the offer is still valid and parties match
        require!(
            offer_details.party_two == *ctx.accounts.party_two.key,
            EscrowError::InvalidParty
        );

        // Transfer the ask tokens from party two to party one
        let cpi_accounts_ask = Transfer {
            from: ctx.accounts.party_two_send_account.to_account_info().clone(),
            to: ctx.accounts.party_one_receive_account.to_account_info().clone(),
            authority: ctx.accounts.party_two.to_account_info().clone(),
        };
        let cpi_program_ask = ctx.accounts.token_program.to_account_info();
        let cpi_ctx_ask = CpiContext::new(cpi_program_ask, cpi_accounts_ask);
        token::transfer(cpi_ctx_ask, offer_details.ask_amount)?;

        // Transfer the offered tokens from the temp account to party two's account
        let temp_seeds = &[b"temp".as_ref(), &[*ctx.bumps.get("temp_token_account").unwrap()]];
        let cpi_accounts_offer = Transfer {
            from: ctx.accounts.temp_token_account.to_account_info().clone(),
            to: ctx.accounts.party_two_receive_account.to_account_info().clone(),
            authority: ctx.accounts.temp_token_account.to_account_info().clone(),
        };
        let cpi_program_offer = ctx.accounts.token_program.to_account_info();
        let cpi_ctx_offer =
            CpiContext::new(cpi_program_offer, cpi_accounts_offer).with_signer(&[temp_seeds]);
        token::transfer(cpi_ctx_offer, offer_details.offer_amount)?;

        Ok(())
    }

    // Close Offer
    pub fn close_offer(ctx: Context<CloseOffer>) -> Result<()> {
        let offer_details = &ctx.accounts.offer_details;

        // Transfer tokens back to party one's account
        let temp_seeds = &[b"temp".as_ref(), &[*ctx.bumps.get("temp_token_account").unwrap()]];
        let cpi_accounts = Transfer {
            from: ctx.accounts.temp_token_account.to_account_info().clone(),
            to: ctx.accounts.party_one_receive_account.to_account_info().clone(),
            authority: ctx.accounts.temp_token_account.to_account_info().clone(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts).with_signer(&[temp_seeds]);
        token::transfer(cpi_ctx, offer_details.offer_amount)?;

        Ok(())
    }
}

// Context for Create Offer
#[derive(Accounts)]
#[instruction(offer_amount: u64, ask_amount: u64)]
pub struct CreateOffer<'info> {
    #[account(mut)]
    pub party_one: Signer<'info>,
    pub party_two: UncheckedAccount<'info>,
    #[account(mut)]
    pub party_one_send_account: Box<Account<'info, TokenAccount>>,
    #[account(init, payer = party_one, space = 8 + 64)]
    pub offer_details: Account<'info, OfferDetails>,
    #[account(
        init,
        seeds = [b"temp"],
        bump,
        payer = party_one,
        token::mint = offer_token_mint,
        token::authority = temp_token_account
    )]
    pub temp_token_account: Box<Account<'info, TokenAccount>>,
    pub offer_token_mint: Account<'info, Mint>,
    pub ask_token_mint: Account<'info, Mint>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

// Context for Accept Offer
#[derive(Accounts)]
pub struct AcceptOffer<'info> {
    #[account(mut)]
    pub party_two: Signer<'info>,
    #[account(mut)]
    pub party_two_send_account: Box<Account<'info, TokenAccount>>,
    #[account(mut)]
    pub party_two_receive_account: Box<Account<'info, TokenAccount>>,
    #[account(mut)]
    pub party_one_receive_account: Box<Account<'info, TokenAccount>>,
    #[account(
        mut,
        constraint = offer_details.party_one == *party_one.key,
        constraint = offer_details.party_two == *party_two.key
    )]
    pub offer_details: Account<'info, OfferDetails>,
    #[account(mut)]
    pub temp_token_account: Box<Account<'info, TokenAccount>>,
    pub token_program: Program<'info, Token>,
}

// Context for Close Offer
#[derive(Accounts)]
pub struct CloseOffer<'info> {
    #[account(mut)]
    pub party_one: Signer<'info>,
    #[account(mut)]
    pub party_one_receive_account: Box<Account<'info, TokenAccount>>,
    #[account(mut, close = party_one)]
    pub offer_details: Account<'info, OfferDetails>,
    #[account(mut)]
    pub temp_token_account: Box<Account<'info, TokenAccount>>,
    pub token_program: Program<'info, Token>,
}

// Offer Details struct
#[account]
pub struct OfferDetails {
    pub party_one: Pubkey,
    pub party_two: Pubkey,
    pub offer_amount: u64,
    pub ask_amount: u64,
    pub offer_token: Pubkey,
    pub ask_token: Pubkey,
}

// Custom error for the Escrow program
#[error_code]
pub enum EscrowError {
    #[msg("Invalid party provided.")]
    InvalidParty,
}
```

## Explanation
**1. Create Offer:**
- Initializes an offer by creating an `OfferDetails` account.
- Transfers the `offer_amount` of tokens from `party_one` to a temporary token account controlled by a PDA.
**2. Accept Offer:**
- Validates that `party_two` matches the original offer.
- Transfers `ask_amount` tokens from `party_two` to `party_one`.
- Transfers `offer_amount` tokens from the temporary PDA account to `party_two`.
**3. Close Offer:**
- Allows `party_one` to close the offer and reclaim their tokens if `party_two` has not accepted.

## Running the Program
**Build the program:**

```bash
anchor build
```

**Deploy the program:**

```bash
anchor deploy
```

This program should serve as a solid starting point. Make sure to test it thoroughly on a local or devnet environment before deploying to the mainnet.