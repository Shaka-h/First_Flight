---
title: Protocol Audit Report
author: Miriam shaka
date: April 5, 2025
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries PuppyRaffle Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Cyfrin](https://cyfrin.io)
Lead Auditors: 
- Miriam Shaka

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [EggHuntGame: Eggstravaganza NFT Hunt](#egghuntgame-eggstravaganza-nft-hunt)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
    - [\[H-1\] Weak Randomness in `EggHuntGame::searchForEgg` allows players to influence or predict the `EggHuntGame::random` to get lower than the `EggHuntGame::eggFindThreshold` and mint themselves an egg](#h-1-weak-randomness-in-egghuntgamesearchforegg-allows-players-to-influence-or-predict-the-egghuntgamerandom-to-get-lower-than-the-egghuntgameeggfindthreshold-and-mint-themselves-an-egg)
    - [\[H-2\] There is no check if the egg depositer is the egg owner](#h-2-there-is-no-check-if-the-egg-depositer-is-the-egg-owner)
    - [\[M-1\] Unsafe `ERC721::_mint()` is used](#m-1-unsafe-erc721_mint-is-used)

# Protocol Summary

# EggHuntGame: Eggstravaganza NFT Hunt

About

EggHuntGame is a gamified NFT experience where participants search for hidden eggs to mint unique Eggstravaganza Egg NFTs.
Players engage in an interactive hunt during a designated game period, and successful egg finds can be deposited into a secure Egg Vault.

# Disclaimer

The  team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

- Commit Hash: 2a47715b30cf11ca82db148704e67652ad679cd8


## Scope 

- In Scope: All contracts in the `src` directory are in scope.

```
./src/
#-- EggHuntGame.sol       // Main game contract managing the egg hunt lifecycle and minting process.
#-- EggVault.sol          // Vault contract for securely storing deposited Egg NFTs.
#-- EggstravaganzaNFT.sol // ERC721-style NFT contract for minting unique Egg NFTs.
```

## Roles

Game Owner: The deployer/administrator who starts and ends the game, adjusts game parameters, and manages ownership.
Player: Participants who call the egg search function, mint Egg NFTs upon successful searches, and may deposit them into the vault.
Vault Owner: The owner of the EggVault contract responsible for managing deposited eggs.


# Executive Summary

This audit reviewed the Eggstravaganza NFT Hunt core contracts. A total of 4 issues were identified. Most issues are solvable with minor code changes

## Issues found


| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 1                      |
| Medium   | 3                      |
| Low      | 1                      |
| Info     | 7                      |
| Gas      | 2                      |
| Total    | 16                     |

# Findings

# High

### [H-1] Weak Randomness in `EggHuntGame::searchForEgg` allows players to influence or predict the `EggHuntGame::random` to get lower than the `EggHuntGame::eggFindThreshold` and mint themselves an egg

**Description:** Hashing `msg.sender`, `block,timestamp`, `eggCounter` and `block.prevrandao` together creates a predictable final number. Malicious players can manipulate these values or know them ahead of time to get a random number below the eggFindThreshold.

**Impact:** The weak randomness undermines the fairness and unpredictability of the game. A user can gain an unfair advantage by timing or manipulating `searchForEgg` to consistently `mintEgg`. 

**Proof of Concept:**

1. Validators can know the values of `block.timestamp` and `block.prevrandao` ahead of time. See the [solidity blog on prevrandao](https://soliditydeveloper.com/prevrandao).
2. User can mine/manipulate their `msg.sender` value.
3. The `eggCounter` is public, allowing off-chain simulation of the random value before sending a transaction.

Using on-chain values as a randomness seed is a [well-documented attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.

**Recommended Mitigation:** Consider using a cryptographically provable random number generator such as [Chainlink VRF](https://docs.chain.link/vrf)


### [H-2] There is no check if the egg depositer is the egg owner

**Summary**

The `EggVault::depositEgg` function has no checking system for a depositor, allowing anyone to call this function without ensuring that the `depositor` is the rightful owner of the `tokenId`.

**Vulnerability Details**

**Proof of Concept:**

1. Mint egg to vault
2. Attacker calls `EggVault::depositEgg` and the function will think that this is a valid `depositor`
3. Attacker calls `EggVault::withdrawEgg`, freely withdraw his stolen NFT from the vault.

**Proof of Code:**

Add the following code to the `EggHuntGameTest.t.sol` file.

```Solidity
    function test_Attacker_Steals_Egg_FromVault() public {
        vm.prank(address(game));
        nft.mintEgg(address(vault), 999);

        // assuming bob is the attacker
        vm.prank(bob);
        vault.depositEgg(999, bob);

        assertEq(vault.eggDepositors(999), bob);
        assertTrue(vault.isEggDeposited(999));

        vm.prank(bob);
        vault.withdrawEgg(999);

        assertEq(nft.ownerOf(999), bob);
    }
```

**Impact**

1. Attackers can withdraw other people's NFTs
2. Original depositor cannot withdraw his NFT

**Tools Used**

1. Foundry

**Recommendations**

To prevent this problem, we should add an NFT ownership check before assigning a `depositor`.

```diff
    function depositEgg(uint256 tokenId, address depositor) public {
        require(eggNFT.ownerOf(tokenId) == address(this), "NFT not transferred to vault");
+       require(eggNFT.ownerOf(tokenId) == msg.sender, "Caller is not the owner");
        require(!storedEggs[tokenId], "Egg already deposited");
        storedEggs[tokenId] = true;
        eggDepositors[tokenId] = depositor;
        emit EggDeposited(depositor, tokenId);
    }
```


### [M-1] Unsafe `ERC721::_mint()` is used

**Summary**

The smart contract exposes a vulnerability by using the `_mint` function directly instead of the `safeMint` function in the `mintEgg` method. This issue can result in unsafe minting, potentially causing problems such as tokens being minted to invalid addresses or incompatible contracts.

**Vulnerability Details**

The `mintEgg` function allows only the approved game contract to mint NFTs. However, it uses the `_mint` function directly, which does not check for the safety of minting tokens to arbitrary addresses or addresses that may not be able to handle the NFT. The proper function to use in this context is `safeMint`, which ensures that tokens are minted safely to addresses that can accept them.

**Code Snippet:**
```js
function mintEgg(address to, uint256 tokenId) external returns (bool) {
    require(msg.sender == gameContract, "Unauthorized minter");
    _mint(to, tokenId);  // Vulnerability: using _mint instead of safeMint
    totalSupply += 1;
    return true;
}
```

**Impact**

* Token Loss: If an NFT is minted to a contract address that cannot handle the token (e.g., a contract that does not implement the IERC721Receiver interface), the minted token may be lost.

* Security Risks: The direct use of \_mint exposes the contract to potential issues with minting to malicious or invalid addresses, which could have unforeseen consequences.

* Reduced Interoperability: The absence of safe checks limits the contract's compatibility with other applications or contracts that expect NFTs to be safely transferable.

**Tools Used**

1. Foundry.
2. Manual Review.

**Recommendations**

Always use safeMint.

``` diff
function mintEgg(address to, uint256 tokenId) external returns (bool) {
    require(msg.sender == gameContract, "Unauthorized minter");
-   _mint(to, tokenId);
+   _safeMint(to, tokenId);
    totalSupply += 1;
    return true;
}
```