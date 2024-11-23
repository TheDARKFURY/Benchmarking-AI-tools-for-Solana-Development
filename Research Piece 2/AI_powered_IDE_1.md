Alright, let's break down how to build a Solana auction program using SPL tokens. I'll outline the core components and provide a code example for a basic implementation:

**1. Project Setup**
- Create a new Solana program project:
```bash
cargo new solana-auction --lib
```

**2. Program State**
We'll store auction details in an Auction struct:

```rust
use solana_program::{
   account_info::{next_account_info, AccountInfo},
   entrypoint,
   entrypoint::ProgramResult,
   program_error::ProgramError,
   pubkey::Pubkey,
   sysvar::{clock::Clock, rent::Rent},
};
use spl_token::{
   instruction::{approve, transfer, TransferParams},
   state::Account,
};
use std::convert::TryInto;
use std::mem::size_of;
use chrono::{DateTime, Utc};
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AuctionInstruction {
   /// Starts an auction
   InitAuction {
       /// The duration of the auction (in seconds)
       duration: u64,
       /// The price of the item being auctioned (in SPL tokens)
       start_price: u64,
       /// The token mint for the auction
       token_mint: Pubkey,
       /// The public key of the seller
       seller: Pubkey,
   },
   /// Places a bid on an auction
   Bid {
       /// The amount of SPL tokens being bid
       bid_amount: u64,
   },
   /// Ends an auction and transfers the tokens to the winner
   EndAuction,
}
pub struct Auction {
   /// The duration of the auction (in seconds)
   duration: u64,
   /// The start time of the auction
   start_time: DateTime<Utc>,
   /// The end time of the auction
   end_time: DateTime<Utc>,
   /// The current highest bid
   current_bid: u64,
   /// The token mint for the auction
   token_mint: Pubkey,
   /// The public key of the seller
   seller: Pubkey,
}
impl Auction {
   pub fn new(
       duration: u64,
       start_price: u64,
       token_mint: Pubkey,
       seller: Pubkey,
   ) -> Self {
       let clock = Clock::get().unwrap();
       let start_time = Utc::now();
       let end_time = start_time.checked_add_signed(chrono::Duration::seconds(duration)).unwrap();
       Self {
           duration,
           start_time,
           end_time,
           current_bid: start_price,
           token_mint,
           seller,
       }
   }
}
```

**3. Define Instructions**
Define the instructions for the auction program:
- **InitAuction:** Initialize a new auction.
- **Bid:** Place a bid on an auction.
- **EndAuction:** End an auction and transfer the tokens to the winner.

**4. Program Logic**
Implement the logic for each instruction:

```rust
pub fn process_instruction(
   program_id: &Pubkey,
   accounts: &[AccountInfo],
   instruction_data: &[u8],
) -> ProgramResult {
   let instruction = AuctionInstruction::unpack(instruction_data)?;
   match instruction {
       AuctionInstruction::InitAuction {
           duration,
           start_price,
           token_mint,
           seller,
       } => {
           msg!("Instruction: InitAuction");
           process_init_auction(
               program_id,
               accounts,
               duration,
               start_price,
               token_mint,
               seller,
           )
       }
       AuctionInstruction::Bid { bid_amount } => {
           msg!("Instruction: Bid");
           process_bid(program_id, accounts, bid_amount)
       }
       AuctionInstruction::EndAuction => {
           msg!("Instruction: EndAuction");
           process_end_auction(program_id, accounts)
       }
   }
}
```

**5. Helper Functions**
Implement helper functions for each instruction:

