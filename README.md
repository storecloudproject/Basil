**Basil: An open source proposal for improving the throughput of Tendermint(™), a BFT consensus algorithm**

(NOTE: This is not a Whitepaper; it’s a contribution to the Tendermint open source project)

<table>
  <tr>
    <td>Chris McCoy
chris@storeco.in</td>
    <td>Rag Bhagavatha
rag@storeco.in</td>
  </tr>
</table>


March 2018

v0.7

## Abstract

Tendermint is open source software for securely and consistently replicating the application state on many machines. The replication is Byzantine fault-tolerant in that it works even if up to 1/3 of machines fail in arbitrary ways. Tendermint core implements the blockchain consensus engine, which ensures that the same transactions are recorded on every machine in the same order. While highly performant, Tendermint core suffers from the curse of peer-to-peer system, namely, the communication overhead when a large number of machines are involved in the consensus. As the number of peers increases the throughput suffers, which makes Tendermint core unsuitable for large deployments with thousands of peers. In this paper we propose a novel idea to optimize the peer-to-peer communication, which ensures that the throughput is unaffected by the size of the network. Our proposal is an open source idea to potentially improve the core Tendermint consensus engine.

## Introduction

Tendermint consists of two components: a blockchain consensus engine and a generic application interface. The consensus engine, called Tendermint Core, ensures that the same transactions are recorded on every machine in the same order. The application interface, called the Application BlockChain Interface (ABCI), enables the transactions to be processed in any programming language. The Tendermint core is run in a network of peer machines where the application state is replicated. The ABCI app is responsible for implementing any business logic, such as validating and executing transactions. When a transaction is submitted to any of the peers running Tendermint core, the following steps take place.

1. The receiving peer calls CheckTx() on the connected ABCI app to validate the transaction. If the validation succeeds, the receiving peer queues the transaction to its local mempool and relays the transaction to its peers.

2. Each peer, which receives the relayed transaction repeats the CheckTx() call before queuing it in its local mempool.

Separate checks are required because each peer maintains the state machine locally, independent of other peers. However this leads to unnecessary calls to CheckTx() by every peer. It also burdens the ABCI app because all the peers who receive the transactions want to validate them before adding them to their local mempool. This can be optimized.

Secondly, the block proposer is elected on a round-robin basis, who proposes the new block for voting. The transactions included in the proposed block get committed, when the block is committed via a 3-stage process. Since the new blocks are produced one-at-a-time and the peers only focus on the transactions in the proposed block during the consensus rounds, a secure *shared* mempool can be devised for queuing validated transactions. The elected block producers dequeue the transactions from the shared mempool to create the new blocks.

In this paper, we examine an approach that optimizes the peer-to-peer messages required to validate the transactions and complete the consensus round. Specifically, we propose the following two enhancements to Tendermint core.

1. Eliminate the need for relaying the incoming transactions to other peers by devising a shared mempool that guarantees total order of transactions

2. Use the shared mempool to build the new block during consensus round

This approach is agnostic to the number of peers in the network. Since peer-to-peer communication is eliminated the consensus throughput remains unaffected by the number of peers. 

## Trust

Tendermint core offloads the trust responsibilities to the associated ABCI app. It focuses only on secure and consistent state machine replication among participating peers. For example, it relies on the ABCI app to validate the received transaction via a call to CheckTx() API. It *trusts* the ABCI interface to do the right things to validate the transaction based on whatever business rules applicable for that app. This reliance on the ABCI app for trust can be extended to avoid relaying transactions among the peers and the resulting unnecessary and repeated transaction validations. In this section, we propose a *shared* state machine model that the peers can rely on during the consensus rounds. We also demonstrate that the shared state is secure and tamper-proof.

Fig. 1 below shows the traditional interaction between Tendermint core and ABCI app.

![image alt text](image_0.png)

Fig. 1 — Interaction between Tendermint core and ABCI app

