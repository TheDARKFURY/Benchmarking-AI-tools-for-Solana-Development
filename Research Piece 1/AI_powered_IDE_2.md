Certainly! I'll create a Solana program that implements the escrow system you've described. I'll provide the main structure and key functions of the program, along with explanations for each part.

```rust:src/lib.rs
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod token_escrow {
    use super::*;

    pub fn create_offer(
        ctx: Context<CreateOffer>,
        offer_amount: u64,
        ask_amount: u64,
    ) -> Result<()> {
        // Implementation for creating an offer
        // ...
    }

    pub fn accept_offer(ctx: Context<AcceptOffer>) -> Result<()> {
        // Implementation for accepting an offer
        // ...
    }

    pub fn close_offer(ctx: Context<CloseOffer>) -> Result<()> {
        // Implementation for closing an offer
        // ...
    }
}

#[derive(Accounts)]
pub struct CreateOffer<'info> {
    // Account structs for creating an offer
    // ...
}

#[derive(Accounts)]
pub struct AcceptOffer<'info> {
    // Account structs for accepting an offer
    // ...
}

#[derive(Accounts)]
pub struct CloseOffer<'info> {
    // Account structs for closing an offer
    // ...
}

#[account]
pub struct OfferDetails {
    // Offer details struct
    // ...
}
```

This is the basic structure of the Solana program. Now, let's implement each function and add the necessary account structures.

## 1. Create Offer:

```rust:src/lib.rs
// ... existing code ...

#[program]
pub mod token_escrow {
    use super::*;

    pub fn create_offer(
        ctx: Context<CreateOffer>,
        offer_amount: u64,
        ask_amount: u64,
    ) -> Result<()> {
        let offer_details = &mut ctx.accounts.offer_details;
        offer_details.party_one = ctx.accounts.party_one.key();
        offer_details.party_two = ctx.accounts.party_two.key();
        offer_details.offer_amount = offer_amount;
        offer_details.ask_amount = ask_amount;
        offer_details.offer_mint = ctx.accounts.offer_mint.key();
        offer_details.ask_mint = ctx.accounts.ask_mint.key();

        let (pda, _bump) = Pubkey::find_program_address(
            &[b"escrow", offer_details.to_account_info().key.as_ref()],
            ctx.program_id,
        );

        let transfer_instruction = Transfer {
            from: ctx.accounts.party_one_send_account.to_account_info(),
            to: ctx.accounts.temp_account.to_account_info(),
            authority: ctx.accounts.party_one.to_account_info(),
        };

        token::transfer(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                transfer_instruction,
            )
            .with_signer(&[&[
                b"escrow",
                offer_details.to_account_info().key.as_ref(),
                &[_bump],
            ]]),
            offer_amount,
        )?;

        Ok(())
    }

    // ... other functions ...
}

#[derive(Accounts)]
pub struct CreateOffer<'info> {
    #[account(mut)]
    pub party_one: Signer<'info>,
    /// CHECK: This is not dangerous because we don't read or write from this account
    pub party_two: UncheckedAccount<'info>,
    pub offer_mint: Account<'info, Mint>,
    pub ask_mint: Account<'info, Mint>,
    #[account(
        init,
        payer = party_one,
        space = 8 + OfferDetails::LEN,
        seeds = [b"offer_details", party_one.key().as_ref(), party_two.key().as_ref()],
        bump
    )]
    pub offer_details: Account<'info, OfferDetails>,
    #[account(
        mut,
        constraint = party_one_send_account.owner == party_one.key(),
        constraint = party_one_send_account.mint == offer_mint.key()
    )]
    pub party_one_send_account: Account<'info, TokenAccount>,
    #[account(
        init,
        payer = party_one,
        token::mint = offer_mint,
        token::authority = offer_details,
    )]
    pub temp_account: Account<'info, TokenAccount>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
}

#[account]
pub struct OfferDetails {
    pub party_one: Pubkey,
    pub party_two: Pubkey,
    pub offer_amount: u64,
    pub ask_amount: u64,
    pub offer_mint: Pubkey,
    pub ask_mint: Pubkey,
}

impl OfferDetails {
    pub const LEN: usize = 32 + 32 + 8 + 8 + 32 + 32;
}

// ... existing code ...
```

## 2. Accept Offer:

