---
title: How Private Key Generation Works Through a Seed Phrase
slug: seed-phrase-key-generation
lang: en
date: '2026-09-08'
description: In the world of cryptocurrencies and blockchain, private keys play a crucial role in securing your assets. But how are they properly generated? One of the most common…
canonical: https://medium.com/@dev.kudasov/how-private-key-generation-works-through-a-seed-phrase-c3e4e08ccda2
---

![](https://cdn-images-1.medium.com/max/1024/0*D_DICv27CgFBpSN0.jpeg)

In the world of cryptocurrencies and blockchain, private keys play a crucial role in securing your assets. But how are they properly generated? One of the most common methods is using a **seed phrase**, a set of randomly generated words that can be used to recover your wallet. Let’s explore how this process works and how to generate a private key from a seed phrase.

**What is a Seed Phrase?**

A **seed phrase** (or mnemonic phrase) is a set of **12, 15, 18, or 24 randomly generated words** that can be used to create a cryptographic key and restore a cryptocurrency wallet. If you’ve ever used a crypto wallet, you’ve likely encountered a seed phrase.

The seed phrase acts as a **master key**: if you lose access to your wallet, you can recover it just by entering these words. That’s why it’s crucial to store your seed phrase securely — anyone who obtains it can access your funds.

**How is a Private Key Generated?**

The process of generating a private key from a seed phrase involves several steps. Let’s go through them in detail.

## 1. Generating the Seed Phrase

The first step is to create a seed phrase based on a random sequence of bits, known as **entropy**. This process follows the **BIP-39** standard, widely used in crypto wallets. Here’s how it works:

- **Generating entropy:** A random sequence of bits is created, typically **128 to 256 bits** in length.

- **Checksum calculation:** A checksum is added to the entropy by taking the first bits of the **SHA-256 hash** of the entropy.

- **Splitting into segments:** The resulting bit string is split into **11-bit segments**, and each segment is used as an index to select a word from the **BIP-39 wordlist** (which contains **2048 words**).

This process creates a mnemonic phrase. For example, your seed phrase might look like this:
```
abandon ability able about above absent absorb abstract absurd abuse access accident
```
## 2. Converting the Seed Phrase into a Binary Seed

Once we have a seed phrase, it is converted into a **binary seed** using **PBKDF2-HMAC-SHA512**, which applies a hashing function to the seed phrase along with a salt (by default, “mnemonic” + passphrase):
```
Seed = PBKDF2-HMAC-SHA512(mnemonic, salt = "mnemonic" + passphrase)
```
This process involves **2048 iterations** for enhanced security. The result is a **512-bit binary value**, which is then used for cryptographic key generation.

## 3. Generating the Private Key

Once we have the binary seed, we can use it to derive a **private key**. This process follows the **BIP-32 standard** (Hierarchical Deterministic Wallets). The private key is generated using **elliptic curve cryptography** (e.g., **secp256k1** in Ethereum).

A **private key** is essentially a **random number** that serves as the foundation for generating a **public key** and, consequently, a **crypto wallet address**.

## 4. Generating the Public Key and Address

From the private key, a **public key** is derived using mathematical operations. In networks like **Ethereum**, the public key is then **hashed using Keccak-256**, which produces the **wallet address**.

It’s important to note that your **public address** is simply a derivative of the private key, making each wallet unique and secure.

## Example: Generating a Private Key in React Native

Now, let’s look at how we can achieve the same result in **React Native** using JavaScript and libraries like react-native-bip39, ethereumjs-wallet, and ethereumjs-util.

## Installation of Required Libraries

Run the following command to install the necessary dependencies:
```
npm install ethers
```
Here is a React Native example that generates a **seed phrase, derives a private key, and obtains an Ethereum wallet address**:
```
import { ethers } from "ethers";

// Generate seed-phrase and private key
const generatePrivateKeyFromSeedPhrase = async () => {
// Generate 12-words seed-phrase
const mnemonic = ethers.Wallet.createRandom().mnemonic.phrase;
console.log("Mnemonic:", mnemonic);

// Create HD Wallet from seed-phrase
const wallet = ethers.Wallet.fromMnemonic(mnemonic);

// Get private key from Wallet
const privateKey = wallet.privateKey;
console.log("Private Key:", privateKey);

// Get Ethereum address from Wallet
const address = wallet.address;
console.log("Ethereum Address:", address);
};

generatePrivateKeyFromSeedPhrase();
```
## Example: Generating a Private Key in Dart

Below is an example of Dart code that demonstrates how to **generate a seed phrase, convert it into a binary seed, and derive a private key**:
```
import 'package:bip39/bip39.dart' as bip39;
import 'package:hex/hex.dart';
import 'package:ed25519_hd_key/ed25519_hd_key.dart';

void generatePrivateKeyFromSeedPhrase() async {
// Generate a seed phrase (e.g., 12 words)
String mnemonic = bip39.generateMnemonic();
print("Mnemonic: $mnemonic");

// Convert the seed phrase into a binary seed
String seed = bip39.mnemonicToSeedHex(mnemonic);
print("Seed: $seed");

// Generate a master private key from the binary seed
var masterKey = await ED25519_HD_KEY.derivePath("m/44'/60'/0'/0/0", HEX.decode(seed));
print("Private Key: ${HEX.encode(masterKey.key)}");
}

void main() {
generatePrivateKeyFromSeedPhrase();
}
```
## Conclusion

Generating a private key through a **seed phrase** is a **secure and standardized** method for managing your crypto assets. Thanks to protocols like **BIP-39 and BIP-32**, this process is **universally recognized** and used by most modern crypto wallets.

🔒 **Always store your seed phrase in a secure location and never share it with anyone.** Your seed phrase is the **key to your assets**, and losing it could mean losing access to your funds.

![](https://medium.com/_/stat?event=post.clientViewed&referrerSource=full_rss&postId=c3e4e08ccda2)
