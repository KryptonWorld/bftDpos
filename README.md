# BFT-DPoS

## Over View

The EOS BFT-DPoS in general works in following steps:

1. Initialize with 21 block producers
2. Randomly shuffle these producers to schedule the order of each of them to produce blocks
3. Each producer produces 6 blocks in her round
4. A block is always produced at some prefixed time stamp, i.e., every 0.5 second
5. If a producer receives a block, which is a ```complete``` block, she does the following
   * There are four possible states of a block ```incomplete```, ```complete```, ```validated```, ```irreversible```
   * If she finds the block invalid or with state ```incomplete```, she can discard it
   * She then resolves possible forks, i.e., there might be ambiguous blocks of the same timestamp. She chooses the block that completes the longest chain
   * If the block she receives is already in her database, she can discard it
   * She then sign the block header and broadcast block header and its signature (this is a confirmation of the block)
6. If a producer receives a confirmation of a block header,  
   * If the block sate is ```complete``` and the number of signatures is greater than ```2*21/3+1```, she promote the block to state ```validated```
   * Note that when a producer signs a block, she also confirms the whole chain that with this block as a header. For instance, if she confirms block 100, and confirms block 200, then she also confirms all the blocks between 100 and 200. In the EOS system, every producer confirms all the blocks before his producing cycle and put the number of blocks confirmed in the block header. A producer never double confirms a block
   * When receives a new block confirmation from a producer, if this block confirms a sequence of blocks, the first 1/3 of the ```validated``` blocks become ```irreversible```
7. If the current round is scheduled for a producer, she makes a block and broadcast it immediately. The block sate is then ```complete```
   * The producer at the end of a schedule proposes a new schedule to the block
8. A system contract is responsible for voting new producers
   * The schedule_producer_function is executed when a producer is starting a new block
   * The system contract modifies the ```global_state```, which is a subset of the state of the virtual machine
   * The producer validates the block based on the ```global_state```


The above procedure effective implements the two-message BFT system. For instance, each block is confirmed if it receives two message from a producer who confirms it directly or indirectly. Irreverisbility requires least 2/3 producers' confirmation. This is BFT.
