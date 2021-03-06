# BFT-DPoS

-- Lin F. Yang
-- 08/23/2018

## Overview

There are two concensus schemes used in the EOS system. One is the usual **BFT-DPOS** and the other is the **Real-Time BFT**. For the **BFT-DPOS**, the general idea is to have at least 2/3 producers confirming **twice** (directly or indirectly) on a block before it is *irreversible*.  As Vtalik stated, BFT is generally not achievable if a producer confiming it once. The **Real-Time BFT** is obtained by collecting confirmations on each produced block independently. Collecting more than 2/3 confirmations on a block can be done in parallel and thus only takes time of a single round trip communication, which can be done in 0.5 seconds.

## BFT-DPOS
The more detailed steps of BFT-DPOS are as follows. Producers are producing blocks in rounds. In the begining of each round, a producer make a block and put a ```confirm_count``` in the block header. This represents the number of blocks she is confirming before this block (she can only confirm blocks that are before her current block and she cannot double confirming a same block). She then broadcast this block. Once a producer receives a block from another producer, she also put a counter on the number of confirmations of each block she has in her database. Once a block receives confirmations from more than 2/3 producers, she mark that as *irreversible candiadates*. The latest *irreversible candidate* is the proposed latest irreversible block (LIB). Once more than 2/3 producers implicitly agree on the proposed LIB, the proposed LIB becomes the final LIB.

## Realtime-BFT
The *Real-Time BFT* is obtained by collecting confirmations on each produced block independently. Once a block is produced by a producer, she broadcasts it and start collecting confirmations from other producers. If a block receives confirmations from more than 2/3 producers, it becomes ```BFT-irreversible```. Since the confirmations are not stored in block header, this method can only be used as a tool for users to obtain fast confirmation.

## Detail Steps

The EOS BFT-DPoS in general works in following steps:

1. Initialize with 21 block producers
2. Pseudo-randomly shuffle these producers to schedule the order of each of them to produce blocks
3. Each producer produces 12 blocks in her round
4. A block is always produced at some prefixed time stamp, i.e., every 0.5 second
5. If a producer receives a block, which is a ```complete``` block, she does the following
   * If she finds the block invalid or with state ```incomplete```, she can discard it
   * She then resolves possible forks, i.e., there might be ambiguous blocks of the same timestamp. She chooses the block that completes the longest chain
   * If the block she receives is already in her database, she can discard it
   * She then signs the block header and broadcasts block header and its signature (this is a confirmation of the block)
6. If a producer receives a confirmation of a block header,  
   * If the block sate is ```complete``` and the number of signatures is greater than ```2*21/3+1```, she promote the block to state ```BFT-irreversible```
   * Note that when a producer signs a block, she also confirms the whole chain that with this block as a header. For instance, if she produces block 100, and produce block 200, then she also confirms all the blocks from 100 to 200 (not including 200). In the EOS system, every producer confirms all the blocks before his producing cycle and put the number of blocks confirmed in the block header. A producer never double confirms a block
   * When receives a new block confirmation from a producer, if this block confirms a sequence of blocks before this block, she computes the number of confirmations of each other reversible block. If the a block has more then 2/3 confirmations, it become proposed-irreversible. The latest proposed-irreversible one is the proposed LIB. If a proposed-LIB receives more than 2/3\*nproducers confirmation, it becomes ```DPOS-irreversible```. For instance, the chain extending it has more than 2/3\*nproducers  distinct proposed-LIBs (each producer has a different proposed-LIB).
   * A block is ```irreversible``` if either it is ```BFT-irreversible``` or ```DPOS-irreversible```
   * **Currently, EOS only updates DPOS-irreversibility**
7. If the current round is scheduled for a producer, she makes a block and broadcast it immediately. The block sate is then ```complete```
   * The producer updates the new elected producers every one minute
   * If the producers change, then the producer proposes a new schedule
   * A new schedule takes effect after the block proposing it becomes ```DPOS-irreversible``` (the position is fixed!)
8. A system contract is responsible for voting new producers
   * The schedule_producer_function is executed when a producer is starting a new block (by generating a transaction)
   * The system contract modifies the ```global_state```, which is a subset of the state of the virtual machine
   * Another producer validates the block based on the ```global_state```
   
   
## Suggestions for implementation on KETH

1. What is stored in the blockheader?
    * ```confirmed``` the number of blocks this block confirms.  
       By signing this block this producer is confirming blocks ```[block_num() - confirmed, blocknum())``` 
       as being the best blocks for that range and that he has not signed any other statements that would contradict. 
       No producer should sign a block with overlapping ranges or it is proof of byzantine behavior. 
       When producing a block a producer is always confirming at least the block he is building off of.  
       A producer cannot confirm "this" block, only prior blocks.
    * ```schedule_version``` the producer schedule version that validates this block. This is used to
       indicate that the prior block which included new_producers->version has been marked
       irreversible and that the new producer schedule takes effect this block. 
    * ```new_producers``` schedule of new producers. This is optional, adding this if only new producers are scheduled. 
       If this is empty, then the schedule is still the current version. The ```new_producers``` is a schedule type 
       which includes a vector of producers and the version number. 
       The new_producers take effect once this block become ```DPoS-irreversible```.
2. What is stored in the system contract?
    * the producer candidates
    * the voters
    * the votes of each voter
    * the current schedule
3. What does the system contract do?
    * it is called every one minute
    * it proposes a new schedule to if the voted producers change, the schedule will be proposed to the block header
4. How to validate a block?
    * check whether the ```schedule_version``` matches the schedule in the latest prior DPoS-irreversible block that contains ```new_producers```
    * check whether the producer is the asigned one of the ```schedule_version```
    * run system contract ```on_block``` action (which proposes new schedule), and it is valid
    * check all the transactions