```rust
fn process_init_auction(
   program_id: &Pubkey,
   accounts: &[AccountInfo],
   duration: u64,
   start_price: u64,
   token_mint: Pubkey,
   seller: Pubkey,
) -> ProgramResult {
   let account_iter = &mut accounts.iter();
   let auction_account = next_account_info(account_iter)?;
   let initializer = next_account_info(account_iter)?;
   let token_account = next_account_info(account_iter)?;
   let system_program = next_account_info(account_iter)?;
   let rent = &Rent::from_account_info(next_account_info(account_iter)?)?;
   // Check that the initializer can send tokens
   if !initializer.is_signer {
       return Err(ProgramError::MissingRequiredSignature);
   }
   // Check that the auction account is owned by the program
   if *auction_account.owner != *program_id {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Check that the token account is owned by the token program
   if *token_account.owner != spl_token::id() {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Check that the rent is paid
   if !rent.is_exempt(auction_account.lamports(), auction_account.data_len()) {
       return Err(ProgramError::InsufficientFunds);
   }
   // Create the auction account
   let auction_state = Auction::new(duration, start_price, token_mint, seller);
   let auction_state_bytes = bincode::serialize(&auction_state)?;
   solana_program::program::invoke(
       &solana_program::system_instruction::create_account(
           initializer.key,
           auction_account.key,
           rent.minimum_balance(auction_state_bytes.len()),
           auction_state_bytes.len() as u64,
           program_id,
       ),
       &[initializer.clone(), auction_account.clone(), system_program.clone()],
   )?;
   // Store the auction state in the auction account
   let mut auction_account_data = auction_account.data.borrow_mut();
   auction_state_bytes.copy_to_slice(&mut auction_account_data[..]);
   Ok(())
}
fn process_bid(program_id: &Pubkey, accounts: &[AccountInfo], bid_amount: u64) -> ProgramResult {
   let account_iter = &mut accounts.iter();
   let auction_account = next_account_info(account_iter)?;
   let bidder = next_account_info(account_iter)?;
   let token_account = next_account_info(account_iter)?;
   let system_program = next_account_info(account_iter)?;
   // Check that the bidder can send tokens
   if !bidder.is_signer {
       return Err(ProgramError::MissingRequiredSignature);
   }
   // Check that the auction account is owned by the program
   if *auction_account.owner != *program_id {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Check that the token account is owned by the token program
   if *token_account.owner != spl_token::id() {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Get the auction state from the auction account
   let mut auction_state: Auction = bincode::deserialize(&auction_account.data.borrow())?;
   // Check that the bid is higher than the current highest bid
   if bid_amount <= auction_state.current_bid {
       return Err(ProgramError::InvalidInstructionData);
   }
   // Check that the auction is still active
   if Utc::now() >= auction_state.end_time {
       return Err(ProgramError::InvalidInstructionData);
   }
   // Transfer the bid amount from the bidder's token account to the auction's token account
   let transfer_params = TransferParams {
       source: token_account.key,
       destination: auction_account.key,
       owner: bidder.key,
       amount: bid_amount,
   };
   let transfer_ix = transfer(&spl_token::id(), &transfer_params)?;
   solana_program::program::invoke_signed(
       &transfer_ix,
       &[
           token_account.clone(),
           auction_account.clone(),
           bidder.clone(),
           system_program.clone(),
       ],
       &[&[
           bidder.key.as_ref(),
           token_account.key.as_ref(),
       ]],
   )?;
   // Update the auction state with the new bid
   auction_state.current_bid = bid_amount;
   let auction_state_bytes = bincode::serialize(&auction_state)?;
   let mut auction_account_data = auction_account.data.borrow_mut();
   auction_state_bytes.copy_to_slice(&mut auction_account_data[..]);
   Ok(())
}
fn process_end_auction(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
   let account_iter = &mut accounts.iter();
   let auction_account = next_account_info(account_iter)?;
   let seller = next_account_info(account_iter)?;
   let token_account = next_account_info(account_iter)?;
   let system_program = next_account_info(account_iter)?;
   // Check that the auction account is owned by the program
   if *auction_account.owner != *program_id {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Get the auction state from the auction account
   let mut auction_state: Auction = bincode::deserialize(&auction_account.data.borrow())?;
   // Check that the auction has ended
   if Utc::now() < auction_state.end_time {
       return Err(ProgramError::InvalidInstructionData);
   }
   // Transfer the current bid amount from the auction's token account to the seller's token account
   let transfer_params = TransferParams {
       source: auction_account.key,
       destination: seller.key,
       owner: program_id,
       amount: auction_state.current_bid,
   };
   let transfer_ix = transfer(&spl_token::id(), &transfer_params)?;
   solana_program::program::invoke_signed(
       &transfer_ix,
       &[
           auction_account.clone(),
           seller.clone(),
           auction_account.clone(),
           system_program.clone(),
       ],
       &[&[
           program_id.as_ref(),
           auction_account.key.as_ref(),
       ]],
   )?;
   Ok(())
}
```

**6. Instructions Unpacking**
Implement a function to unpack the instruction data:

```rust
impl AuctionInstruction {
   pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
       let (tag, rest) = input.split_first().ok_or(ProgramError::InvalidInstructionData)?;
       Ok(match tag {
           0 => {
               let (duration, rest) = rest.split_at(8);
               let duration = u64::from_le_bytes(duration.try_into().unwrap());
               let (start_price, rest) = rest.split_at(8);
               let start_price = u64::from_le_bytes(start_price.try_into().unwrap());
               let (token_mint, rest) = rest.split_at(32);
               let token_mint = Pubkey::new(&token_mint[..]);
               let (seller, _) = rest.split_at(32);
               let seller = Pubkey::new(&seller[..]);
               Self::InitAuction {
                   duration,
                   start_price,
                   token_mint,
                   seller,
               }
           }
           1 => {
               let (bid_amount, _) = rest.split_at(8);
               let bid_amount = u64::from_le_bytes(bid_amount.try_into().unwrap());
               Self::Bid { bid_amount }
           }
           2 => Self::EndAuction,
           _ => return Err(ProgramError::InvalidInstructionData),
       })
   }
}
```

