---
title: "Building Smart Contracts for Dutch Auction (Part 2)"
date: 2018-04-16
draft: false
---

In the previous article, we've figured out how typical ICOs work from a technical POV, picked up the proper toolbox for development, and learned how to deal with potential EVM problems.

Now it’s time to write an actual Dutch Auction in Solidity!

<!--more-->

From the Wikipedia's definition of [Dutch Auction](https://en.wikipedia.org/wiki/Dutch_auction):

> Most commonly, it means an auction in which the auctioneer begins with a high asking price, and lowers it until some participant accepts the price, or it reaches a predetermined reserve price.

A Dutch Auction has several advantages over a classic crowdsale:

- Allows participation that does not offer price advantages to wealthier participants.
- Allows participation from those who wish to buy large amounts, to participate fairly.
- Ensures extended purchasing event period to prevent network delays, long queues, and price advantages to those who can afford to pay more gas.
- Prioritizes the number of participants over the number of dollars raised

![A 1957 Dutch Auction in Germany to sell fruit](https://upload.wikimedia.org/wikipedia/commons/c/c5/Bundesarchiv_B_145_Bild-F004491-0002%2C_Kirschenversteigerung_an_der_Mosel.jpg)

Just like a real-life auction, we have to accept the bids to-be-claimed-later and save them in contract storage. In every bid we have to fix current token price, stake amount, and have a couple of helper notes (e.g. whether bid tokens were already claimed or the bid was placed using Bitcoin donation):

```solidity
contract DutchAuction is PriceDecay150 { 
    // Auction Bid 
    struct Bid {
        uint256 price; 
        uint256 transfer; 
        bool placed; 
        bool claimed; 
        bool isBitcoin;
    }

// Address to bid mapping 
mapping (address => Bid) public bids; 
```

As you can notice, a Dutch Auction in Ethereum is as simple as a [ERC20 Token](https://eips.ethereum.org/EIPS/eip-20) — main variable called bids defines the details of the bid, placed by a specific Ethereum address.

This, however, introduces us to the first limitation — **only a single bid can be placed by a unique address**. Of course, we can modify our mapping to store an array of bids per address and then add additional loops in placing / claiming functions. But this complication will increase the overall contract interaction gas costs (because of [expensive storage](https://ethereum.stackexchange.com/a/896/31032)) and gas costs for each new address bid. To avoid the unnecessary increase in cost per user, we delegate this problem to our fellow bidders — generating a new address in almost every Ethereum wallet is as easy as a couple of mouse clicks!

Every real-life Dutch Auction is split into two main stages — the bidding process and the actual goods selling after the final price is set. Obviously, you can’t sell anything before the bidding has started and you can’t accept the bids after the price is finalized. These kinds of security limitations can be achieved through Solidity’s [enumerations](http://solidity.readthedocs.io/en/v0.4.21/types.html#enums) and [function modifiers](http://solidity.readthedocs.io/en/v0.4.21/common-patterns.html#restricting-access):

```solidity
// Auction Stages
enum Stages {
    AuctionDeployed, 
    AuctionStarted, 
    AuctionEnded, 
    TokensDistributed
} 

// Current stage 
Stages public current_stage;

// Stage modifier 
modifier atStage(Stages _stage) { 
    require(current_stage == _stage); 
} 
```

After the auction contract is deployed to the network, it will have initial `AuctionDeployed` state. Then, `startAuction()` method will be called by the contract owner, allowing investors to place bids using [default fallback function](http://solidity.readthedocs.io/en/v0.4.21/contracts.html#fallback-function) `()`. Under some conditions, stage will be changed to AuctionEnded, allowing participants to start claiming their tokens by calling `claimTokens()` function. Every function listed above is limited by `atStage` modifier, which will throw EVM error and return any transferred funds back to the sender if the stage is incorrect:

```solidity
// Setup auction 
function startAuction(
    address _tokenAddress, 
    uint256 offering, 
    uint256 bonus
) external isOwner atStage (Stages.AuctionDeployed) {
    // Setup logic
} 
 
// Default fallback function 
function () public payable atStage(Stages.AuctionStarted) { 
    placeBidGeneric(msg.sender, msg.value, false); 
} 

// Claim tokens 
function claimTokens() external atStage(Stages.AuctionEnded) {
    // Claim logic
} 
```

Why not start an auction immediately after the contract is deployed? In `startAuction()`, we’re creating a `ShopToken` instance using it’s address in order to automatically transfer tokens to their owners. But this can be done directly in the constructor function, which implies that `ShopToken` contract is deployed before `DutchAuction`. We've decided to do it vice versa, ensuring the auction contract receives specified number of tokens for sale, using scriptable [Truffle migrations](https://www.trufflesuite.com/docs/truffle/getting-started/running-migrations):

```javascript
// Deploy contracts and start auction 
deployer.deploy(
    DutchAuction, 
    priceStart, 
    pricePrecision, 
    minimumBid, 
    claimPeriod, 
    walletAddress
).then(
    deployer.deploy(
        ShopToken, 
        DutchAuction.address, 
        initialSupply, 
        auctionSupply
    ).then(function() { 
        DutchAuction.deployed().then(
            function(instance) { 
                instance.startAuction(
                    ShopToken.address, 
                    offering.toString(), 
                    bonus.toString()
                )
            }
        )
    )
);
```

So far, we’ve learned the core concepts of a Dutch Auction and its representation in the Solidity language. In the next part, we will deal with the exponential price decay, Bitcoin-funded bids, and DApp development problems.

Originally published on [EVERY @ Medium](https://medium.com/@shopevery/building-smart-contracts-for-a-dutch-auction-part-2-69554d20f91c)