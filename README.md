# Building a random number generator system on the celo-blockchain

## Introduction

Transparency and unanimity are essential in a Celo blockchain. They enable independent verification and make the system deterministic. However, because the blockchain is deterministic, it is difficult to produce random numbers. External oracles can be utilized to generate random data to get around this. We can add randomness while keeping transparency by incorporating these oracles. Measures like fairness checks and safe coding are crucial when developing blockchain betting games to prevent abuse. Overall, methods like using oracles make it possible to generate random numbers in a Celo blockchain and guarantee safe and open betting.

In this Tutorial, we'll examines the Celo blockchain's random number generation process and offers instructions for creating a Solidity contract for a casino betting game that uses random numbers. Additionally, methods for combating fraud in blockchain gambling games will be covered.


## Utilize randomness cases

The Celo blockchain random number generation is the subject of this tutorial. While deterministic processes can produce [pseudorandom numbers](https://en.wikipedia.org/wiki/Pseudorandomness) that are adequate for some uses, such as statistical sampling, they might not be appropriate for use cases that call for truly unpredictable numbers. The solutions for obtaining true randomness in the Celo blockchain environment will be covered in detail in this tutorial.

## NFTs

Numerous NFT initiatives on the Celo blockchain, including [OptiPunks](https://www.optipunks.com/), [Optimistic Bunnies](https://www.optimisticbunnies.com/), and [Optimistic Loogies](https://optimistic.loogies.io/), use random attribute assignment for minting NFTs. It is essential that the minters stay in the dark about the results of the mint until it is finished because the value of these traits can vary. 

## Games

Randomness is essential to how games work in the ecosystem surrounding the Celo blockchain. It generates hidden information and makes decision-making possible, improving gameplay. Without randomization, the blockchain games that can be played on Celo are those where all players have complete information, like chess or checkers. 

## The commit/reveal protocol

Due to its transparency, the Celo blockchain presents a hurdle when trying to generate random numbers. The commit/reveal protocol, however, provides a workaround. This protocol uses a [cryptographic hash function](https://en.wikipedia.org/wiki/Cryptographic_hash_function) to generate a mutually agreed-upon random value using a secret number that is not recorded on the blockchain. Participants can create random numbers on the Celo blockchain while maintaining transparency by utilizing this protocol. Let’s take a look at how it works:

1. Side A generates a random number, randomA
2. Side A sends a message with the hash of that number, hash(randomA). This
commits Side A to the value randomA, because while no one can guess the value of randomA, once side A provides it everyone can check that its value is correct
3. Side B sends a message with another random number, randomB
4. Side A reveals the value of randomA in a third message
5. Both sides accept that the random number is randomA ^ randomB, the [exclusive or (XOR)](https://en.wikipedia.org/wiki/Exclusive_or) of the two values

The benefit of using XOR in this situation is that both parties determine the value equally, preventing either party from selecting a favorable "random" value.

## Casino betting game tutorial

To illustrate the use of a random number generator in a real blockchain game, let's examine the code for Casino.sol, a betting game implemented in Solidity. This contract utilizes the commit/reveal scheme and can be accessed on the Celo blockchain GitHub.

## Setting up the contract

Let’s walk through the code for the Casino.sol betting game; it’s in this GitHub file.

First, we specify the license and Solidity version:

```
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;
```

Next, we define a contract called Casino. [Solidity contracts](https://blog.logrocket.com/writing-smart-contracts-solidity/) are somewhat similar to objects in other programming languages.

```
contract Casino {
```

Now, we create a [struct](https://docs.soliditylang.org/en/v0.8.14/structure-of-a-contract.html#struct-types), ProposedBet, where we’ll store information about proposed bets:

```
struct ProposedBet {
   address sideA;
   uint value;
   uint placedAt;
   bool accepted;  
 }    // struct ProposedBet
```

This struct does not include the commitment, the hash(randomA) value, because that value is used as the key to locate the ProposedBet. However, it does contain the following fields:

|Field|Type|Purpose|
|-----|----|-------|
|sideA|address|	the [address](https://docs.soliditylang.org/en/v0.8.14/types.html#address) that proposes the bet
|value|integer|	the size of the bet in Wei, the smallest denomination of Ether
|placedAt|integer|	the [timestamp](https://en.wikipedia.org/wiki/Unix_time) of the proposal
|accepted|Boolean|	whether the proposal has been accepted

N.B., the placedAt field is not used in this example, but I’ll explain later in this article why it’s important to keep track of this information

Next, we create an AcceptedBet struct to store the extra information after the bet is accepted.

An interesting difference here is that sideB provides us with randomB directly, rather than a hash.

```
struct AcceptedBet {
   address sideB;
   uint acceptedAt;
   uint randomB;
 }   // struct AcceptedBet
```

Here are the [mappings](https://docs.soliditylang.org/en/v0.8.14/types.html#mapping-types) that store the proposed and accepted bets:

```
// Proposed bets, keyed by the commitment value
 mapping(uint => ProposedBet) public proposedBet;
 // Accepted bets, also keyed by commitment value
 mapping(uint => AcceptedBet) public acceptedBet;
```

We then created an event called BetProposed. The standard method for communicating with the outside world in Solidity smart contracts is through events. This event notifies everyone that a user (in this case, sideA) is putting up a wager and its value.

```
event BetProposed (
   uint indexed _commitment,
   uint value
 );
```

We've now created a new event called BetAccepted. This incident signals to everyone that it is now time for randomA to be revealed, especially sideA, who made the bet. A message cannot be sent from the celo blockchain to a single user.

```
 event BetAccepted (
   uint indexed _commitment,
   address indexed _sideA
 );
```

Next, we create an event, BetSettled. This event is emitted when the bet is settled.

```
event BetSettled (
   uint indexed _commitment,
   address winner,
   address loser,
   uint value   
 );
```

Now, we create a proposeBet function. The commitment is the sole parameter to this function.

Everything else (the value of the bet and the identity of sideA) is available as part of the transaction.

Notice that this function is payable. This means that it can accept Ether in payment.

```
// Called by sideA to start the process
 function proposeBet(uint _commitment) external payable {
```

ost externally called functions, like proposedBet shown here, start with a bunch of require statements.

```
require(proposedBet[_commitment].value == 0,
     "there is already a bet on that commitment");
   require(msg.value > 0,
     "you need to actually bet something");
```

When creating a smart contract, we must consider the possibility of a malicious attempt to call the function. This presumption will lead us to implement safeguards.

In the above code, we have two conditions:

1. If there is already a bet on the commitment, reject this one. Otherwise, people might try to use it to overwrite existing bets, which would cause the amount sideA put in to get stuck in the contract forever
2. If the bet is for 0 Wei, reject it

If neither of these two conditions is met, we write the information to proposedBet.

We don't have to construct a new struct, load it with data, and then assign it to the mapping because of how Ethereum storage functions. Instead, each commitment value already has a struct that is zeroed out; we just need to change it.

```
 proposedBet[_commitment].sideA = msg.sender;
   proposedBet[_commitment].value = msg.value;
   proposedBet[_commitment].placedAt = block.timestamp;
   // accepted is false by default
```

Now, we tell the world about the proposed bet and the amount:

```
 emit BetProposed(_commitment, msg.value);
 }  // function proposeBet
```

We need two parameters to know what the user is accepting: the commitment and the user’s random value.

```
// Called by sideB to continue
 function acceptBet(uint _commitment, uint _random) external payable {
```

In the below code, we check for three potential issues before accepting the bet:

1. If the bet has already been accepted by someone, it cannot be accepted again
2. If the sideA‘s address is zero, it means that no one actually made the bet
3. sideB needs to bet the same amount as sideA

```
require(!proposedBet[_commitment].accepted,
     "Bet has already been accepted");
   require(proposedBet[_commitment].sideA != address(0),
     "Nobody made that bet");
   require(msg.value == proposedBet[_commitment].value,
     "Need to bet the same amount as sideA");
```

If all the requirements have been met, we create the new AcceptedBet, mark in the proposedBet that it had been accepted, and then emit a BetAccepted message.

```
acceptedBet[_commitment].sideB = msg.sender;
   acceptedBet[_commitment].acceptedAt = block.timestamp;
   acceptedBet[_commitment].randomB = _random;
   proposedBet[_commitment].accepted = true;
   emit BetAccepted(_commitment, proposedBet[_commitment].sideA);
 }   // function acceptBet
```

This next function is the great reveal!

sideA reveals randomA, and we are able to see who won:

```
 // Called by sideA to reveal their random value and conclude the bet
 function reveal(uint _random) external {
```

We don’t need the commitment itself as a parameter, because we can derive it from randomA.

```
 uint _commitment = uint256(keccak256(abi.encodePacked(_random)));
```

Solidity only allows us to transmit ETH to addresses of the type address payable to limit the danger of inadvertently sending it to addresses where it will get trapped.

```
 address payable _sideA = payable(msg.sender);
   address payable _sideB = payable(acceptedBet[_commitment].sideB);
```

The agreed random value is an XOR of the two random values, as explained below:

```
uint _agreedRandom = _random ^ acceptedBet[_commitment].randomB;
```

We’re going to use the value of the bet in multiple places within the contract, so for brevity and readability, we’ll create another variable, _value, to hold it.

```
 uint _value = proposedBet[_commitment].value;
```

There are two cases in which that proposedBet[_commitment].sideA == msg.sender does not equal to the commitment.

1. The user did not place the bet
2. The value provided as _random is incorrect. In this case, _commitment will be a different value and therefore the proposed bet in that location won’t have the correct value for sideA.

```
require(proposedBet[_commitment].sideA == msg.sender,
     "Not a bet you placed or wrong value");
```

```
 require(proposedBet[_commitment].accepted,
     "Bet has not been accepted yet");
```

The above proposedBet[_commitment].accepted function will only makes sense after the bet has been accepted.

Next, we use the least significant bit of the value to decide the winner:

```
// Pay and emit an event
   if (_agreedRandom % 2 == 0) {
```

Here, we give the winner the bet and emit a message to tell the world the bet has been settled.

```
  // sideA wins
     _sideA.transfer(2*_value);
     emit BetSettled(_commitment, _sideA, _sideB, _value);
   } else {
     // sideB wins
     _sideB.transfer(2*_value);
     emit BetSettled(_commitment, _sideB, _sideA, _value);     
   }
```

Now, we’ll delete the bet storage, which is no longer needed.

On the Celo blockchain, anyone can inspect the blockchain's history and view the commitment and revealed value of a bet. However, in the context of gas refunds and storage optimization, it may be necessary to delete this data to reclaim storage that is no longer required. This deletion process aims to optimize gas usage and storage management on the Celo blockchain.

```
  // Cleanup
   delete proposedBet[_commitment];
   delete acceptedBet[_commitment];
```

Finally, we have the end of the function and contract:

```
 }  // function reveal
}   // contract Casino
```

## Transacting with a rollup

As of this writing, the most economical way to transact on Ethereum is to [use a rollup](https://ethereum.org/en/layer-2/).

In the Celo blockchain ecosystem, the most cost-effective approach for transactions is to utilize a rollup. A rollup operates by writing all transactions to the Ethereum network but performs the processing in a more affordable environment. Ethereum's uncensorable nature allows anyone to verify the blockchain state.

The state root, containing the summarized data, is then posted to Layer 1 with guarantees, either [mathematical](https://ethereum.org/en/developers/docs/scaling/zk-rollups/) or [economic](https://ethereum.org/en/developers/docs/scaling/optimistic-rollups/), ensuring its correctness. By utilizing the state root, one can prove ownership or any other aspect of the state.

This mechanism enables inexpensive processing on the rollup or Layer 2, while storing transaction data on Ethereum or Layer 1, which can be comparatively expensive. At the time of writing, Layer 1 gas costs approximately 20,000 times more than Layer 2 gas in the [rollup](https://www.optimism.io/). You can [refer to the current ratio of Layer 1 to Layer 2 gas prices](https://public-grafana.optimism.io/d/9hkhMxn7z/public-dashboard?orgId=1&refresh=5m) to stay updated.

Due to this cost consideration, the Casino.sol game only reveals the randomA value. Although it could have been designed to retrieve the commitment value and distinguish between incorrect values and nonexistent bets, implementing such functionality on a rollup would significantly increase transaction costs.

## Testing the contract

casino-test.js is the JavaScript code that tests the Casino.sol contract. It is repetitive, so I’ll only explain the interesting parts.

The hash function on the ethers package ethers.utils.keccak256 accepts a string that contains a hexadecimal number. This number is not converted to 256bits if it is smaller, so for example 0x01, 0x0001, and 0x000001 all hash to different values. To create a hash that would be identical to the one produced on Solidity, we would need a 64-character number, even if it is 0x00..00. Using the hash function here is a simple way to make sure the value we generate is 32bytes.

```
const valA = ethers.utils.keccak256(0xBAD060A7)
```

We want to check both possible results: a sideA win and a sideB win.

If the value sideB sends is the same as the one hashed by sideA, the result is zero (any number xor itself is zero), and therefore sideB loses.

```
const hashA = ethers.utils.keccak256(valA)
const valBwin = ethers.utils.keccak256(0x600D60A7)
const valBlose = ethers.utils.keccak256(0xBAD060A7)
```

When conducting local testing with the Hardhat EVM, the revert reason is presented as a [Buffer](https://nodejs.org/api/buffer.html) object within the stack trace. However, when connected to a live Celo blockchain, the revert reason is obtained from the reason field. This distinction reflects how the revert reason is accessed and handled in the Celo blockchain environment.

This function lets us ignore this difference in the rest of the code.

```
// Chai's expect(<operation>).to.be.revertedWith behaves
// strangely, so I'm implementing that functionality myself
// with try/catch
const interpretErr = err => {
 if (err.reason)
   return err.reason
 else
   return err.stackTrace[0].message.value.toString('ascii')
}
```

Below is the standard way to use the [Chai testing library](https://www.chaijs.com/). We describe a piece of code with a number of it statements to denote actions that should happen.

```
describe("Casino", async () => {
 it("Not allow you to propose a zero Wei bet", async () => {
```

Here’s the standard Ethers mechanism for [creating a new instance of a contract](https://docs.ethers.io/v5/api/contract/contract-factory/#ContractFactory--creating):

```
 f = await ethers.getContractFactory("Casino")
   c = await f.deploy()
```

By default, transactions have a value (amount of attached Wei) of zero.

```
try {
     tx = await c.proposeBet(hashA)
```

The function call tx.wait() returns a Promise object. The expression await <Promise> pauses until the promise is resolved, and then either continues (if the promise is resolved successfully) or throws an error (if the promise ends with an error).

```
rcpt = await tx.wait()
```

If there is no error, it means that a zero Wei bet was accepted. This means the code failed the test.

```
// If we get here, it's a fail
     expect("this").to.equal("fail")
```

Here we catch the error and verify that the error matches the one we’d expect from the Casino.sol contract.

If we run using the Hardhat EVM, the Buffer we get back includes some other characters, so it’s easiest to just match to make sure we see the error string rather than check for equality.

```
} catch(err) {
     expect(interpretErr(err)).to
       .match(/you need to actually bet something/)       
   }
 })   // it "Not allow you to bet zero Wei"
```

The other error conditions, such as this one, are pretty similar:

```
it("Not allow you to accept a bet that doesn't exist", async () => {
   .
   .
   .
 })   // it "Not allow you to accept a bet that doesn't exist"
```

[We add an override hash as an additional argument](https://docs.ethers.io/v5/api/contract/contract/#Contract-functionsCall) to allow us to alter the contract interaction's default behavior (for instance, to attach a payment to the transaction). We send 10Wei in this instance to see if this form of wager is accepted:

```
it("Allow you to propose and accept bets", async () => {
   f = await ethers.getContractFactory("Casino")
   c = await f.deploy()
   tx = await c.proposeBet(hashA, {value: 10})
```

If a transaction is successful, we get the receipt when the promise of tx.wait() is resolved.

Among other things, that receipt has all the emitted events. In this case, we expect to have one event: BetProposed.

Of course, in production-level code we’d also check that the parameters emitted are correct.

```
 rcpt = await tx.wait()
   expect(rcpt.events[0].event).to.equal("BetProposed")
   tx = await c.acceptBet(hashA, valBwin, {value: 10})
   rcpt = await tx.wait()
   expect(rcpt.events[0].event).to.equal("BetAccepted")     
 })   // it "Allow you to accept a bet"
```

Sometimes we need to have a few successful operations to get to the failure we want to test, such as an attempt to accept a bet that has already been accepted:

```
it("Not allow you to accept an already accepted bet", async () => {
   f = await ethers.getContractFactory("Casino")
   c = await f.deploy()
   tx = await c.proposeBet(hashA, {value: 10})
   rcpt = await tx.wait()
   expect(rcpt.events[0].event).to.equal("BetProposed")
   tx = await c.acceptBet(hashA, valBwin, {value: 10})
   rcpt = await tx.wait()
   expect(rcpt.events[0].event).to.equal("BetAccepted")
```

In this scenario, if the bet has already been accepted in the example, the transaction will revert but remain on the Celo blockchain. Consequently, if sideA reveals prematurely, it opens the opportunity for anyone else to accept the bet with a winning value. This highlights the importance of carefully managing the timing of revealing values in the Celo blockchain environment to avoid potential exploitation by other participants.

```
try {
     tx = await c.acceptBet(hashA, valBwin, {value: 10})
     rcpt = await tx.wait()  
     expect("this").to.equal("fail")     
   } catch (err) {
       expect(interpretErr(err)).to
         .match(/Bet has already been accepted/)
   }         
 })   // it "Not allow you to accept an already accepted bet"
 it("Not allow you to accept with the wrong amount", async () => {
     .
     .
     .
 })   // it "Not allow you to accept with the wrong amount"  
 it("Not allow you to reveal with wrong value", async () => {
     .
     .
     .
 })   // it "Not allow you to accept an already accepted bet"
 it("Not allow you to reveal before bet is accepted", async () => {
     .
     .
     .
 })   // it "Not allow you to reveal before bet is accepted"
```

So far, we’ve used a single address for everything. However, to check a bet between two users we need to have two user addresses.

We’ll use Hardhat’s [ethers.getSigners()](https://hardhat.org/guides/waffle-testing.html#setting-up) to get an array of signers; all addresses are derived from the same mnemonic. Then, we’ll use the [Contract.connect](https://docs.ethers.io/v5/api/contract/contract/#Contract-connect) method to get a contract object that goes through one of those signers.

```
it("Work all the way through (B wins)", async () => {
   signer = await ethers.getSigners()
   f = await ethers.getContractFactory("Casino")
   cA = await f.deploy()
   cB = cA.connect(signer[1])
```

In this system, Ether is used both as the asset being gambled and as the currency used to pay for transactions. As a result, the change in sideA‘s balance is partially the result of paying for the reveal transaction.

To see how the balance changed because of the bet, we look at sideB.

We check the preBalanceB:

```
   .
   .
   .
   // A sends the transaction, so the change due to the
   // bet will only be clearly visible in B
   preBalanceB = await ethers.provider.getBalance(signer[1].address)
```

And compare it to the postBalanceB:

```
tx = await cA.reveal(valA)
   rcpt = await tx.wait()
   expect(rcpt.events[0].event).to.equal("BetSettled")
   postBalanceB = await ethers.provider.getBalance(signer[1].address)       
   deltaB = postBalanceB.sub(preBalanceB)
   expect(deltaB.toNumber()).to.equal(2e10)  
 })   // it "Work all the way through (B wins)"
 it("Work all the way through (A wins)", async () => {
     .
     .
     .
     .
   expect(deltaB.toNumber()).to.equal(0)
 })   // it "Work all the way through (A wins)"
})     // describe("Casino")
```

## Preventing abuse in the betting game

When creating a smart contract, you should think about potential abuse attempts by malicious individuals and put precautionary measures in place to stop them.

## Protecting from a never reveal

A spiteful, losing sideA could skip issuing the reveal transaction and prevent sideB from collecting on the bet because there is no requirement in the contract for sideA to divulge the random number.

Fortunately, there is a simple fix for this issue: record the timestamp of sideB's acceptance of the wager. Let sideB submit a forfeit transaction to collect on the wager if a set amount of time has transpired since the timestamp and sideA has not reacted with a legitimate revelation.

This is why the placedAt field, which was previously generated, records the time that a function is called.

## Protecting from frontrunning

In the Celo blockchain, Ethereum transactions are not executed immediately but are instead placed in the mempool, where miners (or block proposers) select which transactions to include in the submitted block. Typically, transactions with higher gas payments are prioritized as they offer greater profitability.

However, this structure opens the possibility of [frontrunning](https://consensys.github.io/smart-contract-best-practices/attacks/frontrunning/). For instance, if sideA detects sideB's acceptBet transaction in the mempool, revealing a random value that would cause sideA to lose, sideA can swiftly issue a different acceptBet transaction, potentially from a different address. If sideA's transaction offers more gas to the miner, it is likely to be executed first, allowing sideA to withdraw from the bet instead of losing it.

To address this frontrunning issue, we can remove the information asymmetry. When sideB submits the acceptBet transaction, sideA already knows randomA and randomB, enabling sideA to determine the winner. However, sideB remains unaware until the reveal.

By modifying sideB's acceptBet transaction to disclose only hash(randomB), sideA is also unable to determine the winner, eliminating the incentive for frontrunning. After sideB's acceptance is recorded on the blockchain, both sideA and sideB can issue reveal transactions.

Once one side initiates a reveal, the other side becomes aware of the winner. Introducing forfeit transactions removes the advantage of refusing to reveal, aside from the small charge for the transaction itself.

It's important to note that a potential issue arises if sideB makes the exact same commitment as sideA. In this case, when sideA reveals, sideB can also reveal the same number. However, given the specific design of this game, sideB would essentially ensure sideA's victory in such a scenario

## Conclusion

In this article, we examined a line-by-line review of a Solidity contract for a casino betting game to illustrate the construction of a random number generator for the Celo blockchain. Generating random numbers in a deterministic environment poses challenges, but by involving users in the process, we achieved a robust solution.

Additionally, we explored various strategies to prevent abuse and hostile actions in blockchain betting games, ensuring fair gameplay and maintaining security.

Ultimately, as long as both sides share an interest in the randomness of the outcome, we can have confidence in the fairness of the results within the Celo blockchain ecosystem.











