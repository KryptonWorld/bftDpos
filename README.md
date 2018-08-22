# BFT-DPoS

## Over View

The EOS BFT-DPoS in general works in following steps:

1. Initialize with 21 block producers
2. Randomly shuffle these producers to schedule the order of each of them to produce blocks
3. Each producer produces 6 blocks in her round
4. A block is always produced at some prefixed time stamp, i.e., every 0.5 second
5. If the current round is scheduled for a producer, she make a block and broadcast it immediately
6. If a producer receives a block, which is a pending block, she does the following
   * She first resolves the forks, i.e., there might be ambigrous blocks of the same timestamp. She chooses the block with the longest chain
   * She then sign the block and broadcast the signature of the block
   * If a block receives 2/3 * 21 signatures, she marks that as an "irreversable" block and push it to the local database 
7. The producer at the end of a schedule proposes a new schedule to the block
8. A system contract is responsible for vote new producers
   * The schedule_producer_function is executed when a producer is starting a new block
   * The system contract modifies the ```global_state```, which is a subset of the state of the virtual machine
   * The producer validates the block based on the ```global_state```

