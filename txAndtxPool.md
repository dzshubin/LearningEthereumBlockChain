# Analysis of Tx and TxPool module

## SendTransaction through RPC API

If you are running full node and RPC is on, then you can process RPC call. for instance, we can send a tx through RPC. 

### func (s *PublicTransactionPoolAPI) SendTransaction(ctx context.Context, args SendTxArgs) (common.Hash, error)

below is the code(for simplity, I **ignore** some unnecessary code)
```
func (s *PublicTransactionPoolAPI) SendTransaction(ctx context.Context, args SendTxArgs) (common.Hash, error) {
    .........................

    signed, err := wallet.SignTx(account, tx, chainID)
    if err != nil {
        return common.Hash{}, err
    }
    return submitTransaction(ctx, s.b, signed)
}
```

Now, we know that you need to sign tx with your private key first, but that isn't the point of this article, so I'm not going to dive to it.

Let's take a look with submitTx .

### SubmitTransaction(ctx context.Context, b Backend, tx *types.Transaction) (common.Hash, error)

 First, we're going to send Tx, and print information to console.

blow is the code. again, **ignored** some code.
```
func submitTransaction(ctx context.Context, b Backend, tx *types.Transaction) (common.Hash, error) {
    if err := b.SendTx(ctx, tx); err != nil {
        return common.Hash{}, err
    }
    if tx.To() == nil {
        log.Info("Submitted contract creation", "fullhash", tx.Hash().Hex(), "contract", addr.Hex())
    } else {
        log.Info("Submitted transaction", "fullhash", tx.Hash().Hex(), "recipient", tx.To())
    }
    return tx.Hash(), nil
}

```

### SubmitTransaction -> SendTx ->AddLocal -> addTx
So, finally, we jump into addTx function.

What does this function do?

#### Point one
func (pool *TxPool) add(tx *types.Transaction, local bool) (bool, error)

1. if we already have this tx in the pool, return false and error

2. validateTx, check if blow rules are meet 

   + tx.size() > 32 * 1024
   + tx.value >= 0
   + tx.gas() < block gas limit
   + sign process is proper
   + drop non-local underpriced tx (lower than mimum gas price )
   + tx.nonce > currentNonce
   + currentBalance > tx.cost
   + tx.gas() > intrGas
3. if pool is full {count > pending + queue }
    + if the tx isn't local then check if the tx is underpriced than cheapest tx in the pool
    + otherwise, make room for this tx.
         + discard some tx in the pool.priced, not local underpriced tx
         +  remove tx
             + remove from pool.all
             + if tx is in the pool.pending[addr], ....
             + if tx is in the queue
<br/>
4. if this tx is replacing a pending one.
     + replace older tx and meanwhile remove from the pool. then **post NewTxsEvent**

5.  if the above condition doesn't satisfy. then enqueued it



#### point two
func (pool *TxPool) promoteExecutables(accounts []common.Address)
simply put, this method does the job that moves the tx in the queue to the pending.


Of course, there is some check the sanity of tx, such as the order of tx(nonce), gas limit and so on... 

blow is code 
```
func (pool *TxPool) promoteExecutables(accounts []common.Address) {
    // Track the promoted transactions to broadcast them at once
    var promoted []*types.Transaction

    // Gather all the accounts potentially needing updates
    if accounts == nil {
        accounts = make([]common.Address, 0, len(pool.queue))
        for addr := range pool.queue {
            accounts = append(accounts, addr)
        }
    }
    // Iterate over all accounts and promote any executable transactions
    for _, addr := range accounts {
        list := pool.queue[addr]
        if list == nil {
            continue // Just in case someone calls with a non existing account
        }
        // Drop all transactions that are deemed too old (low nonce)
        for _, tx := range list.Forward(pool.currentState.GetNonce(addr)) {
            hash := tx.Hash()
            log.Trace("Removed old queued transaction", "hash", hash)
            pool.all.Remove(hash)
            pool.priced.Removed()
        }
        // Drop all transactions that are too costly (low balance or out of gas)
        drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
        for _, tx := range drops {
            hash := tx.Hash()
            log.Trace("Removed unpayable queued transaction", "hash", hash)
            pool.all.Remove(hash)
            pool.priced.Removed()
            queuedNofundsCounter.Inc(1)
        }
        // Gather all executable transactions and promote them
        for _, tx := range list.Ready(pool.pendingState.GetNonce(addr)) {
            hash := tx.Hash()
            if pool.promoteTx(addr, hash, tx) {
                log.Trace("Promoting queued transaction", "hash", hash)
                promoted = append(promoted, tx)
            }
        }
        // Drop all transactions over the allowed limit
        if !pool.locals.contains(addr) {
            for _, tx := range list.Cap(int(pool.config.AccountQueue)) {
                hash := tx.Hash()
                pool.all.Remove(hash)
                pool.priced.Removed()
                queuedRateLimitCounter.Inc(1)
                log.Trace("Removed cap-exceeding queued transaction", "hash", hash)
            }
        }
        // Delete the entire queue entry if it became empty.
        if list.Empty() {
            delete(pool.queue, addr)
        }
    }
    // Notify subsystem for new promoted transactions.
    if len(promoted) > 0 {
        go pool.txFeed.Send(NewTxsEvent{promoted})
    }


.........................
}
```


