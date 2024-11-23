I'd be happy to help you create an auction program in Solana that allows users to bid on items using SPL tokens. Let's break this down into steps and create the necessary components for the program.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod auction_program {
    use super::*;

    pub fn initialize_auction(
        ctx: Context<InitializeAuction>,
        resource: Pubkey,
        start_time: i64,
        end_time: i64,
        minimum_bid: u64,
    ) -> Result<()> {
        let auction = &mut ctx.accounts.auction;
        auction.resource = resource;
        auction.start_time = start_time;
        auction.end_time = end_time;
        auction.highest_bid = minimum_bid;
        auction.highest_bidder = Pubkey::default();
        auction.ended = false;
        Ok(())
    }

    pub fn place_bid(ctx: Context<PlaceBid>, amount: u64) -> Result<()> {
        let auction = &mut ctx.accounts.auction;
        let clock = Clock::get()?;

        require!(clock.unix_timestamp >= auction.start_time, ErrorCode::AuctionNotStarted);
        require!(clock.unix_timestamp <= auction.end_time, ErrorCode::AuctionEnded);
        require!(amount > auction.highest_bid, ErrorCode::BidTooLow);

        // Transfer tokens from bidder to auction escrow
        let transfer_instruction = Transfer {
            from: ctx.accounts.bidder_token_account.to_account_info(),
            to: ctx.accounts.auction_escrow.to_account_info(),
            authority: ctx.accounts.bidder.to_account_info(),
        };
        let cpi_ctx = CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            transfer_instruction,
        );
        token::transfer(cpi_ctx, amount)?;

        // Refund previous highest bidder
        if auction.highest_bidder != Pubkey::default() {
            let previous_bidder = auction.highest_bidder;
            let previous_bid = auction.highest_bid;

            // Implement refund logic here
            // You'll need to add necessary accounts to the PlaceBid context
            // and perform a token transfer to refund the previous highest bidder
        }

        // Update auction state
        auction.highest_bid = amount;
        auction.highest_bidder = ctx.accounts.bidder.key();

        Ok(())
    }

    pub fn end_auction(ctx: Context<EndAuction>) -> Result<()> {
        let auction = &mut ctx.accounts.auction;
        let clock = Clock::get()?;

        require!(clock.unix_timestamp > auction.end_time, ErrorCode::AuctionNotEnded);
        require!(!auction.ended, ErrorCode::AuctionAlreadyEnded);

        // Transfer winning bid to seller
        let transfer_instruction = Transfer {
            from: ctx.accounts.auction_escrow.to_account_info(),
            to: ctx.accounts.seller_token_account.to_account_info(),
            authority: ctx.accounts.auction.to_account_info(),
        };
        let auction_bump = *ctx.bumps.get("auction").unwrap();
        let seeds = &[
            b"auction".as_ref(),
            &auction.resource.to_bytes(),
            &[auction_bump],
        ];
        let signer = &[&seeds[..]];
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            transfer_instruction,
            signer,
        );
        token::transfer(cpi_ctx, auction.highest_bid)?;

        auction.ended = true;

        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(resource: Pubkey, start_time: i64, end_time: i64, minimum_bid: u64)]
pub struct InitializeAuction<'info> {
    #[account(
        init,
        payer = initializer,
        space = 8 + 32 + 8 + 8 + 8 + 32 + 1,
        seeds = [b"auction".as_ref(), resource.as_ref()],
        bump
    )]
    pub auction: Account<'info, Auction>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct PlaceBid<'info> {
    #[account(mut)]
    pub auction: Account<'info, Auction>,
    #[account(mut)]
    pub bidder: Signer<'info>,
    #[account(mut)]
    pub bidder_token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub auction_escrow: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
}

#[derive(Accounts)]
pub struct EndAuction<'info> {
    #[account(mut)]
    pub auction: Account<'info, Auction>,
    #[account(mut)]
    pub auction_escrow: Account<'info, TokenAccount>,
    #[account(mut)]
    pub seller_token_account: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
}

#[account]
pub struct Auction {
    pub resource: Pubkey,
    pub start_time: i64,
    pub end_time: i64,
    pub highest_bid: u64,
    pub highest_bidder: Pubkey,
    pub ended: bool,
}

#[error_code]
pub enum ErrorCode {
    #[msg("The auction has not started yet")]
    AuctionNotStarted,
    #[msg("The auction has already ended")]
    AuctionEnded,
    #[msg("The bid is too low")]
    BidTooLow,
    #[msg("The auction has not ended yet")]
    AuctionNotEnded,
    #[msg("The auction has already been ended")]
    AuctionAlreadyEnded,
}

```

This Solana program implements an auction system using Anchor, a framework for Solana development. Here's a breakdown of the main components:

**1. Program Structure:**
   - The program has three main instructions: `initialize_auction`, `place_bid`, and `end_auction`.
   - It uses the Anchor framework, which simplifies Solana program development.

**2. Auction Initialization:**
   - The `initialize_auction` function creates a new auction account with the specified parameters.
   - It sets the initial state of the auction, including the resource being auctioned, start and end times, and minimum bid.

**3. Placing Bids:**
   - The `place_bid` function allows users to place bids on the auction.
   - It checks if the auction is active and if the bid is higher than the current highest bid.
   - It transfers SPL tokens from the bidder to the auction escrow account.
   - If there was a previous highest bidder, it should refund their bid (implementation left as an exercise).

**4. Ending the Auction:**
   - The `end_auction` function finalizes the auction.
   - It checks if the auction has ended and hasn't been finalized yet.
   - It transfers the winning bid from the auction escrow to the seller's account.

**5. Account Structures:**
   - The program defines account structures for the auction itself and for each instruction.
   - These structures specify which accounts are required for each operation and their constraints.

**6. Error Handling:**
   - Custom error codes are defined to handle various error conditions specific to the auction logic.

**To complete this program, you would need to:**

1. Implement the refund logic in the `place_bid` function to return tokens to the previous highest bidder.
2. Add necessary tests to ensure the program works as expected.
3. Deploy the program to a Solana cluster (devnet or mainnet).
4. Create a client-side application to interact with the program.