**7. main.rs**
Update the main.rs file:

```rust
use solana_program::{
   account_info::{next_account_info, AccountInfo},
   entrypoint,
   entrypoint::ProgramResult,
   program_error::ProgramError,
   pubkey::Pubkey,
   sysvar::{clock::Clock, rent::Rent},
};
use spl_token::{
   instruction::{approve, transfer, TransferParams},
   state::Account,
};
use std::convert::TryInto;
use std::mem::size_of;
use chrono::{DateTime, Utc};
// Declare and export the program's entrypoint
entrypoint!(process_instruction);
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AuctionInstruction {
   /// Starts an auction
   InitAuction {
       /// The duration of the auction (in seconds)
       duration: u64,
       /// The price of the item being auctioned (in SPL tokens)
       start_price: u64,
       /// The token mint for the auction
       token_mint: Pubkey,
       /// The public key of the seller
       seller: Pubkey,
   },
   /// Places a bid on an auction
   Bid {
       /// The amount of SPL tokens being bid
       bid_amount: u64,
   },
   /// Ends an auction and transfers the tokens to the winner
   EndAuction,
}
pub struct Auction {
   /// The duration of the auction (in seconds)
   duration: u64,
   /// The start time of the auction
   start_time: DateTime<Utc>,
   /// The end time of the auction
   end_time: DateTime<Utc>,
   /// The current highest bid
   current_bid: u64,
   /// The token mint for the auction
   token_mint: Pubkey,
   /// The public key of the seller
   seller: Pubkey,
}
impl Auction {
   pub fn new(
       duration: u64,
       start_price: u64,
       token_mint: Pubkey,
       seller: Pubkey,
   ) -> Self {
       let clock = Clock::get().unwrap();
       let start_time = Utc::now();
       let end_time = start_time.checked_add_signed(chrono::Duration::seconds(duration)).unwrap();
       Self {
           duration,
           start_time,
           end_time,
           current_bid: start_price,
           token_mint,
           seller,
       }
   }
}
pub fn process_instruction(
   program_id: &Pubkey,
   accounts: &[AccountInfo],
   instruction_data: &[u8],
) -> ProgramResult {
   let instruction = AuctionInstruction::unpack(instruction_data)?;
   match instruction {
       AuctionInstruction::InitAuction {
           duration,
           start_price,
           token_mint,
           seller,
       } => {
           msg!("Instruction: InitAuction");
           process_init_auction(
               program_id,
               accounts,
               duration,
               start_price,
               token_mint,
               seller,
           )
       }
       AuctionInstruction::Bid { bid_amount } => {
           msg!("Instruction: Bid");
           process_bid(program_id, accounts, bid_amount)
       }
       AuctionInstruction::EndAuction => {
           msg!("Instruction: EndAuction");
           process_end_auction(program_id, accounts)
       }
   }
}
fn process_init_auction(
   program_id: &Pubkey,
   accounts: &[AccountInfo],
   duration: u64,
   start_price: u64,
   token_mint: Pubkey,
   seller: Pubkey,
) -> ProgramResult {
   let account_iter = &mut accounts.iter();
   let auction_account = next_account_info(account_iter)?;
   let initializer = next_account_info(account_iter)?;
   let token_account = next_account_info(account_iter)?;
   let system_program = next_account_info(account_iter)?;
   let rent = &Rent::from_account_info(next_account_info(account_iter)?)?;
   // Check that the initializer can send tokens
   if !initializer.is_signer {
       return Err(ProgramError::MissingRequiredSignature);
   }
   // Check that the auction account is owned by the program
   if *auction_account.owner != *program_id {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Check that the token account is owned by the token program
   if *token_account.owner != spl_token::id() {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Check that the rent is paid
   if !rent.is_exempt(auction_account.lamports(), auction_account.data_len()) {
       return Err(ProgramError::InsufficientFunds);
   }
   // Create the auction account
   let auction_state = Auction::new(duration, start_price, token_mint, seller);
   let auction_state_bytes = bincode::serialize(&auction_state)?;
   solana_program::program::invoke(
       &solana_program::system_instruction::create_account(
           initializer.key,
           auction_account.key,
           rent.minimum_balance(auction_state_bytes.len()),
           auction_state_bytes.len() as u64,
           program_id,
       ),
       &[initializer.clone(), auction_account.clone(), system_program.clone()],
   )?;
   // Store the auction state in the auction account
   let mut auction_account_data = auction_account.data.borrow_mut();
   auction_state_bytes.copy_to_slice(&mut auction_account_data[..]);
   Ok(())
}
fn process_bid(program_id: &Pubkey, accounts: &[AccountInfo], bid_amount: u64) -> ProgramResult {
   let account_iter = &mut accounts.iter();
   let auction_account = next_account_info(account_iter)?;
   let bidder = next_account_info(account_iter)?;
   let token_account = next_account_info(account_iter)?;
   let system_program = next_account_info(account_iter)?;
   // Check that the bidder can send tokens
   if !bidder.is_signer {
       return Err(ProgramError::MissingRequiredSignature);
   }
   // Check that the auction account is owned by the program
   if *auction_account.owner != *program_id {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Check that the token account is owned by the token program
   if *token_account.owner != spl_token::id() {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Get the auction state from the auction account
   let mut auction_state: Auction = bincode::deserialize(&auction_account.data.borrow())?;
   // Check that the bid is higher than the current highest bid
   if bid_amount <= auction_state.current_bid {
       return Err(ProgramError::InvalidInstructionData);
   }
   // Check that the auction is still active
   if Utc::now() >= auction_state.end_time {
       return Err(ProgramError::InvalidInstructionData);
   }
   // Transfer the bid amount from the bidder's token account to the auction's token account
   let transfer_params = TransferParams {
       source: token_account.key,
       destination: auction_account.key,
       owner: bidder.key,
       amount: bid_amount,
   };
   let transfer_ix = transfer(&spl_token::id(), &transfer_params)?;
   solana_program::program::invoke_signed(
       &transfer_ix,
       &[
           token_account.clone(),
           auction_account.clone(),
           bidder.clone(),
           system_program.clone(),
       ],
       &[&[
           bidder.key.as_ref(),
           token_account.key.as_ref(),
       ]],
   )?;
   // Update the auction state with the new bid
   auction_state.current_bid = bid_amount;
   let auction_state_bytes = bincode::serialize(&auction_state)?;
   let mut auction_account_data = auction_account.data.borrow_mut();
   auction_state_bytes.copy_to_slice(&mut auction_account_data[..]);
   Ok(())
}
fn process_end_auction(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
   let account_iter = &mut accounts.iter();
   let auction_account = next_account_info(account_iter)?;
   let seller = next_account_info(account_iter)?;
   let token_account = next_account_info(account_iter)?;
   let system_program = next_account_info(account_iter)?;
   // Check that the auction account is owned by the program
   if *auction_account.owner != *program_id {
       return Err(ProgramError::IncorrectProgramId);
   }
   // Get the auction state from the auction account
   let mut auction_state: Auction = bincode::deserialize(&auction_account.data.borrow())?;
   // Check that the auction has ended
   if Utc::now() < auction_state.end_time {
       return Err(ProgramError::InvalidInstructionData);
   }
   // Transfer the current bid amount from the auction's token account to the seller's token account
   let transfer_params = TransferParams {
       source: auction_account.key,
       destination: seller.key,
       owner: program_id,
       amount: auction_state.current_bid,
   };
   let transfer_ix = transfer(&spl_token::id(), &transfer_params)?;
   solana_program::program::invoke_signed(
       &transfer_ix,
       &[
           auction_account.clone(),
           seller.clone(),
           auction_account.clone(),
           system_program.clone(),
       ],
       &[&[
           program_id.as_ref(),
           auction_account.key.as_ref(),
       ]],
   )?;
   Ok(())
}
impl AuctionInstruction {
   pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
       let (tag, rest) = input.split_first().ok_or(ProgramError::InvalidInstructionData)?;
       Ok(match tag {
           0 => {
               let (duration, rest) = rest.split_at(8);
               let duration = u64::from_le_bytes(duration.try_into().unwrap());
               let (start_price, rest) = rest.split_at(8);
               let start_price = u64::from_le_bytes(start_price.try_into().unwrap());
               let (token_mint, rest) = rest.split_at(32);
               let token_mint = Pubkey::new(&token_mint[..]);
               let (seller, _) = rest.split_at(32);
               let seller = Pubkey::new(&seller[..]);
               Self::InitAuction {
                   duration,
                   start_price,
                   token_mint,
                   seller,
               }
           }
           1 => {
               let (bid_amount, _) = rest.split_at(8);
               let bid_amount = u64::from_le_bytes(bid_amount.try_into().unwrap());
               Self::Bid { bid_amount }
           }
           2 => Self::EndAuction,
           _ => return Err(ProgramError::InvalidInstructionData),
       })
   }
}
```

**8. Compile and Deploy**
- Compile the program:
```bash 
cargo build --release
```
- Deploy the program to a Solana network (devnet, testnet, or mainnet) using the CLI or a tool like Anchor.

## Additional Considerations:
- **Error Handling:** Add more robust error handling and validation.
- **Account Ownership:** You might want to use a PDA (Program Derived Address) for the auction account to ensure only the program can control it.
- **Auction End Time:** You'll need a mechanism to determine when the auction has ended. You could use a combination of the Clocksysvar and a timer.
- **Testing:** Thoroughly test the program on a devnet or testnet.

