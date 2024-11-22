Certainly! I'll create a Solana program that implements the escrow system you've described. This program will use **Rust** and the Solana Program Library (SPL) for token operations. Let's break it down into parts and implement the functionality step by step.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};
use anchor_lang::solana_program::program::invoke_signed;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod solana_escrow {
    use super::*;

    pub fn create_offer(
        ctx: Context<CreateOffer>,
        offer_amount: u64,
        ask_amount: u64,
        bump: u8,
    ) -> Result<()> {
        let offer = &mut ctx.accounts.offer;
        offer.party_one = ctx.accounts.party_one.key();
        offer.party_two = ctx.accounts.party_two.key();
        offer.offer_token = ctx.accounts.offer_token.key();
        offer.ask_token = ctx.accounts.ask_token.key();
        offer.offer_amount = offer_amount;
        offer.ask_amount = ask_amount;
        offer.bump = bump;

        // Transfer tokens from party_one to temp_account
        let transfer_instruction = Transfer {
            from: ctx.accounts.party_one_send_account.to_account_info(),
            to: ctx.accounts.temp_account.to_account_info(),
            authority: ctx.accounts.party_one.to_account_info(),
        };

        let cpi_ctx = CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            transfer_instruction,
        );

        token::transfer(cpi_ctx, offer_amount)?;

        Ok(())
    }

    pub fn accept_offer(ctx: Context<AcceptOffer>) -> Result<()> {
        let offer = &ctx.accounts.offer;

        // Transfer ask_amount from party_two to party_one
        let transfer_to_party_one = Transfer {
            from: ctx.accounts.party_two_send_account.to_account_info(),
            to: ctx.accounts.party_one_receive_account.to_account_info(),
            authority: ctx.accounts.party_two.to_account_info(),
        };

        let cpi_ctx = CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            transfer_to_party_one,
        );

        token::transfer(cpi_ctx, offer.ask_amount)?;

        // Transfer offer_amount from temp_account to party_two
        let transfer_to_party_two = Transfer {
            from: ctx.accounts.temp_account.to_account_info(),
            to: ctx.accounts.party_two_receive_account.to_account_info(),
            authority: ctx.accounts.pda.to_account_info(),
        };

        let seeds = &[
            b"escrow".as_ref(),
            &offer.party_one.to_bytes(),
            &[offer.bump],
        ];
        let signer = &[&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            transfer_to_party_two,
            signer,
        );

        token::transfer(cpi_ctx, offer.offer_amount)?;

        // Close the temp_account
        let close_temp_account = token::CloseAccount {
            account: ctx.accounts.temp_account.to_account_info(),
            destination: ctx.accounts.party_one.to_account_info(),
            authority: ctx.accounts.pda.to_account_info(),
        };

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            close_temp_account,
            signer,
        );

        token::close_account(cpi_ctx)?;

        Ok(())
    }

    pub fn close_offer(ctx: Context<CloseOffer>) -> Result<()> {
        let offer = &ctx.accounts.offer;

        // Transfer offer_amount from temp_account back to party_one
        let transfer_back = Transfer {
            from: ctx.accounts.temp_account.to_account_info(),
            to: ctx.accounts.party_one_receive_account.to_account_info(),
            authority: ctx.accounts.pda.to_account_info(),
        };

        let seeds = &[
            b"escrow".as_ref(),
            &offer.party_one.to_bytes(),
            &[offer.bump],
        ];
        let signer = &[&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            transfer_back,
            signer,
        );

        token::transfer(cpi_ctx, offer.offer_amount)?;

        // Close the temp_account
        let close_temp_account = token::CloseAccount {
            account: ctx.accounts.temp_account.to_account_info(),
            destination: ctx.accounts.party_one.to_account_info(),
            authority: ctx.accounts.pda.to_account_info(),
        };

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            close_temp_account,
            signer,
        );

        token::close_account(cpi_ctx)?;

        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(offer_amount: u64, ask_amount: u64, bump: u8)]
