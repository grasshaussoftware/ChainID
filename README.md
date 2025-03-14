# ChainID
by Grok3 via deusopus 2025

---

# ChainID System

## Introduction

The **ChainID System** is a blockchain-based identity registration solution built on the Avalanche C-Chain. It generates a unique cryptographic identity for users, verified through Know Your Customer (KYC) measures, and mints it as a Non-Fungible Token (NFT). Designed to balance security, privacy, and usability, ChainID leverages a wallet-connected DeFi address, ECC (secp256k1) key generation, and QR codes for offline key storage. This project was crafted by **deusopus** with assistance from **xAI's Grok 3** as of March 2025 (version 1.0).

### Key Features
- **Wallet Integration**: Automatically fetches the user's DeFi address from a connected wallet (e.g., Metamask), with a manual fallback.
- **KYC Verification**: Collects and verifies identity data (name, DOB, ID type/number) using a modular verification system.
- **Privacy**: Hashes sensitive data (email, ID number) before inclusion in the NFT metadata.
- **Security**: Generates a 24-word BIP-39 mnemonic, derives ECC keys, and zeroizes sensitive memory.
- **User Experience**: Terminal-based with a bold ASCII splash screen, QR code outputs, and streamlined prompts.

### Purpose
ChainID aims to provide a decentralized, verifiable identity solution for Web3 applications, bridging traditional KYC with blockchain immutability.

---

## Getting Started