### Relations with mining
What a long trip to here~! 

But, what's relations, you may ask, with mining, anyway?


In the initialization of the miner, we will new a worker instance, and, in the process of newWorker function. We would subscribe the **newTxEvent** with a channel, so if there is a new tx coming, we would know it.


blow is code 
```
func newWorker(config *params.ChainConfig, engine consensus.Engine, eth Backend, mux *event.TypeMux, recommit time.Duration, gasFloor, gasCeil uint64, isLocalBlock func(*types.Block) bool) *worker {
    worker := &worker{
        config:             config,
        engine:             engine,
        eth:                eth,
        mux:                mux,
        chain:              eth.BlockChain(),
        gasFloor:           gasFloor,
        gasCeil:            gasCeil,
        isLocalBlock:       isLocalBlock,
        localUncles:        make(map[common.Hash]*types.Block),
        remoteUncles:       make(map[common.Hash]*types.Block),
        unconfirmed:        newUnconfirmedBlocks(eth.BlockChain(), miningLogAtDepth),
        pendingTasks:       make(map[common.Hash]*task),
        txsCh:              make(chan core.NewTxsEvent, txChanSize),
        chainHeadCh:        make(chan core.ChainHeadEvent, chainHeadChanSize),
        chainSideCh:        make(chan core.ChainSideEvent, chainSideChanSize),
        newWorkCh:          make(chan *newWorkReq),
        taskCh:             make(chan *task),
        resultCh:           make(chan *types.Block, resultQueueSize),
        exitCh:             make(chan struct{}),
        startCh:            make(chan struct{}, 1),
        resubmitIntervalCh: make(chan time.Duration),
        resubmitAdjustCh:   make(chan *intervalAdjust, resubmitAdjustChanSize),
    }
    // Subscribe NewTxsEvent for tx pool
    **worker.txsSub = eth.TxPool().SubscribeNewTxsEvent(worker.txsCh)**
    // Subscribe events for blockchain
    worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
    worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)

    // Sanitize recommit interval if the user-specified one is too short.
    if recommit < minRecommitInterval {
        log.Warn("Sanitizing miner recommit interval", "provided", recommit, "updated", minRecommitInterval)
        recommit = minRecommitInterval
    }

    go worker.mainLoop()
    go worker.newWorkLoop(recommit)
    go worker.resultLoop()
    go worker.taskLoop()

    // Submit first work to initialize pending state.
    worker.startCh <- struct{}{}

    return worker
}
```

### Deal with newTxEvent
Through the above code, we know if there is a new tx, we would get it through the channel **worker.txsCh**

And... in the mainLoop(), we are keeping an eye on txsCh, if a new tx coming, we can do something.


```
func (w *worker) mainLoop() {
.................
        case ev := <-w.txsCh:
            // Apply transactions to the pending state if we're not mining.
            //
            // Note all transactions received may not be continuous with transactions
            // already included in the current mining block. These transactions will
            // be automatically eliminated.
            if !w.isRunning() && w.current != nil {
                w.mu.RLock()
                coinbase := w.coinbase
                w.mu.RUnlock()

                txs := make(map[common.Address]types.Transactions)
                for _, tx := range ev.Txs {
                    acc, _ := types.Sender(w.current.signer, tx)
                    txs[acc] = append(txs[acc], tx)
                }
                txset := types.NewTransactionsByPriceAndNonce(w.current.signer, txs)
                w.commitTransactions(txset, coinbase, nil)
                w.updateSnapshot()
            } else {
                // If we're mining, but nothing is being processed, wake on new transactions
                if w.config.Clique != nil && w.config.Clique.Period == 0 {
                    w.commitNewWork(nil, false, time.Now().Unix())
                }
            }
            atomic.AddInt32(&w.newTxs, int32(len(ev.Txs)))
}

```

### Seal a block
And. Finally, we will seal a block. we need to include transactions to a block. Where are the txs from?? pool.pending!

blow is code
```
// commitNewWork generates several new sealing tasks based on the parent block.
func (w *worker) commitNewWork(interrupt *int32, noempty bool, timestamp int64) {
..................

    // Fill the block with all available pending transactions.
    pending, err := w.eth.TxPool().Pending()
.......
}
```



<br/>
***Boom!** **We have got all the connections!***






