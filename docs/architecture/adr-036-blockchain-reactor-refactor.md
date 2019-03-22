# ADR 036: Blockchain Reactor Refactor  

## Changelog

19-03-2019: Initial draft

## Context

The Blockchain Reactor's high level responsibility is to enable peers who are
far behind the current state of the consensus to quickly catch up by downloading
many blocks in parallel, verifying their commits, and executing them against the
ABCI application. The current architecture diagram of the blockchain reactor can be found here: 

![Blockchain Reactor Architecture Diagram](img/bc-reactor.png)

The major issue with this reactor is difficulty to understand the 
current design and the implementation, and the problem with writing tests.
More precisely, the current architecture consists of dozens of routines and it is tightly depending on the `Switch`, making writing unit tests almost impossible. Current tests require setting up complex dependency graphs and dealing with concurrency. Note that having dozens of routines is in this case  overkill as most of the time routines sits idle waiting for something to happen (message to arrive or timeout to expire). Due to dependency on the `Switch`, 
testing relatively complex network scenarios and failures (for example adding and removing peers) is very complex tasks and frequently lead to complex tests with not deterministic behavior ([#3400]).   

This resulted in several issues (some are closed and some are still open): 
[#3400], [#2897], [#2896], [#2699], [#2888], [#2457], [#2622], [#2026].  
Note that at the design level, the blockchain reactor share those problems with other reactors. Therefore, improving the blockchain reactor might serve as a guideline for other reactors to improve understanding, correctness, performance and testability. 

## Decision

To remedy these issues we plan a major refactor of the blockchain reactor. 
We suggest a different concurrency architecture where the core algorithm (we call it `Controller`) is extracted into a finite state machine. 
This simplifies understanding and it makes higher confidence in the implementation as it is much simpler writing unit tests. 
Furthermore, all networking related events are from the state machine just different input events,
which will make testing of complex networking scenarios much simpler.  

### Implementation changes

We make the following assumptions of the system: 

* a node is connected to a random subset of all nodes that represents its peer set. Some nodes are correct and some might be faulty. We don't make assumptions about ratio of  
  faulty nodes, i.e., it is possible that all nodes in some peer set are faulty. 
* we assume that communication between correct nodes is synchronous, i.e., if a correct node `p` sends a message `m` to a correct node `q` at time `t`, then `q` will receive
  message the latest at time `t+Delta` where `Delta` is a system parameter that is known. `Delta` is normally order of magnitude bigger than the real communication delay 
  (maximum) between correct nodes. Therefore if a correct node `p` sends a request message to a correct node `p` at time `t` and there is no the corresponding reply at time
  `t + 2*Delta`, then `p` can assume that `q` is faulty. Note that the network assumptions for the consensus reactor are different 
  (we assume partially synchronous network there). 
* we assume that all correct peers will periodically send us `StatusReport` message with its current height. Some of those messages might be lost or delayed, but we will
  eventually get more recent message that anyway supersedes the previous messages. We can think about these messages as some kind of heartbeat mechanism. `StatusReport` 
  messages are not explicitly sent by `Controller`, i.e., we assume that sending of these messages is done by lower level mechanism, for example by Reactor itself.     
    

The `Controller` can be modelled as a function with clearly defined inputs:

* `State` - current state of the node. Contains data about connected peers and its behavior, pending requests, received blocks, etc.
* `Event` - significant events in the network.

producing clear outputs:

* `State` - updated state of the node,
* `Message` - signal what message to send,
* `TimeoutTrigger` - signal that timeout should be triggered

```go
type Event int

const (
	EventUnknown Event = iota
    StatusReport
    BlockRequest
    NewBlock
    RemovePeer
    TimeoutInfo
)

type BlockRequest struct {
    Height int64,
    PeerID ID 
}

type NewBlock struct {
    Height        int64,
    Block         Block,
    Commit        Commit
    PeerID        ID,
    CurrentHeight int64
}

type StatusReport struct {
    PeerID ID,
    Height int64
}

type RemovePeer struct {
    PeerID ID,
    Height int64
}

type TimeoutInfo struct {
    PeerID ID,
    Height int64
}

type Message int

const (
	MessageUnknown Message = iota
    BlockRequestMessage    
)

type BlockRequestMessage struct {
    Height int64,
    PeerID ID 
}

type BlockResponseMessage struct {
    Height        int64,
    Block         Block,
    Commit        Commit
    PeerID        ID,
    CurrentHeight int64
}

type StatusReportMessage struct {
    Height int64
}

type TimeoutTrigger struct {
    PeerID ID,
    Height int64,
    Duration time.Duration
}

The Controller state machine can be in two modes (states): `FastSyncMode` when it is trying to catch up with the network by downloading committed blocks,
and `ConsensusMode` in which it executes Tendermint consensus protocol.  

type State int

const (
	StateUnknown State = iota
	FastSyncMode       
	ConsensusMode
)

Initially, a process is in `FastSyncMode`. Eventually, algorithm switches to the (`ConsensusMode`). 

A `Controller` has the following local state:

type ControllerState struct {
	Height              int64                // the first block that is not committed 
	State               State                // mode of operation of the state machine   
    PeerMap             map[ID]PeerStats     // map of peer IDs to peer statistics (info about outstanding block requests and peer current height) 
    MaxRequestPending   int64                // maximum height of the pending requests
    FailedRequests      List[int64]          // list of failed block requests
    PendingRequestsNum  int                  // number of pending requests
    Store               []BlockInfo          // contains list of downloaded blocks
    Executor            BlockExecutor        // store, verify and executes blocks 
}

type PeerStats struct {
	Height              int64
	PendingRequest      int64                 // it can be list in case there are multiple outstanding requests per peer
}

type BlockInfo struct {
    Block   Block   
    Commit  Commit
	PeerID  ID          // a peer from which we downloading the corresponding Block and Commit
}

func ControllerInit(state ControllerState, startHeight int64) ControllerState {
    state.Height = startHeight
    state.State = FastSyncMode
    state.MaxRequestPending = startHeight - 1 
    state.PendingRequestsNum = 0
}

func ControllerHandle(event Event, state ControllerState) (ControllerState, Message, TimeoutTrigger, Error) {
    msg = nil
    timeout = nil
    error = nil
    
    switch state.State {
        case ConsensusMode:
            switch event := event.(type) {
                case BlockRequest:
                    msg = createBlockResponseMessage(state, event)
                    return state, msg, timeout, error   
                default: 
                    error = "Only respond to BlockRequests while in ConsensusMode!"
                    return state, msg, timeout, error  
            }
        
        case FastSyncMode: 
            switch event := event.(type) {
                case BlockRequest:
                    msg = createBlockResponseMessage(state, event)
                    return state, msg, timeout, error
                
                case StatusReport:
                    if state.PeerMap[event.PeerID] does not exist {
                        peerStats = PeerStats {-1, -1}
                    }
                    if event.Height > peerStats.Height { peerStats.Height = event.Height }
                    if event.Height > state.MaxPeerHeight { state.MaxPeerHeight = event.Height }
                    if peerStats.PendingRequest == -1 {
                        msg = createBlockRequestMessage(state, event.PeerID, peerStats.Height)
                        if msg != nil { 
                            peerStats.PendingRequests = msg.Height 
                            state.PendingRequestsNum++
                            timeout = TimeoutTrigger { msg.PeerID, msg.Height, PeerTimeout }
                        }
                    }
                    state.PeerMap[event.PeerID] = peerStats
                    return state, msg, timeout, error   
                
                case RemovePeer:
                    if state.PeerMap[event.PeerID] exists {
                        pendingRequest = state.PeerMap[event.PeerID].PendingRequest
                        if pendingRequest != -1 { add(state.FailedRequests, pendingRequest) }
                        delete(state.PeerMap, event.PeerID) 
                        if state.MaxPeerHeight == state.PeerMap[event.PeerID].Height { state.MaxPeerHeight = computeMaxPeerHeight(state) }       
                    } else { error = "Removing unknown peer!" } 
                    
                case NewBlock:
                    if state.PeerMap[event.PeerID] exists {
                        peerStats = state.PeerMap[event.PeerID]
                        if peerStats.PendingRequest == event.Height {
                            peerStats.PendingRequest = -1
                            state.PendingRequestsNum--
                            if event.CurrentHeight > peerStats.Height { peerStats.Height = event.CurrentHeight }
                            if event.CurrentHeight > state.MaxPeerHeight { state.MaxPeerHeight = event.CurrentHeight }
                            state.Store[event.Height] = BlockInfo { event.Block, event.PeerID }
                            state = verifyBlocks(state)
                            if state.Height >= state.MaxPeerHeight - 1 { state.Step = ConsensusMode }  
                            msg = createBlockRequestMessage(state, event.PeerID, peerStats.Height)
                            if msg != nil { 
                                peerStats.PendingRequests = msg.Height 
                                state.PendingRequestsNum++
                                timeout = TimeoutTrigger { msg.PeerID, msg.Height, PeerTimeout }
                            }
                        } else { error = "Received Block from wrong peer!" } 
                    } else { error = "Received msg from unknown peer!" }
                    
                    state.PeerMap[event.PeerID] = peerStats
                    return state, msg, timeout, error       
                        
                case TimeoutInfo: 
                    if state.PeerMap[event.PeerID] exists {
                        peerStats = state.PeerMap[event.PeerID]
                        if peerStats.PendingRequest == event.Height {
                            add(state.FailedRequests, pendingRequest)
                            // TODO: Should I try to resend this request immediately to some peer that declare to have that block (before adding it to the failedRequests)?
                            // Other option is to wait on status msg to come and to engage with a peer at that point with retried blocks!
                            delete(state.PeerMap, event.PeerID)
                            state.PendingRequestsNum--
                            error = "Not responsive peer"    
                        }    
                    } else { state.PendingRequestsNum-- }
                    if state.NumberOfPendingRequests == 0 { state.Step = ConsensusMode }  
                    return state, msg, timeout, error
                
                default: 
                    error = "Received msg from unknown peer!"
                    return state, msg, timeout, error  
            }

    }
}

func createBlockResponseMessage (state ControllerState, event BlockRequest) BlockResponseMessage {
    msg = nil 
    if state.PeerMap[event.PeerID] does not exist {
        peerStats = PeerStats {-1, -1}
    }
    if state.Executor contains block with height event.Height and event.Height > peerStats.Height {
        peerStats = event.Height
        msg = BlockResponseMessage { 
            Height: event.Height, 
            Block: state.Executor.getBlock(eventHeight), 
            Commit: state.Executor.getCommit(eventHeight),
            PeerID: event.PeerID,
            CurrentHeight: state.Height - 1 
        }
    }
    state.PeerMap[event.PeerID] = peerStats
    return msg
}

func createBlockRequestMessage(state ControllerState, peerID ID, peerHeight int64) BlockRequestMessage {
    msg = nil 
    blockNumber = -1
    if exist request r in state.FailedRequests such that r <= peerHeight {
       blockNumber = r 
       delete(state.FailedRequests, r)       
    } else if state.MaxRequestPending < peerHeight {
       blockNumber = state.MaxRequestPending ++     
    }
    if blockNumber > -1 { msg = BlockRequestMessage { blockNumber, peerID } }
    return msg
}

func verifyBlocks(state State) State {
    done = false
    while state.Height < state.MaxPeerHeight and !done {
        block = state.Store[height]
        if block != nil {
            verified = verify block.Block using block.Commit
            if verified {
                execute block  
                state.Height++
                if state.PeerMap[block.PeerID] exists { 
                    if state.PeerMap[block.PeerID].Height > state.MaxPeerHeight { state.MaxPeerHeight = state.PeerMap[block.PeerID].Height }
                    // TODO: How to reset max peer height upon peer remove! I need to remember not just declared peer height 
                    // but also "confirmed" peer height. Computation of max peer height should take into account only "confirmed" peer heights?    
                }
            } else { 
                add(state.FailedRequests, height)
                // TODO: extract this into function remove peer!
                if state.PeerMap[block.PeerID] exists {
                    pendingRequest = state.PeerMap[block.PeerID].PendingRequest
                    if pendingRequest != -1 { add(state.FailedRequests, pendingRequest) }
                    delete(state.PeerMap, event.PeerID) 
                    if state.MaxPeerHeight == state.PeerMap[event.PeerID].Height { state.MaxPeerHeight = computeMaxPeerHeight(state) }       
                }
                done = true
            }   
        } else { done = true }
    }
    return state                 
}

    




## Status

> A decision may be "proposed" if it hasn't been agreed upon yet, or "accepted" once it is agreed upon. If a later ADR changes or reverses a decision, it may be marked as "deprecated" or "superseded" with a reference to its replacement.

{Deprecated|Proposed|Accepted}

## Consequences

> This section describes the consequences, after applying the decision. All consequences should be summarized here, not just the "positive" ones.

### Positive

### Negative

### Neutral

## References

> Are there any relevant PR comments, issues that led up to this, or articles referrenced for why we made the given design choice? If so link them here!

* {reference link}
