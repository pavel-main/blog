---
title: "Building Smart Contracts for Dutch Auction (Part 1)"
date: 2018-03-28
draft: false
---

In this series, we’ll talk about common challenges we encountered during the development of smart contracts for a non-standard token sale.

Initial requirements for EVERY Token Sale:

- Dutch auction with duration of 30 days
- Exponential price decay, starting from $19.99 down to $0.01 per token
- Ability to accept Bitcoin-funded bids

<!--more-->

## Anatomy of an ICO

First of all, what exactly is a so-called “ICO” from a technical perspective?

Classic crowdsale smart contract code consists of two Solidity classes: the ERC20 Token and the token sale contract.

The most important part of an ERC20 contract is the balances variable, which represents the amount of tokens owned by specific Ethereum address:

```solidity
/**
 * Basic token
 * Basic version of StandardToken, with no allowances
 */
contract BasicToken is ERC20Basic, SafeMath {

  mapping(address => uint) balances;

  function balanceOf(address _owner) constant returns (uint balance) {
    return balances[_owner];
  }
}
```

All other functions operate on this mapping, allowing a trusted account to transfer, check their balance, or approve allowances. The constructor function `DutchAuction()`, called during contract deployment to the network, usually defines the total supply of tokens and immediately transfers them to the deployer address.

Typical Crowdsale contracts have a single payable `buyTokens()` function (the actual name will vary across contracts), which accepts Ethereum currency from the sender, performs some checks, and then calls connected token contract functions to reward the sender, transferring tokens from the contract deployer.

More complicated examples have soft or/and hard cap, token price discounts depending on the time, ability to refund, etc.

## Toolbox

Before starting to code, it’s important to pickup a proper set of tools to avoid shooting yourself in the foot.

{{< tweet 714794487042727936 >}}

[Truffle Suite](https://www.trufflesuite.com) is an industry-leading development framework for smart contracts, which provides automated compilation, scripting, unit testing (both in Solidity and JavaScript), and deployment to multiple Ethereum networks.

Another very helpful tool from the Truffle team is [ganache-cli](https://github.com/trufflesuite/ganache-cli) (formerly TestRPC), which allows you to run a local simulation of Ethereum blockchain without it being necessary to download ~3 million mainnet blocks worth hundreds of gigabytes.

[OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts) is a Solidity library containing a set of token, crowdsale contracts, and other interfaces, which are continuously tested and security audited by the development community. It also provides a lot of helpful test functions for development with Ganache. Notable examples include `expectThrow()`, which allows to tests to catch EVM exceptions, and `increaseTime()`, which enables local environment time movement simulation.

Example of `increaseTime()` usage in JavaScript tests:

```js
// Fast-forward to 1st day to decrease price
await increaseTime(helpers.byDays(1));

// Place 1st bid
await assertAcceptedBid(2, bids.second);
await assertReceivedWei(received.after2);

// Place 2nd bid
await assertAcceptedBid(3, bids.third);
await assertReceivedWei(received.after3);

// Fast-forward 7 days to claim tokens
await increaseTime(helpers.byDays(8));
```

[Ethlint](https://github.com/duaraghav8/Ethlint) is an automated tool to check your code conformity with the official [Solidity Style Guide](http://solidity.readthedocs.io/en/latest/style-guide.html). It’s pretty configurable, so you can turn off debatable cosmetic rules like the two blank line gap between contract declaration and its first function.

[solidity-coverage](https://github.com/sc-forks/solidity-coverage) measures which lines and branches of your code are covered by related unit tests, generating beautiful HTML reports using [istanbul](https://github.com/gotwarlost/istanbul).

Finally, [CircleCI](https://circleci.com) helped us to connect all of this tools and automatically run them one-by-one on every new feature branch and master branch updates, ensuring we don’t have any compilation errors, test failures, or obvious security flaws.

## Preparation

Even experienced software engineers make mistakes. A typo, an environment-specific (mis)configuration, or an unobvious EVM behaviour can lead to problems. It’s very important to be able to detect parts of the code responsible for them.

{{< tweet 399339918213652480 >}}

In Solidity, all of the logging is achieved through the concept of [Events](https://solidity.readthedocs.io/en/v0.4.21/contracts.html#events). Whenever you’re having a failure in tests, Truffle console shows exact events that were emitted during test transactions, helping you to figure out which parts of the code weren’t reached:

![Failed test report in Truffle console](solidity-tests.png)

Since every operation in EVM has a [non-zero gas price](https://docs.google.com/spreadsheets/d/1n6mRqkBz3iWcOlRem_mO09GtSKEKrAsfO7Frgx18pNU/edit#gid=0), it’s inefficient to put event emitting wherever you want, like during classic programming (e.g. a webserver can have request logging with DEBUG severity). Take a pen and paper, sit back and think about the crucial parts of your contract — what exactly do you want to track and be sure of?

For our case, the Dutch Auction, they’re split into three main categories:

- Auction stage transition events (deployed / started / ended)
- Bid placement events (accepted / rejected / refunded)
- Token claiming events (after the price is finalized)

```solidity
event AuctionDeployed(uint256 indexed priceStart);

event AuctionStarted(uint256 _startTime);

event AuctionEnded(
  uint256 priceFinal, 
  uint256 _endTime, 
  Endings ending
);

event BidAccepted(
  address indexed _address, 
  uint256 price, 
  uint256 transfer, 
  bool isBitcoin
);

event BidPartiallyRefunded(
  address indexed _address, 
  uint256 transfer
); 

event FundsTransfered(
  address indexed _bidder, 
  address indexed wallet, 
  uint256 amount
);

event TokensClaimed(
  address indexed _address, 
  uint256 amount
); 

event TokensDistributed(); 
```

A built-in step-by-step [debugger included in Truffle](https://www.trufflesuite.com/docs/truffle/getting-started/debugging-your-contracts) is also worth mentioning and has saved a lot of time, despite being an alpha feature during the development.

## Wrap-up

So far we’ve defined some foundational terminology together, in next parts we will talk about the actual requirements fulfillment, while overcoming or implementing things like:

- Solidity’s arithmetic limitations
- Measuring gas price through benchmarks
- Connecting BTC and ETH blockchains
- DApp development issues