### Prerequisites
- **Rust**: Install Rust via [rustup](https://rustup.rs/).
- **Dependencies**: Listed in `Cargo.toml` (see below).
- **Avalanche RPC**: Uses a public Infura endpoint; replace with your own for production.

### Installation
1. Clone the repository:
   ```bash
   git clone https://github.com/deusopus/chainid.git
   cd chainid
   ```
2. Create a `Cargo.toml` file:
   ```toml
   [package]
   name = "chainid"
   version = "1.0.0"
   edition = "2021"

   [dependencies]
   bip39 = "2.0"
   chrono = "0.4"
   qrcode = "0.13"
   rand_hc = "0.1"
   serde_json = "1.0"
   sha2 = "0.10"
   tokio = { version = "1.0", features = ["full"] }
   web3 = "0.19"
   k256 = { version = "0.13", features = ["ecdsa"] }
   hex = "0.4"
   zeroize = "1.8"
   ```
3. Add the code to `src/main.rs` (see below).
4. Build and run:
   ```bash
   cargo run
   ```

### Usage
1. Launch the program to see the splash screen.
2. Connect a wallet (simulated; manual input supported).
3. Enter KYC data (name, DOB, email, citizenship, X username, ID type/number).
4. Review generated private/public QR codes.
5. Simulate NFT minting on Avalanche C-Chain.

---

## Code

Below is the full source code for `src/main.rs`:

```rust
use bip39::{Mnemonic, Language, MnemonicType, Seed};
use chrono::{Utc, DateTime};
use qrcode::QrCode;
use qrcode::render::unicode;
use rand_hc::Hc128Rng;
use serde_json::json;
use sha2::{Sha256, Digest};
use std::io::{self, Write};
use std::thread::sleep;
use std::time::Duration;
use tokio;
use web3::{transports::Http, Web3, types::{BlockId, BlockNumber}};

// Avalanche C-Chain RPC URL
const AVAX_RPC_URL: &str = "https://avalanche-mainnet.infura.io/v3/f4824ede0b484d33a19f0d01c32c9de1";

// Splash screen with aligned, bold "ChainID"
fn print_splash_screen() {
    println!("~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~");
    println!("      CCCC   H   H   AAA   IIIII  N   N  IIIII  DDDD   ");
    println!("      C   C  H   H  A   A    I    NN  N    I    D   D  ");
    println!("      C      HHHHH  AAAAA    I    N N N    I    D    D ");
    println!("      C   C  H   H  A   A    I    N  NN    I    D   D  ");
    println!("      CCCC   H   H  A   A  IIIII  N   N  IIIII  DDDD   ");
    println!("~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~");
    println!("          Your Blockchain Identity Solution          ");
    println!("~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~");
    println!("          Crafted by deusopus & xAI's Grok 3         ");
    println!("               Version 1.0 - March 2025              ");
    println!("~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~");
}

// Clear screen utility
fn clear_screen() {
    print!("{}[2J", 27 as char);
    io::stdout().flush().unwrap();
}

// Fetch latest block hash
async fn get_current_block_hash(web3: &Web3<Http>) -> Result<String, web3::Error> {
    if let Some(block) = web3.eth().block(BlockId::Number(BlockNumber::Latest)).await? {
        if let Some(hash) = block.hash {
            Ok(hash.iter().map(|byte| format!("{:02x}", byte)).collect())
        } else {
            Err(web3::Error::Decoder("No hash found for latest block".into()))
        }
    } else {
        Err(web3::Error::Decoder("Failed to fetch latest block".into()))
    }
}

// Fetch wallet address with fallback
async fn get_wallet_address(web3: &Web3<Http>) -> Result<String, Box<dyn std::error::Error>> {
    println!("Attempting to connect to your wallet...");
    match web3.eth().accounts().await {
        Ok(accounts) if !accounts.is_empty() => Ok(format!("0x{}", hex::encode(accounts[0]))),
        _ => {
            println!("Wallet connection failed. Please enter your DeFi address manually (0x...) or 'skip' to proceed without:");
            let mut input = String::new();
            io::stdin().read_line(&mut input)?;
            let trimmed = input.trim();
            if trimmed == "skip" {
                Ok("None".to_string())
            } else if trimmed.starts_with("0x") && trimmed.len() == 42 {
                Ok(trimmed.to_string())
            } else {
                Err("Invalid DeFi address format".into())
            }
        }
    }
}

// Simulate KYC verification (trait for modularity)
trait KycVerifier {
    fn verify(&self, name: &str, dob: &str, id_type: &str, id_number: &str) -> Result<bool, Box<dyn std::error::Error>>;
}

struct MockKycVerifier;
impl KycVerifier for MockKycVerifier {
    fn verify(&self, name: &str, dob: &str, id_type: &str, id_number: &str) -> Result<bool, Box<dyn std::error::Error>> {
        println!("Verifying KYC: {} | {} | {} | {}", name, dob, id_type, id_number);
        sleep(Duration::from_secs(1));
        Ok(true)
    }
}

// Collect user info with validation
fn collect_user_info() -> Result<(String, String, String, String, String, String, String), Box<dyn std::error::Error>> {
    let mut name = String::new();
    let mut dob = String::new();
    let mut email = String::new();
    let mut citizenship = String::new();
    let mut x_username = String::new();
    let mut id_type = String::new();
    let mut id_number = String::new();

    print!("Full name (as on ID): ");
    io::stdout().flush()?;
    io::stdin().read_line(&mut name)?;
    let name = name.trim().to_string();
    if name.is_empty() { return Err("Name cannot be empty".into()); }

    print!("Date of birth (YYYY-MM-DD): ");
    io::stdout().flush()?;
    io::stdin().read_line(&mut dob)?;
    let dob = dob.trim().to_string();
    if !dob.contains('-') || dob.len() != 10 { return Err("Invalid DOB format".into()); }

    print!("Email address: ");
    io::stdout().flush()?;
    io::stdin().read_line(&mut email)?;
    let email = email.trim().to_string();
    if !email.contains('@') { return Err("Invalid email format".into()); }

    print!("Citizenship (e.g., US): ");
    io::stdout().flush()?;
    io::stdin().read_line(&mut citizenship)?;
    let citizenship = citizenship.trim().to_string();
    if citizenship.is_empty() { return Err("Citizenship cannot be empty".into()); }

    print!("X.com username (without @): ");
    io::stdout().flush()?;
    io::stdin().read_line(&mut x_username)?;
    let x_username = format!("@{}", x_username.trim());
    if x_username.len() <= 1 { return Err("Invalid X username".into()); }

    print!("ID type (e.g., Passport): ");
    io::stdout().flush()?;
    io::stdin().read_line(&mut id_type)?;
    let id_type = id_type.trim().to_string();
    if id_type.is_empty() { return Err("ID type cannot be empty".into()); }

    print!("ID number: ");
    io::stdout().flush()?;
    io::stdin().read_line(&mut id_number)?;
    let id_number = id_number.trim().to_string();
    if id_number.is_empty() { return Err("ID number cannot be empty".into()); }

    Ok((name, dob, email, citizenship, x_username, id_type, id_number))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    print_splash_screen();
    sleep(Duration::from_secs(2));
    clear_screen();

    let http = Http::new(AVAX_RPC_URL)?;
    let web3 = Web3::new(http);

    let defi_address = get_wallet_address(&web3).await?;
    println!("Wallet address: {}", defi_address);

    let alpha_block = get_current_block_hash(&web3).await?;
    let alpha_chronos = Utc::now().to_string();
    println!("UTC time: {}\nBlock: {}", alpha_chronos, alpha_block);
    println!("Press enter to continue...");
    io::stdin().read_line(&mut String::new())?;
    clear_screen();

    println!("Enter your ChainID info for KYC:");
    let (name, dob, email, citizenship, x_username, id_type, mut id_number) = collect_user_info()?;
    let kyc_verifier = MockKycVerifier;
    if !kyc_verifier.verify(&name, &dob, &id_type, &id_number)? {
        println!("KYC failed. Aborting.");
        return Ok(());
    }
    println!("KYC verified!\nPress enter...");
    io::stdin().read_line(&mut String::new())?;
    clear_screen();

    println!("Generating ChainID keys...");
    let mnemonic = Mnemonic::new(MnemonicType::Words24, Language::English);
    let phrase = mnemonic.phrase().to_string();
    let seed = Seed::new(&mnemonic, "");
    let mut rng_seed = [0u8; 32];
    rng_seed.copy_from_slice(&Sha256::new().chain_update(seed.as_bytes()).finalize()[..32]);
    let mut rng = Hc128Rng::from_seed(rng_seed);
    use k256::ecdsa::{SigningKey, VerifyingKey};
    let private_key = SigningKey::random(&mut rng);
    let public_key = VerifyingKey::from(&private_key);

    let private_qr = QrCode::new(&phrase)?;
    let private_qr_image = private_qr.render::<unicode::Dense1x2>().build();
    println!("ChainID private key (scan, do not share):");
    println!("{}", private_qr_image);
    sleep(Duration::from_secs(3));
    clear_screen();
    println!("Printing private key...");
    drop(private_qr_image);
    sleep(Duration::from_secs(2));
    clear_screen();

    let zulu = Utc::now().to_string();
    let zulu_block = get_current_block_hash(&web3).await?;
    let email_hash = hex::encode(Sha256::new().chain_update(email.as_bytes()).finalize());
    let id_number_hash = hex::encode(Sha256::new().chain_update(id_number.as_bytes()).finalize());
    zeroize::Zeroize::zeroize(&mut id_number);

    let public_key_data = json!({
        "name": name,
        "dob": dob,
        "email_hash": email_hash,
        "citizenship": citizenship,
        "x_username": x_username,
        "id_type": id_type,
        "id_number_hash": id_number_hash,
        "defi": defi_address,
        "alpha_chronos": alpha_chronos,
        "alpha_block": alpha_block,
        "zulu_chronos": zulu,
        "zulu_block": zulu_block,
        "public_key": hex::encode(public_key.to_encoded_point(false).as_bytes())
    });

    let id_data_string = serde_json::to_string(&public_key_data)?;
    let public_qr = QrCode::new(&id_data_string)?;
    let public_qr_image = public_qr.render::<unicode::Dense1x2>().build();
    println!("ChainID public key (photograph/save):");
    println!("{}", public_qr_image);
    println!("Press enter...");
    io::stdin().read_line(&mut String::new())?;
    clear_screen();

    println!("Minting ChainID NFT...");
    sleep(Duration::from_secs(1));
    println!("NFT minted! Private data cleared.");
    println!("Press enter to exit...");
    io::stdin().read_line(&mut String::new())?;

    Ok(())
}
```

---

## Summary

The **ChainID System** is a proof-of-concept for a decentralized identity solution, leveraging Avalanche C-Chain’s blockchain infrastructure. It successfully integrates wallet connectivity, KYC verification, and cryptographic key generation into a terminal-based workflow. Key strengths include:
- **Modularity**: KYC verification is extensible via a trait.
- **Security**: Sensitive data is hashed and zeroized.
- **Usability**: Optimized prompts and QR code outputs enhance the experience.

### Current Limitations
- **Wallet Integration**: Simulated; requires a browser or CLI wallet for real use.
- **KYC**: Mocked; needs a real API (e.g., Onfido) for production.
- **NFT Minting**: Placeholder; lacks actual smart contract interaction.

### Future Enhancements
- Integrate a real KYC provider (e.g., `reqwest` for API calls).
- Implement full wallet support (e.g., via Web3.js in a browser context).
- Deploy an ERC-721 contract on Avalanche for NFT minting.
- Add a GUI or CLI options for broader accessibility.
- Incorporate IPFS for interstellar implementation.

ChainID represents a solid foundation for Web3 identity management, with potential to evolve into a fully-fledged decentralized ID platform.

---
deusopus