When a peer is elected as a proposer, it takes the transactions from its local mempool to build the new block and proposes the new block to its peers for voting. The transactions in the mempool may have been received by the proposing peer as well as from other peers. This means, the proposer somehow needs to grab a list of *validated* transactions to propose the new block for voting. Since transaction validation itself is offloaded to the ABCI app, maintaining the list of validated transactions can also be offloaded to it. So, we need a *shared* mempool with the following properties.

1. Total order of transactions — Transaction order must be guaranteed as multiple peers submit validated transactions to the shared mempool

2. Secure — It must be secure, so the peers can trust the entries in the shared mempool during consensus rounds

3. Consistent — The entries in the shared mempool must be consistent, so it offers the same view to all the peers

4. Available — The shared mempool must have high availability because the progress of consensus rounds depends on it

### Total order of transactions

Tendermint core cannot have blind trust on the ABCI app for total ordering. The peers forming the core consensus engine must do some work to guarantee total ordering without excessive communication overhead. We propose an approach similar to Proof-of-History used by Loom [1] that provides a way to cryptographically verify passage of time between two events. This approach is much simpler than maintaining synchronized clocks using NTP (Network Time Protocol) or vector clocks among the participating peers because these approaches still need peer-to-peer communication to agree on the total order. The ordering protocol uses a cryptographically secure function (CSF) written so that the output cannot be predicted from the input without completely executing the CSF to generate the output.  The function is run in a sequence, its previous output as the current input thus forming a series of outputs. Data for an event can be timestamped into this sequence by appending data as part of the input to the function. This guarantees that the data associated with a particular output must have been created prior to the data associated with the next output because the outputs form a sequence. Total order of the associated events is thus ensured.

One example of cryptographic secure function is sha256. Its output cannot be predicted without running the function. We can run the function starting with a random value and feed the output of the function as the input to the same function to form a sequence of outputs. Table 1 shows an output series with an initial random value as a timestamp — "2018-03-30 17:23:19 UTC".

<table>
  <tr>
    <td>Sequence #</td>
    <td>Input</td>
    <td>Output</td>
  </tr>
  <tr>
    <td>1</td>
    <td>sha256("2018-03-30 17:23:19 UTC")</td>
    <td>747a20d1d1788bc10aba1e67de95f3417083beca0d5966dda91ad7141abba729</td>
  </tr>
  <tr>
    <td>2</td>
    <td>sha256(“747a20d1d1788bc10aba1e67de95f3417083beca0d5966dda91ad7141abba729”)</td>
    <td>f8769a929f78b13df2ff73d1684a4acd425d7f233b5c271cb3439d02d77f246f</td>
  </tr>
  <tr>
    <td>3</td>
    <td>sha256(“f8769a929f78b13df2ff73d1684a4acd425d7f233b5c271cb3439d02d77f246f”)</td>
    <td>a4f8c12527884b91ae282d831ee926e8514466fc9ce6a095abd23c49c1aa88b6</td>
  </tr>
  <tr>
    <td>4</td>
    <td>sha256(“a4f8c12527884b91ae282d831ee926e8514466fc9ce6a095abd23c49c1aa88b6”)</td>
    <td>db1893f6caaeddacd29fc008bab127ced6b9ebbf00f4fcc1809ffbdbb9421a40</td>
  </tr>
</table>


Table 1 — A sample output sequence with cryptographic secure function

The 4th output "db1893f6caaeddacd29fc008bab127ced6b9ebbf00f4fcc1809ffbdbb9421a40" cannot be predicted without runing the function 4 times starting with the predefined random string. If we have events attached to this sequence as we generate the outputs, we can be assured that the events must have occured in the same sequence.

Table 2 shows a scheme that attaches event data to the input for the function. An event happening at the time of generating the next output can append the event data (such as the hash of the transaction, for example) to the input, in effect timestamping the event. In the following table H(Tx) is any hash function such as sha256 itself that produces the hash of event data.

