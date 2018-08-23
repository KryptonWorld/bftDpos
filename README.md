# BFT-DPoS

## Over View

There are two concensus scheme used in the paper. One is the usual **BFT-DPOS** and the other is the **Real-Time BFT**. For the **BFT-DPOS**, the general idea is to have at least 2/3 producers confirming **twice** (directly or indirectly) on a block before it is *irreversible*.  As Vtalik stated, BFT is generally not achievable if a producer confiming it once. The **Real-Time BFT** is obtained by collecting confirmations on each produced block independently. Collecting more than 2/3 confirmations on a block can be done in parallel and thus only take time of a single round trip communication, which can be done in 0.5 seconds.

### BFT-DPOS
The more detailed steps of BFT-DPOS is as follows. Producers are producing blocks in rounds. In the begining of each round, a producer make a block and put a ```confirm_count``` in the block header. This represents the number of blocks she is confirming before this block (she can only confirm blocks that are before her current block and she cannot double confirming a same block). She then broadcast this block. Once a producer receives a block from another producer, she also put a counter on the number of confirmations of each block she has in her database. Once a block receives confirmations from more than 2/3 producers, she mark that as *irreversible candiadates*. Once an irreversible candiadates receives another confirmation from more than 2/3 producers, that block is *irreversible*.


The EOS BFT-DPoS in general works in following steps:

1. Initialize with 21 block producers
2. Randomly shuffle these producers to schedule the order of each of them to produce blocks
3. Each producer produces 6 blocks in her round
4. A block is always produced at some prefixed time stamp, i.e., every 0.5 second
5. If a producer receives a block, which is a ```complete``` block, she does the following
   * If she finds the block invalid or with state ```incomplete```, she can discard it
   * She then resolves possible forks, i.e., there might be ambiguous blocks of the same timestamp. She chooses the block that completes the longest chain
   * If the block she receives is already in her database, she can discard it
   * She then sign the block header and broadcast block header and its signature (this is a confirmation of the block)
6. If a producer receives a confirmation of a block header,  
   * If the block sate is ```complete``` and the number of signatures is greater than ```2*21/3+1```, she promote the block to state ```BFT-irreversible```
   * Note that when a producer signs a block, she also confirms the whole chain that with this block as a header. For instance, if she confirms block 100, and confirms block 200, then she also confirms all the blocks between 100 and 200. In the EOS system, every producer confirms all the blocks before his producing cycle and put the number of blocks confirmed in the block header. A producer never double confirms a block
   * When receives a new block confirmation from a producer, if this block confirms a sequence of blocks from his last confirmed one, the first 1/3 of the ```validated``` blocks become ```DPOS-irreversible```
   * A block is ```irreversible``` if either it is ```BFT-irreversible``` or ```DPOS-irreversible```
   * **Currently, EOS only updates DPOS-irreversibility**
7. If the current round is scheduled for a producer, she makes a block and broadcast it immediately. The block sate is then ```complete```
   * The producer updates the new elected producers every one minute
   * If the producers change, then the producer propose a new schedule
   * A new schedule takes effect after the block proposing it becomes ```DPOS-irreversible``` (the position is fixed!)
8. A system contract is responsible for voting new producers
   * The schedule_producer_function is executed when a producer is starting a new block
   * The system contract modifies the ```global_state```, which is a subset of the state of the virtual machine
   * The producer validates the block based on the ```global_state```
