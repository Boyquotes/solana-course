# Summary

# Overview

# Lab

This lab will primarily focus on writing tests in typescript, but we will need to run a program locally against these tests. For this reason, we will need to go through a few steps to ensure we have a proper environment on your machine for the program to run. The on-chain program has already been written for you and is included in the lab starter code.

The program contains a few instructions that showcase what the CPI Guard can protect against. We will write tests invoking these instructionsboth with a CPI Guarded Token Account and a Token Account without a CPI Guard.

The tests have been broken up into specific files in the `/tests` directory. Each file serves as its own unit test that will invoke a specific instruction on our program.

The program has five instructions: `malicious_close_account`, `prohibited_approve_account`, `prohibited_set_authority`, `unauthorized_burn`, `set_owner`.

Each of these instructions makes a CPI to the `Token Extension Program` and attempts to take an action on the given token program that is potentially malicious unknowingly to the signer of the original transaction.

### 1. Verify Solana/Anchor/Rust Versions

We will be interacting with the `token22` program in this lab and that requires you have solana cli version ≥ 1.18.0. 

To check your version run:
```bash
solana --version
```

If the version printed out after running `solana --version` is less than `1.18.0` then you can update the cli version manually. Note, at the time of writing this, you cannot simply run the `solana-install update` command. This command will not update the CLI to the correct version for us, so we have to explicitly download version `1.18.0`. You can do so with the following command:

```bash
solana-install init 1.18.0
```

If you run into this error at any point attempting to build the program, that likely means you do not have the correct version of the solana CLI installed.

```bash
anchor build
error: package `solana-program v1.18.0` cannot be built because it requires rustc 1.72.0 or newer, while the currently active rustc version is 1.68.0-dev
Either upgrade to rustc 1.72.0 or newer, or use
cargo update -p solana-program@1.18.0 --precise ver
where `ver` is the latest version of `solana-program` supporting rustc 1.68.0-dev
```

You will also want the latest version of the anchor CLI installed. You can follow along the steps listed here to update via avm https://www.anchor-lang.com/docs/avm

or simply run
```bash
avm install latest
avm use latest
```

At the time of writing, the latest version of the Anchor CLI is `0.29.0`

Now, we can check our rust version.

```bash
rustc --version
```

At the time of writing, version `1.26.0` was used for the rust compiler. If you would like to update, you can do so via `rustup`
https://doc.rust-lang.org/book/ch01-01-installation.html

```bash
rustup update
```

Now, we should have all the correct versions installed.

### 2. Get starter code and add dependencies

Let's grab the starter branch.

```bash
git clone https://github.com/Unboxed-Software/token22-staking
cd token22-staking
git checkout starter
```

### 3. Update Program ID and Anchor Keypair

Once in the starter branch, run

`anchor keys list`

to get your program ID.

Copy and paste this program ID in the `Anchor.toml` file

```rust
// in Anchor.toml
[programs.localnet]
token_22_staking = "<YOUR-PROGRAM-ID-HERE>"
```

And in the `programs/token-22-staking/src/lib.rs` file.

```rust
declare_id!("<YOUR-PROGRAM-ID-HERE>");
```

Lastly set your developer keypair path in `Anchor.toml`.

```toml
[provider]
cluster = "Localnet"
wallet = "/YOUR/PATH/HERE/id.json"
```

If you don't know what your current keypair path is you can always run the solana cli to find out.

```bash
solana config get
```

### 4. Confirm the program builds

Let's build the starter code to confirm we have everything configured correctly. If it does not build, please revisit the steps above.

```bash
anchor build
```

You can safely ignore the warnings of the build script, these will go away as we add in the necessary code.

Feel free to run the provided tests to make sure the rest of the dev environment is setup correct. You'll have to install the node dependancies using `npm` or `yarn`. The tests should run, but they do not do anything currently.

```bash
yarn install
anchor test
```

### 5. Approve Delegate

The first CPI Guard we will test is the approve delegate functionality. The CPI Guard prevents approving a delegate of a token account with the CPI Guard enabled via CPI completely. It's important to note that you can approve a delegate on a CPI Guarded account, just not with a CPI. To do so, you must send an instruction directly to the `Token Extensions Program` from a client rather than via another program.

Before we write our test, we need to take a look at the program code we are testing. The `prohibited_approve_account` instruction is what we will be targeting here. 