pub struct CreateOffer<'info> {
    #[account(mut)]
    pub party_one: Signer<'info>,
    /// CHECK: This is not dangerous because we don't read or write from this account
    pub party_two: AccountInfo<'info>,
    pub offer_token: Account<'info, Mint>,
    pub ask_token: Account<'info, Mint>,
    #[account(
        mut,
        constraint = party_one_send_account.owner == party_one.key(),
        constraint = party_one_send_account.mint == offer_token.key(),
    )]
    pub party_one_send_account: Account<'info, TokenAccount>,
    #[account(
        init,
        payer = party_one,
        space = 8 + 32 + 32 + 32 + 32 + 8 + 8 + 1,
        seeds = [b"offer".as_ref(), party_one.key().as_ref()],
        bump
    )]
    pub offer: Account<'info, Offer>,
    #[account(
        init,
        payer = party_one,
        token::mint = offer_token,
        token::authority = pda,
    )]
    pub temp_account: Account<'info, TokenAccount>,
    /// CHECK: This is not dangerous because we don't read or write from this account
    #[account(
        seeds = [b"escrow".as_ref(), party_one.key().as_ref()],
        bump,
    )]
    pub pda: AccountInfo<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct AcceptOffer<'info> {
    /// CHECK: This is not dangerous because we don't read or write from this account
    pub party_one: AccountInfo<'info>,
    #[account(mut)]
    pub party_two: Signer<'info>,
    #[account(
        mut,
        constraint = offer.party_one == party_one.key(),
        constraint = offer.party_two == party_two.key(),
        close = party_one
    )]
    pub offer: Account<'info, Offer>,
    #[account(
        mut,
        constraint = party_one_receive_account.owner == party_one.key(),
        constraint = party_one_receive_account.mint == offer.ask_token,
    )]
    pub party_one_receive_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = party_two_send_account.owner == party_two.key(),
        constraint = party_two_send_account.mint == offer.ask_token,
    )]
    pub party_two_send_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = party_two_receive_account.owner == party_two.key(),
        constraint = party_two_receive_account.mint == offer.offer_token,
    )]
    pub party_two_receive_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = temp_account.mint == offer.offer_token,
        constraint = temp_account.owner == pda.key(),
    )]
    pub temp_account: Account<'info, TokenAccount>,
    /// CHECK: This is not dangerous because we don't read or write from this account
    #[account(
        seeds = [b"escrow".as_ref(), party_one.key().as_ref()],
        bump = offer.bump,
    )]
    pub pda: AccountInfo<'info>,
    pub token_program: Program<'info, Token>,
}

#[derive(Accounts)]
pub struct CloseOffer<'info> {
    #[account(mut)]
    pub party_one: Signer<'info>,
    #[account(
        mut,
        constraint = offer.party_one == party_one.key(),
        close = party_one
    )]
    pub offer: Account<'info, Offer>,
    #[account(
        mut,
        constraint = party_one_receive_account.owner == party_one.key(),
        constraint = party_one_receive_account.mint == offer.offer_token,
    )]
    pub party_one_receive_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = temp_account.mint == offer.offer_token,
        constraint = temp_account.owner == pda.key(),
    )]
    pub temp_account: Account<'info, TokenAccount>,
    /// CHECK: This is not dangerous because we don't read or write from this account
    #[account(
        seeds = [b"escrow".as_ref(), party_one.key().as_ref()],
        bump = offer.bump,
    )]
    pub pda: AccountInfo<'info>,
    pub token_program: Program<'info, Token>,
}

#[account]
pub struct Offer {
    pub party_one: Pubkey,
    pub party_two: Pubkey,
    pub offer_token: Pubkey,
    pub ask_token: Pubkey,
    pub offer_amount: u64,
    pub ask_amount: u64,
    pub bump: u8,
}

```

This Solana program implements the escrow system you described. Let me break down the main components and explain how they work:

**1. Program Structure:**
   - The program is defined using the Anchor framework, which simplifies Solana program development.
   - We have three main instructions: `create_offer`, `accept_offer`, and `close_offer`.

**2. Create Offer:**
   - The `create_offer` instruction initializes an `Offer` account and transfers the offer amount from party one to a temporary account.
   - It uses a PDA (Program Derived Address) as the authority for the temporary account.

**3. Accept Offer:**
   - The `accept_offer` instruction handles the token exchange between the two parties.
   - It transfers the ask amount from party two to party one, and the offer amount from the temporary account to party two.
   - After the exchange, it closes the temporary account.

**4. Close Offer:**
   - The `close_offer` instruction allows party one to cancel the offer and retrieve their tokens.
   - It transfers the tokens from the temporary account back to party one and closes the temporary account.

**5. Account Structures:**
   - `CreateOffer`, `AcceptOffer`, and `CloseOffer` structs define the accounts required for each instruction.
   - The `Offer` struct stores the details of each offer.

**6. Security Measures:**
   - The program uses PDAs to ensure that only the program can transfer tokens from the temporary account.
   - Various constraints are applied to the accounts to prevent unauthorized access or misuse.

This program provides a secure way for two parties to exchange tokens without trusting each other. The escrow mechanism ensures that the exchange happens atomically, preventing either party from backing out once the process has started.