<table>
  <tr>
    <td>Sequence #</td>
    <td>Input</td>
    <td>Output</td>
  </tr>
  <tr>
    <td>1</td>
    <td>sha256("2018-03-30 17:23:19 UTC")</td>
    <td>747a20d1d1788bc10aba1e67de95f3417083beca0d5966dda91ad7141abba729</td>
  </tr>
  <tr>
    <td>2</td>
    <td>sha256(append(H(Tx1), “747a20d1d1788bc10aba1e67de95f3417083beca0d5966dda91ad7141abba729”))</td>
    <td>99948638c673c5ea8beed9c0a53a31d357c320502358bc70f25bd4cfb55f40ef</td>
  </tr>
  <tr>
    <td>3</td>
    <td>sha256(“99948638c673c5ea8beed9c0a53a31d357c320502358bc70f25bd4cfb55f40ef”)</td>
    <td>84a2ee42fd7e65c9b479d559209420a79b9f3690564925ae60c6715c83afc4a4</td>
  </tr>
  <tr>
    <td>4</td>
    <td>sha256(append(H(Tx2), “84a2ee42fd7e65c9b479d559209420a79b9f3690564925ae60c6715c83afc4a4”))</td>
    <td>90e5ddcc53f67b2d6ca6e8ac41bfe34b6b989c4509691def61896c88d1a694c6</td>
  </tr>
</table>


Table 2 — A sample output sequence with event data in the input

Notice sequences 2 and 4 appended the hash of the incoming transactions to the outputs of 1 and 3 respectively, so the corresponding outputs not only depend on the previous outputs, but also the transaction data. Given two transactions Tx1 and Tx2, anyone can verify that Tx2 is created after Tx1.

Finally, we need a way to ensure that the peer creating the output sequences can prove that it received the transactions included in the sequence. This can be achieved with cryptographic signature scheme as shown in table 3 below.

<table>
  <tr>
    <td>Sequence #</td>
    <td>Input</td>
    <td>Output</td>
  </tr>
  <tr>
    <td>1</td>
    <td>sign(sha256("2018-03-30 17:23:19 UTC"), secret)</td>
    <td>747a20d1d1788bc10aba1e67de95f3417083beca0d5966dda91ad7141abba729</td>
  </tr>
  <tr>
    <td>2</td>
    <td>sign(sha256(append(H(Tx1), “747a20d1d1788bc10aba1e67de95f3417083beca0d5966dda91ad7141abba729”)), secret)</td>
    <td>99948638c673c5ea8beed9c0a53a31d357c320502358bc70f25bd4cfb55f40ef</td>
  </tr>
  <tr>
    <td>3</td>
    <td>sign(sha256(“99948638c673c5ea8beed9c0a53a31d357c320502358bc70f25bd4cfb55f40ef”), secret)</td>
    <td>84a2ee42fd7e65c9b479d559209420a79b9f3690564925ae60c6715c83afc4a4</td>
  </tr>
  <tr>
    <td>4</td>
    <td>sign(sha256(append(H(Tx2), “84a2ee42fd7e65c9b479d559209420a79b9f3690564925ae60c6715c83afc4a4”)), secret)</td>
    <td>90e5ddcc53f67b2d6ca6e8ac41bfe34b6b989c4509691def61896c88d1a694c6</td>
  </tr>
</table>


Table 3 — A sample output sequence with the signature of the peer

Each peer generates the output sequence independent of the other at predetermined intervals. Whenever the peers receive transactions from the clients, the following steps take place.

1. The peer calls CheckTx() to validate the transaction as before, but it doesn’t relay the transaction to its peers.

2. It computes the signed data as shown in table 3. It calls SequenceTx() (discussed later in this paper) on the ABCI app to add the sequenced and signed transaction data to the shared mempool. As discussed earlier, a trust is established between Tendermint core and ABCI app, so the same trust is used to persist the sequenced transaction in a shared mempool. 

Notice that there is no relaying of the transactions to the other peers. The unnecessary communication overhead is eliminated altogether. Notice also that only the peers that receive transactions need to call SequenceTx(). 

#### Malicious behavior in sequence generation

Each peer generates the output sequence independent of the other. This leaves one particular attack possible. Some peers may generate the sequence at a higher rate than others, so all the transactions received by them will have higher order (more recent) than other peers. But such malicious behavior is easy to detect because the malicious peer would produce a longer sequence in a given period of time (such as between consensus rounds) compared to the honest peers. Since Tendermint recommends punitive measures (such as slashing, etc.) for misbehaving peers, the same punishment can be extended for this attack also. Since each transaction in the shared mempool is signed by the peer sending it, the malicious peer can be identified by other peers.

