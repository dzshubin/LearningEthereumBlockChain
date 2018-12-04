# The mechanism of  tx syncing and broadcasting

## How tx syncing mechanism work?

Upon creating an Ethereum servicer object, we also create a **protocolManager** object.

```go
    if eth.protocolManager, err = NewProtocolManager(eth.chainConfig, config.SyncMode, config.NetworkId, eth.eventMux, eth.txPool, eth.engine, eth.blockchain, chainDb); err != nil {
        return nil, err
    }
```

And, in the body of the NewProtocolManage function, we initialize a batch of subprotocols which including the P2P protocol. What's worth to notice is the **Run** function which is responsible for handling message send by the new peer.

```go

    manager := &ProtocolManager{
        ......................
    manager.SubProtocols = make([]p2p.Protocol, 0, len(ProtocolVersions))
    for i, version := range ProtocolVersions {
        // Skip protocol version if incompatible with the mode of operation
        if mode == downloader.FastSync && version < eth63 {
            continue
        }
        // Compatible; initialise the sub-protocol
        version := version // Closure for the run
        manager.SubProtocols = append(manager.SubProtocols, **p2p.Protocol**{
            Name:    ProtocolName,
            Version: version,
            Length:  ProtocolLengths[i],
            **Run**: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
                peer := manager.newPeer(int(version), p, rw)
                select {
                case manager.newPeerCh <- peer:
                    manager.wg.Add(1)
                    defer manager.wg.Done()
                    return manager.handle(peer)
                case <-manager.quitSync:
                    return p2p.DiscQuitting
                }
            },
            NodeInfo: func() interface{} {
                return manager.NodeInfo()
            },
            PeerInfo: func(id enode.ID) interface{} {
                if p := manager.peers.Peer(fmt.Sprintf("%x", id[:8])); p != nil {
                    return p.Info()
                }
                return nil
            },
        })
    }

```


And, after finding a new peer connected, we will do a lot of prerequisite check on it and .... run the peer's P2P protocol's Run function. (skipped the whole process of connecting a new peer)


And, when we invoke the Run function, we, then invoke **manage.handle** function, then eventually, invoke **handleMsg** function which is the main loop of handling incoming message sent by remote peers.  For instance, received a new tx, we validated it, we add it to tx pool. we've analyzed before, you remember it, right? 

We take the process of handling new tx for example.

```go
func (pm *ProtocolManager) handleMsg(p *peer) error {
............................
    case msg.Code == TxMsg:
        // Transactions arrived, make sure we have a valid and fresh chain to handle them
        if atomic.LoadUint32(&pm.acceptTxs) == 0 {
            break
        }
        // Transactions can be processed, parse all of them and deliver to the pool
        var txs []*types.Transaction
        if err := msg.Decode(&txs); err != nil {
            return errResp(ErrDecode, "msg %v: %v", msg, err)
        }
        for i, tx := range txs {
            // Validate and mark the remote transaction
            if tx == nil {
                return errResp(ErrDecode, "transaction %d is nil", i)
            }
            p.MarkTransaction(tx.Hash())
        }
        pm.txpool.AddRemotes(txs)
..................................
}
```



Oops, I almost forget it! Before we enter handleMsg, we have to send all our tx in tx pool to the new peer.

```go
func (pm *ProtocolManager) syncTransactions(p *peer) {
    var txs types.Transactions
    pending, _ := pm.txpool.Pending()
    for _, batch := range pending {
        txs = append(txs, batch...)
    }
    if len(txs) == 0 {
        return
    }
    select {
    case pm.txsyncCh <- &txsync{p, txs}:
    case <-pm.quitSync:
    }
}
```




## How tx broadcasting mechanism work?

The analysis about it is quite easier than before. Let's take a look, come on!

```go
func (pm *ProtocolManager) Start(maxPeers int) {
    pm.maxPeers = maxPeers

    // broadcast transactions
    pm.txsCh = make(chan core.NewTxsEvent, txChanSize)
    pm.txsSub = pm.txpool.SubscribeNewTxsEvent(pm.txsCh)
    go pm.txBroadcastLoop()

    // broadcast mined blocks
    pm.minedBlockSub = pm.eventMux.Subscribe(core.NewMinedBlockEvent{})
    go pm.minedBroadcastLoop()

    // start sync handlers
    go pm.syncer()
    go pm.txsyncLoop()
}
```


First, we subscribe to the **NewTxsEvent**, so if someone(process) fire this event, we can receive the information from channel --**pm.txsCh**.  The new tx come from local, or remote maybe, we don't care.
```go
func (pm *ProtocolManager) txBroadcastLoop() {
    for {
        select {
        case event := <-pm.txsCh:
            pm.BroadcastTxs(event.Txs)

        // Err() channel will be closed when unsubscribing.
        case <-pm.txsSub.Err():
            return
        }
    }
}
```

As you can see, txBroadcastLoop just do the job broadcast tx to other peers. 

p.sync start **synchronizing** hashes and blocks from all the peers. 

What about pm.txsyncLoop? Oh, he takes care of the **initial transaction sync** for each new connection. 
```go
func (pm *ProtocolManager) txsyncLoop() {
    .......................................
    for {
        select {
        case s := <-pm.txsyncCh:
            pending[s.p.ID()] = s
            if !sending {
                send(s)
            }
        case err := <-done:
            sending = false
            // Stop tracking peers that cause send failures.
            if err != nil {
                pack.p.Log().Debug("Transaction send failed", "err", err)
                delete(pending, pack.p.ID())
            }
            // Schedule the next send.
            if s := pick(); s != nil {
                send(s)
            }
        case <-pm.quitSync:
            return
        }
    }
}
```

## Congratulation!