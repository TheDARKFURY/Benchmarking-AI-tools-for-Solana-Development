I'll create a basic structure for a Solana auction program that allows users to bid on items using SPL tokens. This program will include the main components you've described. I'll provide the code in **Rust**, which is the language used for Solana programs.

Here's a simplified version of the program:

```rust:src/lib.rs
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

declare_id!("Your_Program_ID_Here");

#[program]
pub mod auction_program {
    use super::*;

    pub fn create_auction(
        ctx: Context<CreateAuction>,
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
        auction.seller = *ctx.accounts.seller.key;
        Ok(())
    }

    pub fn place_bid(ctx: Context<PlaceBid>, amount: u64) -> Result<()> {
        let auction = &mut ctx.accounts.auction;
        let clock = Clock::get()?;

        require!(clock.unix_timestamp >= auction.start_time, ErrorCode::AuctionNotStarted);
        require!(clock.unix_timestamp < auction.end_time, ErrorCode::AuctionEnded);
        require!(amount > auction.highest_bid, ErrorCode::BidTooLow);

        // Transfer tokens from bidder to auction escrow
        let cpi_accounts = Transfer {
            from: ctx.accounts.bidder_token_account.to_account_info(),
            to: ctx.accounts.auction_escrow.to_account_info(),
            authority: ctx.accounts.bidder.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        token::transfer(cpi_ctx, amount)?;

        // Update auction state
        auction.highest_bid = amount;
        auction.highest_bidder = *ctx.accounts.bidder.key;

        Ok(())
    }

    pub fn end_auction(ctx: Context<EndAuction>) -> Result<()> {
        let auction = &ctx.accounts.auction;
        let clock = Clock::get()?;

        require!(clock.unix_timestamp >= auction.end_time, ErrorCode::AuctionNotEnded);

        // Transfer tokens from auction escrow to seller
        let cpi_accounts = Transfer {
            from: ctx.accounts.auction_escrow.to_account_info(),
            to: ctx.accounts.seller_token_account.to_account_info(),
            authority: ctx.accounts.auction.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        token::transfer(cpi_ctx, auction.highest_bid)?;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct CreateAuction<'info> {
    #[account(init, payer = seller, space = 8 + 32 + 8 + 8 + 8 + 32 + 32)]
    pub auction: Account<'info, Auction>,
    #[account(mut)]
    pub seller: Signer<'info>,
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
    pub seller: Signer<'info>,
    #[account(mut)]
    pub seller_token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub auction_escrow: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
}

#[account]
pub struct Auction {
    pub resource: Pubkey,
    pub start_time: i64,
    pub end_time: i64,
    pub highest_bid: u64,
    pub highest_bidder: Pubkey,
    pub seller: Pubkey,
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
}
```

This program provides the basic structure for an auction system on Solana using SPL tokens. Here's a brief explanation of the main components:

1. `create_auction`: Initializes a new auction with the given parameters.
2. `place_bid`: Allows users to place bids on an active auction.
3. `end_auction`: Finalizes the auction and transfers the winning bid to the seller.

The `Auction` struct holds the state of each auction, including the resource being auctioned, start and end times, current highest bid, and the highest bidder.

To use this program, you'll need to:

1. Set up the necessary Solana development environment.
2. Replace `"Your_Program_ID_Here"` with your actual program ID.
3. Implement client-side code to interact with this program.
4. Add more error handling and edge cases as needed.
5. Implement additional features like cancelling auctions, handling ties, etc.