### Security, consistency, and availability of ordered of transactions

We alluded to a new API, SequenceTx() above. This new interface must be added to the ABCI specification to build a shared mempool. The shared mempool stores the validated transactions received by all the peers. This responsibility is handed over to the ABCI app because it can use any suitable technology to implement the shared mempool. But more importantly, Tendermint core leverages the trust it has with the ABCI app, so it doesn’t have to deal with managing the shared mempool. It is assumed that the ABCI app ensures data consistency and availability. The order of transactions in the shared mempool are guaranteed by the approach described above.

In order to fulfill the management of shared mempool, getTxs(), and removeTx() interfaces are also added to the ABCI specification. getTxs()returns the current list of validated transactions in the shared mempool. The elected proposer calls this method to get the transactions before constructing the new block. Similarly, removeTx() is called by the proposer after deliverTx() to remove the finalized transaction from the shared mempool. Fig 2 below shows populating the shared mempool.

![image alt text](image_1.png)

Fig. 2 — Populating shared mempool with sequenced, signed transactions

Compare fig. 2 with fig. 1. Notice that the peers

* don’t add the transaction to their local mempool, but use SequenceTx()to store the *signed* transaction in the shared mempool

* don’t relay it to other peers. 

Fig. 3 shows the steps that the proposer takes when it is elected to add the new block to the blockchain.

![image alt text](image_2.png)

Fig. 3 — Block proposal and commit

The flow is similar to fig. 1, except for retrieving the proposal transactions. The proposer gets the proposal transactions from the shared mempool, instead of its local mempool. It uses getTxs() to get the proposal transactions. Since these transactions are added by multiple peers, the proposer needs to ensure that the proposal transactions are indeed valid. The proposer takes the following steps before adding them to the new block.

1. The signature in each proposal transaction is valid and belongs to the current set of validators. The Verify(signature, publicKey) construct is used to verify the signature. A proposal transaction failing signature verification is discarded

2. The verified list of transactions is then ordered based on the scheme described earlier. The ordered list of proposal transactions is then added to the block and block validation happens as usual

3. Once the new block is committed, removeTx() is called for each successful transaction and it is removed from the shared mempool. This step is not shown in fig. 3 above 

The usual locking mechanisms implemented in the Tendermint core works here also. While getTxs()is being processed, no additional transactions are added to the shared mempool. The peers can continue to accept transactions, sequence them, and store them in their local mempool until the shared mempool is unlocked.   

## Future work

The shared mempool opens the possibility of parallel consensus rounds by multiple block producers. Since the transactions are ordered in the shared mempool, multiple block producers can propose multiple blocks for voting. The prevote and precommit phases of multiple proposed blocks can be pipelined, so the voting happens in parallel. Only the commit phase needs to be serial, so the block proposed by the *current* block producer is committed first, followed by the block proposed by the *next* producer, and so on. Since the block producers are elected on a round-robin basis, the order of the producers is known to all the peers, so the transactions can be sliced according to this order. This is especially useful when large number of peers are configured in a network, each of which receives large volumes of transactions. 

## Conclusion

Tendermint core is a high performant consensus engine, but the communication overhead stemming from relaying messages among peers makes it unsuitable for large deployments with thousands of peers. We propose a novel approach to maintain a shared mempool of ordered transactions, which eliminates the need for relaying the incoming transactions to all the peers. The proposal also eliminates the security and trust issues by leveraging the ABCI interface, which Tendermint core trusts inherently. The proposed approach however, adds 3 more APIs to the ABCI interface and offloads the consistency and availability responsibilities to the ABCI app. This should not be perceived as a weakness because the Tendermint core is not a standalone consensus engine and is always used in association with an ABCI app.

## References

1. Loom: A new architecture for a high performance blockchain v0.8.10. Anatoly Yakovenko

2. Tendermint: Consensus without Mining. Jae Kwon

