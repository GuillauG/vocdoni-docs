# Voĉdoni

Status of the document: *work in progress*

## A fully verifiable decentralized anonymous voting system

Cryptography and distributed p2p systems have given rise to a new, revolutionary digital technology which might change the way society organizes: blockchain. Among many other applications, it can allow for secure, transparent, distributed and resilient decision making.

In this document, we propose the system architecture of a decentralized anonymous voting platform.

## Overview

We want to bring decentralized voting to mass adoption. This requires a solution that has a user experience at least on par with current, centralized solutions.

We achieve this by externalizing the heavy lifting to a `relay` network - thereby allowing for lightweight end-user clients. This fundamental architectural choice has the following implications:

+ Minimizes transactions to the blockchain. Can potentially be used in the Ethereum Mainnet
+ `Voter` does not write, only reads from the blockchain
+ `Voter` can participate with an light-client (static web/app)
+ Secure vote anonymization using zk-SNARK
+ Data availability via distributed filesystems (IPFS)
+ Economically incentivized, `relay` network performs actions not possible by light-clients
+ The whole `process` is verifiable by any external observer

![overall](https://github.com/vocdoni/docs/raw/master/img/overall_design.png)

### Identity
The system is agnostic to the identity scheme used.

We are developing our implementation using [Iden3](https://iden3.io), which promises an ideal balance of decentralization and scalability.

Other identity schemes could eventually be integrated.

---

## Definitions
_The following concepts are referenced extensively throughout the document_

### Actors
`Voter`
+ A `voter` is the end user that will vote
+ Its digital representation is called an `identity`
+ Inside the voting `process`, and `identity` is specified by a `public key`
+ Can manage all interactions through a `light-client` (web/app)

`Organizer`
+ The entity that creates and manages a specific voting `process`
+ Needs to interact with the blockchain
+ Needs to add and retrieve data to IPFS
+ Pays for the costs of a `process`
+ Has the highest interest for the `process` to succeed
+ Knows the census beforehand (list of `voters` that are authorized to participate in a specific `process`)

`Relay`
+ Is used as a proxy between the `voter` and the blockchain
+ Is a selfish actor. Good behaviour is ensured through economic incentives
+ It may have to be split into several `relay` types
+ Performs functions that would not be possible on a `light-client`
  - Relays `vote packages` to other `relays`
  - Aggregates `vote packages` and adds them into the blockchain
  - Validates `franchise proofs`
  - Validates anti-spam proof-of-work nonce
  - Ensures data availability on IPFS
  - Is responsible for the data availability of the `vote packages` it has added (will lose stake if those are not available)
  - Exposes an IPFS proxy for the `light-clients`
  - Provides `census` Merkle-proofs to `voter`
  - Exposes an RPC end-point to the blockchain for the `light-client`
  - It should run a full Ethereum node

### Elements

`Process`
+ Represents an individual decision-making contest
+ Each process is identified by a `ProcessId`

`ProcessId`
+ A unique identifier for each process
+ Generated by the `Voting smart-contract` when a new process is created

`Light-client`
+ The client which `voters` will use to vote
+ Could be an app or static website running on a smart-phone
+ Provides UI for easy interaction
+ Runs some moderate computational loads, such as the `franchise proof` and the relay anti-spam proof-of-work

`Census`
+ A Merkle-tree made of the `public keys` of all the `voters`
+ The Merkle-root is hosted in the blockchain as a proof of the census
+ The tree needs to be publicly available (IPFS, DAT) for everyone to verify it.
+ The `zk-SNARK circuit` will use its Merkle-root to validate if a `voter` `public key` is part of it

`Franchise proof`
+ Leverages zk-SNARK technology
+ Used to prove two things without revealing critical data
  1. `Voter` is the owner of the `private key` corresponding to the `public key`
  2. `Voter`'s `public key` is included in the `census`
+ Generated in the `user` light-client
+ Is a CPU and memory intensive process
+ Is validated by the `relays` before adding the `voting package` in the blockchain
+ It is validated by the `organizer` once the `process` ends

`zk-SNARK circuit`
+ Used by the `voter` to generate the `franchise proof`
+ Used by the `relay` and `organizer` to validate the `franchise proof`
+ The same circuit can be use for any `process`
+ It requires a trusted setup to be generated

`Voting smart-contract`
+ The `process metadata` required by a `process` is published here when a new `process` is created
+ The `voter` light client retrieves the `process metadata` from here
+ Aggregated `vote package` hashes are added here
+ It holds the funds used to pay the `relays`
+ It holds the funds that the `relays` need to stake to ensure their good behaviour
+ When a `process` is successfully finished it transfers the funds to the `relays`

`Process metadata`
+ Is the metadata that needs to be public before a `process` starts
  - Merkle Root  of the `census` Merkle tree
  - `Vote encryption key`
  - Available `voting options`
  - `Process` start time (block number)
  - `Process` end time (block number)

`Vote package`
+ Is the set of data sent by the `Voter` to the `relay` in order to vote
  - Franchise proof
  - Encrypted vote: encrypt(selected `vote options` + random nonce)
  - `Nullifier` : hash( `ProcessId` + `Voter` `private key` )
  - `ProcessId`
  - Anti spam proof of work

`Vote encryption key`
+ Provided by `organizer`
+ The public key used by the `Voters` to encrypt their vote
+ Its private key needs to be made public at the end of the process, for everyone to decrypt the votes
+ Multiple `vote encryption keys` can be used to ensure that no one has access to the results before the `process` is finished
+ Entities providing the `vote encryption key` could be required to put some stake to ensure key publishing

`Voting options`
+ A potential option for the user to choose when they vote
+ They are published when a `process` is created
+ They could be exclusive or not

---

## Voting process chronology

### 0. `Identity` creation
  + Before the process in itself starts `voters` must have their digital `identity`  already created.
  + The unique requirement for those `identities` is that they need to be represented by a `public key`.
  + The system is agnostic to the identity system used, but further systems may require additional work to fully integrate.

### 1. The `organizer` generates a census
  + Presently assumes that the `organizer` has a list of all the `voters` that can participate
  + It aggregates all the `public keys` of the `voters` and generates the `census` Merkle tree from them.

### 2. The `organizer` publishes a new voting process.
  + Via a user interface, it provides the required `process metadata` regarding a voting process.
  + Sends a transaction to the `voting smart-contract`
    - It includes the `process metadata`, so is public for the other players
    - The funds sent in the transaction will be used to pay the `relays`
    - The amount sent is proportional to the needs of the `process` (number of participants, relay redundancy...)
  + In parallel it also publishes the `census` to IPFS, to make it available to everyone else.

### 3. `Voter` generates vote

#### Selects `voting options`
  + Gets the `process metadata` from the `voting smart-contract`
  + Gets the `census` Merkle-proof from a `relay`
  + Verifies her `public key` is in the published `census` Merkle tree
  + Selects the desired `voting options` from the `process metadata`

#### Generates vote
+ Encrypts the selected `voting options` and a random nonce with the `vote encryption keys`
  ```
   encrypted_vote = encrypt( selected_voting_options + random_nonce )
  ```
+ Generates the nullifier
```
nullifier = hash( process_id + user_private_key )
```

#### Franchise proof generation
The `franchise proof` is generated by running the `zk-SNARK voting cicuit` with several inputs.

+ **private input:** Private Key, census Merkle-proof
+ **public input:** census Merkle-root, Nullifier, ProcessId, Hash(encrypted vote)
+ **output:** Franchise proof

<p align="center">
  <img src="https://github.com/vocdoni/docs/raw/master/img/SnarksCircuit.png">
</p>



#### Vote packet

`Vote package` is created by aggregating

  - Franchise proof
  - Encrypted vote
  - Nullifier
  - ProcessID

Potentially the `vote package` can be encrypted with one or more `relay` `public keys` in order to choose the relay chain and minimize IP mapping
A proof-of-Work nonce (to avoid relay node spamming) must be attached to the packet. If the PoW is not correct, the relay pool will discard the packet.
The `vote package` and the nonce are sent to the `relay` pool

<p align="center">
  <img width="250" src="https://github.com/vocdoni/docs/raw/master/img/voting_packet.png">
</p>


### 4. `Relays` validate and add the `vote package` into the blockchain
  + The `relay` pool receives the `vote package` from the `user`
  + A `relay` verifies the proof-of-work and the `franchise proof`. If either is invalid, the `vote package` is discarded.
  + Chooses a set of pending `vote packages` and aggregates them into a single bundle of data.
  + Adds the aggregated data bundle to IPFS
  + Uploads the IPFS hash of the bundle to the blockchain

### 5. Finalizing the `process`
  + The owners of the `vote encryption keys` publish the corresponding `private keys`, so the votes and proofs can be decrypted
  + The `organizer` gets the hashes of aggregated `vote package` bundles from the blockchain
  + The `organizer` downloads the aggregated `vote package` bundles from IPFS
  + The `organizer` iterates over all the `vote packages`
    - It decrypts it with the published private key of the `voting encryption key`
    - It runs the `franchise proof` through the `zk-SNARK circuit` and discards invalid proofs
    - It discards `vote-packages` with repeated `nullifier` (double votes)
  - It computes the final results
  + The `organizer` signals bad `relays`
  + The `organizer` makes the `process` closing transaction, uploading the results
  + `Relays` are rewarded according to their contribution

![voting_process](https://github.com/vocdoni/docs/raw/master/img/voting_process.png)

### Known Weaknesses
- zk-SNARK trusted setup
- IP/vote mapping
- Most of the unresolved details are around creating a fully decentralized relay network. Multiple alternatives exist.
- A centralized trusted `relay` is a very valid option in a certain context

---

## Web Frontend

The web frontend and/or SmartPhone APP will allow users and organizers interact with the whole system.
This piece must be as static as possible, thus non dependent on any dynamic database. This will allow multiple ways to access the frontend, for instance via IPFS or ZeroNet, making censorship very hard or even impossible.

And static app makes it easy to checksum it, therefore minimizing atac vectors. 

The web frontend interact with different service providers such as Web3, IPFS gateway or TOR proxy. However the user must be able to choose its own provider, manually or from an official list.

<p align="center">
  <img src="https://github.com/vocdoni/docs/raw/master/img/web_frontend_general.png">
</p>