```rust
pub fn prohibited_approve_account(ctx: Context<ApproveAccount>, amount: u64) -> Result<()> {
        msg!("Invoked ProhibitedApproveAccount");

        msg!("Approving delegate: {} to transfer up to {} tokens.", ctx.accounts.delegate.key(), amount);

        approve(ctx.accounts.approve_delegate_ctx(), amount)?;
        Ok(())
    }
    ...

#[derive(Accounts)]
pub struct ApproveAccount<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        mut,
        token::token_program = token_program,
        token::authority = authority
    )]
    pub token_account: InterfaceAccount<'info, token_interface::TokenAccount>,
    /// CHECK: delegat to approve
    #[account(mut)]
    pub delegate: AccountInfo<'info>,
    pub token_program: Interface<'info, token_interface::TokenInterface>,
}
...
impl <'info> ApproveAccount <'info> {
    pub fn approve_delegate_ctx(&self) -> CpiContext<'_, '_, '_, 'info, Approve<'info>> {
        let cpi_program = self.token_program.to_account_info();
        let cpi_accounts = Approve {
            to: self.token_account.to_account_info(),
            delegate: self.delegate.to_account_info(),
            authority: self.authority.to_account_info()
        };

        CpiContext::new(cpi_program, cpi_accounts)
    } 
}
```

If you are familiar with Solana programs, then this should look like a pretty simple instruction. The instruction expects an `authority` account as a `Signer` and a `token_account` that `authority` is the authority of.

The instruction then invokes the `Approve` instruction on the `Token Extensions Program` and attempts to assign `delegate` as the delegate over the given `token_account`.

Let's open the `/tests/approve-delegate-example.ts` file to begin testing this instruction.  We will need to initialize some test accounts.

```typescript
// test accounts
const payer = anchor.web3.Keypair.generate()
let testTokenMint: PublicKey = null
let userTokenAccount = anchor.web3.Keypair.generate()
let maliciousAccount = anchor.web3.Keypair.generate()
```
Then, we can move on to the "[CPI Guard] Approve Delegate Example" test. You'll notice that some test names have "[CPI Guard]" in the title. Those with this in the title are tests with a CPI Guarded token account and those without it use token accounts without the CPI Guard enabled. They invoke the same exact instruction in our target program, the only difference is whether or not they have the CPI Guard enabled or not. This is done to illustrate what the CPI Guard protects against versus an account without one.

To test our instruction, we first need to create our token mint and a token account with extensions.

```typescript
it("[CPI Guard] Approve Delegate Example", async () => {
    // airdrop tokens to test accounts to pay for transactions
    await safeAirdrop(payer.publicKey, provider.connection)
    await safeAirdrop(provider.wallet.publicKey, provider.connection)
    delay(10000)

    testTokenMint = await createMint(
        provider.connection,
        payer,
        provider.wallet.publicKey,
        undefined,
        6,
        undefined,
        undefined,
        TOKEN_2022_PROGRAM_ID
    )
    await createTokenAccountWithExtensions(
        provider.connection,
        testTokenMint,
        payer,
        payer,
        userTokenAccount
    )
})
```

We have created a Mint and a token account with the CPI Guard enabled. We can now send a transaction to our program so that it will attempt to invoke the Approve delegate instruction on the `Token Extensions Program`.

```typescript
// inside "[CPI Guard] Approve Delegate Example" test block
try {
    const tx = await program.methods.prohibitedApproveAccount(new anchor.BN(1000))
    .accounts({
    authority: payer.publicKey,
    tokenAccount: userTokenAccount.publicKey,
    delegate: maliciousAccount.publicKey,
    tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();

    console.log("Your transaction signature", tx);
} catch (e) {
    assert(e.message == "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2d")
    console.log("CPI Guard is enabled, and a program attempted to approve a delegate");
}
```

Notice we wrap this in a try/catch block. This is because this instruction should fail if the CPI Guard works correctly. We catch the error and assert that the error message is what we expect. 

Now, we essentially do the same thing for the "Approve Delegate Example" test, except we want to pass in a token account without a CPI Guard. To do this, we can simply disavle the CPI Guard on the `userTokenAccount` and resend the transaction.

```typescript
 it("Approve Delegate Example", async () => {
    await disableCpiGuard(
        provider.connection,
        payer,
        userTokenAccount.publicKey,
        payer,
        [],
        undefined,
        TOKEN_2022_PROGRAM_ID
    )

    await program.methods.prohibitedApproveAccount(new anchor.BN(1000))
        .accounts({
            authority: payer.publicKey,
            tokenAccount: userTokenAccount.publicKey,
            delegate: maliciousAccount.publicKey,
            tokenProgram: TOKEN_2022_PROGRAM_ID,
        })
        .signers([payer])
        .rpc();
})
```
This transaction will succeed and the `delegate` account will now have authority to transfer the given amount of tokens from the `userTokenAccount`.

