#P2PC: A better distributed transaction protocol.

##Intro

If you’re working with, learning about or have ever had any interest in real
time distributed transactions then you have almost certainly heard about the
Two Phase Commitment protocol (2PC). That’s because 2PC is an essential part
of coordinating a transaction distributed across multiple machines. There have
been numerous white papers written on 2PC and it's variants as people try to
get faster transaction without sacrificing reliability. Today I'm going to
describe a new improvement on D2PC (Dynamic Two Phase Commitment), which is
itself an optimized variant of T2PC (Tree Two Phase Commitment), that I'm 
calling P2PC (Peer-to-Peer Two Phase Commitment).
 
##2PC

First things first. What is this 2PC garbage you keep talking about? Simply put
2PC is about managing transactions. By transaction we mean an action that will
change the state of a system if successful and can be reverted to the original
state before the transaction if it is unsuccessful and aborts. In the 2PC we
inform all the participants in the transaction that we're going have a
distributed transaction and then need to prepare for it. The various machines
and systems that will be involved in the transaction do all the things they
need to to prepare (set internal state, get locks on required system resources,
snapshot their state in case they need to rollback, etc...) and then reply with
a ready message. When the machine cooridinating the work gets replies from
everyone it will send out a commit message informing all participants that they
are going forward with the transaction and that it should be finalized. If for
whatever reasons one of the participants can't process the transaction (can't
get a lock on the transaction file, out of memory, dog ates it's homework,
etc...) then it sends an abort message rather than a ready message and the
controller will inform all participants that they should abort the transaction
and rollback to the previous state.

##T2PC & D2PC

This bring us to a point where we can discuss T2PC and D2PC. T2PC is a common
implementation of 2PC where the controller doesn't directly communicate with
all the participants in a transaction. Rather it delegates the communication is
a tree structure. As you can see in this really cool diagram.
![t2pc](/imgs/t2pc.png)