```rust:src/lib.rs
// ... existing code ...

#[program]
pub mod token_escrow {
    // ... existing functions ...

    pub fn accept_offer(ctx: Context<AcceptOffer>) -> Result<()> {
        let offer_details = &ctx.accounts.offer_details;

        // Transfer ask_amount from party_two to party_one
        let transfer_to_party_one = Transfer {
            from: ctx.accounts.party_two_send_account.to_account_info(),
            to: ctx.accounts.party_one_receive_account.to_account_info(),
            authority: ctx.accounts.party_two.to_account_info(),
        };

        token::transfer(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                transfer_to_party_one,
            ),
            offer_details.ask_amount,
        )?;

        // Transfer offer_amount from temp_account to party_two
        let (pda, bump) = Pubkey::find_program_address(
            &[b"escrow", offer_details.to_account_info().key.as_ref()],
            ctx.program_id,
        );

        let transfer_to_party_two = Transfer {
            from: ctx.accounts.temp_account.to_account_info(),
            to: ctx.accounts.party_two_receive_account.to_account_info(),
            authority: ctx.accounts.offer_details.to_account_info(),
        };

        token::transfer(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                transfer_to_party_two,
            )
            .with_signer(&[&[
                b"escrow",
                offer_details.to_account_info().key.as_ref(),
                &[bump],
            ]]),
            offer_details.offer_amount,
        )?;

        // Close the temp_account
        let close_temp_account = token::CloseAccount {
            account: ctx.accounts.temp_account.to_account_info(),
            destination: ctx.accounts.party_one.to_account_info(),
            authority: ctx.accounts.offer_details.to_account_info(),
        };

        token::close_account(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                close_temp_account,
            )
            .with_signer(&[&[
                b"escrow",
                offer_details.to_account_info().key.as_ref(),
                &[bump],
            ]]),
        )?;

        Ok(())
    }

    // ... other functions ...
}

#[derive(Accounts)]
pub struct AcceptOffer<'info> {
    #[account(mut)]
    pub party_two: Signer<'info>,
    /// CHECK: This is not dangerous because we don't read or write from this account
    #[account(mut)]
    pub party_one: UncheckedAccount<'info>,
    #[account(
        mut,
        constraint = offer_details.party_two == party_two.key(),
        close = party_one
    )]
    pub offer_details: Account<'info, OfferDetails>,
    #[account(
        mut,
        constraint = party_two_send_account.owner == party_two.key(),
        constraint = party_two_send_account.mint == offer_details.ask_mint
    )]
    pub party_two_send_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = party_two_receive_account.owner == party_two.key(),
        constraint = party_two_receive_account.mint == offer_details.offer_mint
    )]
    pub party_two_receive_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = party_one_receive_account.owner == party_one.key(),
        constraint = party_one_receive_account.mint == offer_details.ask_mint
    )]
    pub party_one_receive_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = temp_account.mint == offer_details.offer_mint,
        constraint = temp_account.owner == offer_details.key()
    )]
    pub temp_account: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
}

// ... existing code ...
```

## 3. Close Offer:

```rust:src/lib.rs
// ... existing code ...

#[program]
pub mod token_escrow {
    // ... existing functions ...

    pub fn close_offer(ctx: Context<CloseOffer>) -> Result<()> {
        let offer_details = &ctx.accounts.offer_details;
        let (pda, bump) = Pubkey::find_program_address(
            &[b"escrow", offer_details.to_account_info().key.as_ref()],
            ctx.program_id,
        );

        // Transfer tokens back to party_one
        let transfer_to_party_one = Transfer {
            from: ctx.accounts.temp_account.to_account_info(),
            to: ctx.accounts.party_one_receive_account.to_account_info(),
            authority: ctx.accounts.offer_details.to_account_info(),
        };

        token::transfer(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                transfer_to_party_one,
            )
            .with_signer(&[&[
                b"escrow",
                offer_details.to_account_info().key.as_ref(),
                &[bump],
            ]]),
            offer_details.offer_amount,
        )?;

        // Close the temp_account
        let close_temp_account = token::CloseAccount {
            account: ctx.accounts.temp_account.to_account_info(),
            destination: ctx.accounts.party_one.to_account_info(),
            authority: ctx.accounts.offer_details.to_account_info(),
        };

        token::close_account(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                close_temp_account,
            )
            .with_signer(&[&[
                b"escrow",
                offer_details.to_account_info().key.as_ref(),
                &[bump],
            ]]),
        )?;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct CloseOffer<'info> {
    #[account(mut)]
    pub party_one: Signer<'info>,
    #[account(
        mut,
        constraint = offer_details.party_one == party_one.key(),
        close = party_one
    )]
    pub offer_details: Account<'info, OfferDetails>,
    #[account(
        mut,
        constraint = party_one_receive_account.owner == party_one.key(),
        constraint = party_one_receive_account.mint == offer_details.offer_mint
    )]
    pub party_one_receive_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = temp_account.mint == offer_details.offer_mint,
        constraint = temp_account.owner == offer_details.key()
    )]
    pub temp_account: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
}

// ... existing code ...
```

This Solana program implements the escrow system for exchanging tokens between two parties as described. Here's a brief explanation of the key components:

1. `create_offer`: This function creates an offer by initializing the `OfferDetails` account and transferring the offer amount to a temporary account controlled by a PDA.

2. `accept_offer`: This function allows the second party to accept the offer. It transfers the ask amount from party two to party one and the offer amount from the temporary account to party two.

3. `close_offer`: This function allows the first party to close the offer if it hasn't been accepted. It transfers the tokens back from the temporary account to party one.

The program uses PDAs (Program Derived Addresses) to control the temporary account, ensuring that only the program can transfer tokens from it. This prevents either party from stealing the tokens before the offer is accepted or closed.

To use this program, you'll need to set up the necessary client-side code to interact with it, including creating and sending the appropriate transactions for each action.