Feel free to save your work and run `anchor test`. All of the tests will run, but these two are the only ones that are doing anything yet. They should both pass at this point.

### 6. Close Account

The close account instruction invokes the `close_account` instrucion on the `Token Extensions Program`. This simply will close the given token account, but you have the ability to define which account you'd like the lamports in this account used for rent can be transferred to. The CPI Guard ensures that this account is always the account owner.

```rust
pub fn malicious_close_account(ctx: Context<MaliciousCloseAccount>) -> Result<()> {
        msg!("Invoked MaliciousCloseAccount");

        msg!("Token account to close : {}", ctx.accounts.token_account.key());

        close_account(ctx.accounts.close_account_ctx())?;
        Ok(())
    }

#[derive(Accounts)]
pub struct MaliciousCloseAccount<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        mut,
        token::token_program = token_program,
        token::authority = authority
    )]
    pub token_account: InterfaceAccount<'info, token_interface::TokenAccount>,
    /// CHECK: malicious account
    #[account(mut)]
    pub destination: AccountInfo<'info>,
    pub token_program: Interface<'info, token_interface::TokenInterface>,
    pub system_program: Program<'info, System>,
}

impl<'info> MaliciousCloseAccount <'info> {
    // close_account for Token2022
    pub fn close_account_ctx(&self) -> CpiContext<'_, '_, '_, 'info, CloseAccount<'info>> {
        let cpi_program = self.token_program.to_account_info();
        let cpi_accounts = CloseAccount {
            account: self.token_account.to_account_info(),
            destination: self.destination.to_account_info(),
            authority: self.authority.to_account_info(),
        };

        CpiContext::new(cpi_program, cpi_accounts)
    }
}
```

The program just invokes the `close_account` instruction, but a potentially malicious client could pass in a different account than the token account owner as the `destination` account. This would be hard to see from a user's perspective unless the wallet notified them. With CPI Guards enabled, the `Token Extension Program` will simply reject the instruction if that is the case.

To test this, we will open up the `/tests/close-account-example.ts` file. First, we have to define our test accounts.

```typescript
// test accounts
const payer = anchor.web3.Keypair.generate()
let testTokenMint: PublicKey = null
let userTokenAccount = anchor.web3.Keypair.generate()
let maliciousAccount = anchor.web3.Keypair.generate()
```

Then, we can create our Mint and Token account with extensions.

```typescript
it("[CPI Guard] Close Account Example", async () => {
    await safeAirdrop(payer.publicKey, provider.connection)
    await safeAirdrop(provider.wallet.publicKey, provider.connection)
    delay(10000)

    testTokenMint = await createMint(
        provider.connection,
        payer,
        provider.wallet.publicKey,
        undefined,
        6,
        undefined,
        undefined,
        TOKEN_2022_PROGRAM_ID
    )
    await createTokenAccountWithExtensions(
        provider.connection,
        testTokenMint,
        payer,
        payer,
        userTokenAccount
    )
})
```

And finally, we can send a transaction to our `malicious_close_account` instruction. Since we have the CPI Guard enabled on this token account, the transaction should fail. Our test verifies it fails for the expected reason.

```typescript
// inside "[CPI Guard] Close Account Example" test block
try {
const tx = await program.methods.maliciousCloseAccount()
    .accounts({
        authority: payer.publicKey,
        tokenAccount: userTokenAccount.publicKey,
        destination: maliciousAccount.publicKey,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();

    console.log("Your transaction signature", tx);
} catch (e) {
    assert(e.message == "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2c")
    console.log("CPI Guard is enabled, and a program attempted to close an account without returning lamports to owner");
}
```

Now, we can disable the CPI Guard and send the same exact transaction in the "Close Account without CPI Guard" test. This transaction should succeed this time.

```typescript
it("Close Account without CPI Guard", async () => {
    await disableCpiGuard(
        provider.connection,
        payer,
        userTokenAccount.publicKey,
        payer,
        [],
        undefined,
        TOKEN_2022_PROGRAM_ID
    )

    await program.methods.maliciousCloseAccount()
        .accounts({
            authority: payer.publicKey,
            tokenAccount: userTokenAccount.publicKey,
            destination: maliciousAccount.publicKey,
            tokenProgram: TOKEN_2022_PROGRAM_ID,
        })
        .signers([payer])
        .rpc();
})
```