## Introduction

With all the buzzes around the scalability of blockchain networks. This article is a simple introduction on how we can potentially scale up a blockchain network.

## What to do?

To start it off, it's worth saying that you can not scale a blockchain network just by what it is. If you lower the block time, upper the block size, then it just makes the hardware requirements of running a node for the network higher, thus making it centralized as you can not run your own node and have to rely on centralized node providing services. The only way to scale up a blockchain is to move some of the work off-chain, while having some sort of connection/settlement/commitment to the chain.

Ideally, good scaling solutions are ones that are:
- "Decentralized" - People can do those off-chain computation on their own, or there exists a network with many participants doing that for you, and this network must provide a reasonable way for any people to join in.
- "Secure" - There should not be any way or any reasonable way for an attacker or a centralized entity to attack and manipulate the protocol, and the security of the protocol *remains approximately the same even if the network scales up further.*

## State channels & Plasma

To start it off, we will have a look at two "layer 2" scaling solutions - state channels and plasma chains.

The basic idea of state channels is that, given a situation of a limited group of people wanting to transact with each other, you can write down planned transactions, compute the result at the end, and only submit one final transation to confirm this result on-chain. State channels are incredibly useful in cases where known parties have to make multiple transactions back and forth in a period of time, cause it reduces all tasks to only one transaction, and every channel's transaction being made are done instantly because it's not on-chain, it's just a mutual short-term agreement between the parties. However, there are several drawbacks to such protocol. If there are not many transactions going on, it might not be exactly useful, and the only thing you can do is payments across a *limited group of people*. It is also impossible to build many types of applications just by using channels, because again, it's limited to a group of parties, so something like a complex decentralized lending protocol which relies on global consensus is not possible. Overall, state channels are still very superior for payments, and one should use them to transfer money if they can because of how fast and cheap it is for the purpose. This type of scaling solution is very popular towards high-end Bitcoin users, with the uprising usage of Lightning Network.

Another type of scaling solution that's also limited to a group of parties is plasma chain. The basic idea is that you deposit some assets into the plasma protool, then, you or any other party in the same transacting group keep transactions off-chain and post state commitment claiming the resulting state of those transactions to the layer 1 periodically. If any party thinks that the commitment is false, they create a dispute and submit proof that this commitment is faulty. In the end, you can choose to withdraw your assets corresponding to the current state commitment.

## Sharding & rollups

Another approach of scaling the blockchain is to split it into many smaller chains, which reduce the overall computation and data storage needed for each of the chain, which is often called "sharding", or considered as "rollups" - another type of layer 2 scaling solution.

This solution is often misunderstood, cause many think it is just literally splitting one chain into child chains without doing anything extra. This would cause huge consensus security concerns, cause by having multiple chains with their own consensus performing a 51% attack is much easier, and if one child chain falls, the whole network falls. Another problem might be that those chains are not connected and the process of communicating between the chains might be a difficult problem.

The correct way to shard the chains is to use one chain as a consensus layer, and then have multiple child chains (or "rollups") uploading their transactions onto it. By doing this, no single child chain have to have their own consensus which means consensus security is not reduced. But the special part about this is that the child chains only care about their chain's transactions posted on the beacon chain and have their own independent state. They do not do anything with other chains' data, transactions from other chains are just there in the beacon block but ignored by them. By doing this, we have sharded transaction computation and state storage from one chain to multiple ones.

To bridge the token from the beacon chain to the rollup and reverse, usually there are two classes of rollups to ensure correctness of the state - Optimistic rollups and ZK rollups. Both upload a state root as a commitment to the global rollup's state, zk rollups will have a small-and-cheap-to-verify proof that can prove correctness of a rollup block uploaded, while Optimistic rollups utilize a fraud proof system, where a rollup block is considered truthy until someone says that it doesn't and challenges the person who uploaded the block.

But you might have noticed that the one thing we didn't shard is data availability (the block, the transactions etc). Even though everything is cheaper, the amount of transactions handled by the entire network is still the same cause it's still just one beacon block we are doing stuff on. So to scale up, we would need to shard data availability (or DA sharding for short). The idea is fairly simple, you have special pieces of data on the beacon block called "blobs", and you have a cryptographic commitment of each of these blobs stored on each L1 blocks. Rollups will upload their data availability (transactions and other data) as these blobs. The unique thing about this model is that nodes will just verify these blobs once against their commitment and then drop them, only storing the commitment. Rollups just have to store their own blobs and ignore others, but they can still request those blobs from other rollup nodes if they wanted to and verify them using the commitment stored. By doing this, the throughput will increase cause more transactions can be handled while sacrificing no decentralization nor security. The only problem with this solution is that there are still the one-time data transportation cost and the cost to verify commitments, so we can only shard data availability to an extent.

