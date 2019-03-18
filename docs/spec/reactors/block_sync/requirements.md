# Blockchain Reactor

The Blockchain Reactor's high level responsibility is to enable peers who are
far behind the current state of the consensus to quickly catch up by downloading
many blocks in parallel, verifying their commits, and executing them against the
ABCI application.

Tendermint full nodes run the Blockchain Reactor as a service to provide blocks
to new nodes. New nodes run the Blockchain Reactor in "fast_sync" mode,
where they actively make requests for more blocks until they sync up.
Once caught up, "fast_sync" mode is disabled and the node switches to
using (and turns on) the Consensus Reactor.

# Requirements

We consider the model in which a correct node p (that runs in "fast_sync" mode) is connected to 
a set of nodes that are the peer set P. Nodes communicate by message passing, and the set P is dynamically 
changing. In order to provide any guarantee at least a single process in the set P should be a correct.
The "fast_sync" protocol should enable the process p to quickly catch up by downloading, verifying and executing
committed blocks. 

We want p to eventually (need to be more precise what it means!) terminate "fast_sync" protocol and enter consensus mode.
Note that p might be part of the active validator set (just lagging behind), so the network progress might depend on 
p catching up and switching to the consensus mode. In case a faulty process could be able to keep p in this mode
forever, the termination guarantees of the complete system are jeopardized. 

Lets assume that p started "fast_sync" mode at time t, and that the set P does not change for a period of time T.
If H is a maximal height a correct node from the peer set P is in at time t, then eventually 
(what if I assume synchronous system model? I could then bound T), p will download all blocks until at least height H, 
and will eventually switch to the consensus mode.

A protocol that provide "fast_sync" capability should fulfill the following properties:

Given a correct node p that runs in "fast_sync" mode, and his peer set P, the following should
eventually happen:

- let we denote with H maximal height a correct node from the peer set P is in. Then eventually, 
 p will download all blocks until at least height H, and will eventually switch to the consensus mode.



`Termination`: Given a validator set V, and two honest validators,
p and q, for each height h, and each round r,
proposer_p(h,r) = proposer_q(h,r)


 