Rollups and DA sharding are incredibly powerful in that it has the same capability as a normal blockchain network, and is the only current safe way to have a "multichain-ish" network.

### Compression tricks

Although this is not related to sharding, it's commonly used in Ethereum rollups to gain more capacity, hence the name "rollups".

To understand how this works, we must have the basic idea of a simple transaction, then we will have a look at how each property can be compressed. 

* Nonce (~3 bytes): Basically entropy so that transactions have different signatures. This can be omitted entirely because we can get the current nonce directly from state, which reduces 3 bytes.
* Gasprice (~8 bytes): How much would a the person pay for 1 gas. We can make the user pay within a fixed range of gasprices, eg. a choice of 16 consecutive powers of two, or can be omitted entirely and gas will be paid to block proposers using other sources like a state channel, which would get to 0-0.5 bytes.
* Gas (3 bytes): Amount of gas. We can either use the same trick as above, or remove it entirely using a fixed-size gas, which would get to 0-0.5 bytes also.
* To (21 bytes): Receiver's address. We can make an account manager to reduce receiver's address. Basically, an address will be registered with an index, and a reasonable index size would be somewhere 4 bytes which we can ensure that we would not be able to max out for several lifetimes.
* Value (~9 bytes): Amount to transact. We can use scientific notations for this, which would result in approximately 3 bytes.
* Signature (~68 bytes): Signature of the transaction. We can use BLS signature aggregation for this. The basic idea is that you can aggregate multiple signatures into only one signature. The cost is fixed to only one signature, so let's just say this one takes approximately 0.5 bytes per transaction.
* From (0 byte): Sender's address. Originally we can omit the sender address because it can be recovered from the signature, but in this case we need it to verify the aggregated signature. So that's 4 bytes added.

(Sizes are in bytes, taken from Vitalik's blog on rollups).

Overall, we have reduced a transaction which has the size of ~112 bytes to somewhere around ~12 bytes, which is almost a ten-time improvement in transaction capacity.

One more important note is that in many cases of a zk rollup, some transaction fields can be removed entirely because they have been proven through the zk proof already, a common example would be signatures.

### Going layer 3?

A wild thought would be to stack rollups on top of each other. While this may sound like a nice idea, it just doesn't work, because in the end the DA is the same and no improvement in capacity is being added. However, we can build state channels and plasma chains on top of rollups.

## Cryptographic tricks

Cryptographic tricks refer to all sorts of cryptography usage in optimization of the blockchain, but here are some of the most popular use cases.

### ZK proof

There are three important aspects in a blockchain: verification, execution, and data storage, succinct zk proof schemes like SNARKs or STARKs can vastly improve efficiency in verification and execution, which results in more capacity gained. 

### Data storage commitment

How would we deal with data storage then? First, we will split it into two types of data storage, chain-related storage and application storage. Chain-related storage can only be optimized through sharding tricks mentioned above, state data sharding through rollups and then do data availability sharding. But with the case of application storage, we have a slightly different approach.

The idea is similar to what I have mentioned about DA sharding, you keep some cryptographic commitment on-chain, store data off-chain, and can then request the data from a p2p network like IPFS or a similar DHT protocol and verify it with the commitment stored. Only problem with this is that data is voluntarily stored, so if your app has a large userbase or a model that makes people host data behind the scene, it would work, but otherwise it wouldn't. A further approach would be to have a protocol similar to Sia, which pays hosts to store data, uses the same data sampling trick for proof of storage, punishes hosts if they don't store data, and incentivizes at download time in an *optimistic* way so that they would provide data at any time. For more on how it works, check out their site.

## Indirect scaling solutions

### Better smart contract languages/compilers

While this may sound dumb, it is actually a fair point.

First, let's talk about contract execution. A bad contract language design/compiler would produce unnecessary opcodes, meaning more execution needed. For example, a contract written in Huff or Vyper (with their current official compilers) might be twice or even three times more gas efficient than one that was written in Solidity (using the solc compiler). We don't even need to shard another chain or use another rollup in the first place if devs adopt those languages rather than Solidity.

Second, let's talk about contract code storage. Again, a bad contract language design/compiler would produce unnecessary opcodes, and that not only contribute to worsened execution efficiency, but also data storage cost for those contracts. A few months prior to the current time of writing, a research has shown that 10% of Solidity contracts in Ethereum are useless opcodes generated by the solc compiler. Yep, tens of gigabytes of the Ethereum blockchain network currently are probably solc bloat, yikes!

People seem to not realize the damages a bad smart contract language may bring. Sure, it hurts apps' performance individually, but the problem is more complicated when a language is mass-adopted. A mass-adopted smart contract language that has 3x more opcodes and 3x lower gas efficiency directly contributes to the network being 3 times less scalable than what it could have been. We don't even need to shard 3 extra chains, we need a better smart contract language designs and compilers!

