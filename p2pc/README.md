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
all the participants in a transaction. Rather it delegates the communication in
a tree structure. As you can see in this really cool diagram.
![t2pc](/imgs/t2pc.png). In T2PC then the controller sends messages to it's
children (in this case Deligates A & D) and they send them to their children
(and they send them to their children, etc...) and then wait for the response
from all their children before responding. If any node receives an abort from
it's children or itself needs to abort it will reply with an abort message.
Otherwise it replies with a ready message. The controller collects the response
from all it's chilren (which implies the state of their entire sub-tree) and
then decided to commit or abort.

D2PC builds on this idea by suggesting that the commit controller should be
dynamically selected among the various nodes of the tree as a way of increasing
the through put of the messages through the system. The idea is the same a T2PC
except that as soon as a node is ready and has recieved a ready (or aobrt) 
message from all then one of it's neighbors it will race it's ready message 
onto the neighbor that hasn't yet responded. In this way the center of control
moves toward the slowest part of the tree. If a node has received messages from
all of it's neighbors when it final is ready then it becomes the commitment
controller and decides whether the transaction should be commited (if all the
message it received where ready message and it is ready) or to abort (the the
CC must abort or it recieved an abort message). For people who like diagrams
here's a diagram. ![d2pc](/imgs/d2pc.png) As you can see in the diagram because
Delegate B was much slower than the other delegates in responding Delegate A
was eventually chosen as the node to decide if the transaction for be
committed. Now there are some issues with D2PC one of which is glare (which can
be show using math will occur in one location and only one location in the
tree, if you want to see the math you have to 
[buy](http://link.springer.com/chapter/10.1007/3-540-58907-4_14) Yoav Raz's
paper on the subject). This can be seen in the following diagram. ![d2pc
glare](/imgs/d2pc_glare.png) In this diagram we can see that the Original
Controller and Delegate A both try to send ready message to each other at about
the same time. In this case you must go through a process to decide whether the
Original Controller or Delegate A will be the commit controller and then the
commitment decision can be made and the transaction finished.

##P2PC

Now you may be asking yourself, why does there need to be a single commitment
controller? If a node has received a message from all of it's neighbors doesn't
a node have enough information to decide if the commit should be finalized or
not? Why not have both node be contollers in the glare case? In fact, lets do
away with the need for explicite controllers altogether! If you were thinking
anything like that then great (if not, you should be). This is where P2PC comes
in.

The protocol algorithm is pretty simple and is as follows

###Algorithm

####Commitment Procedure
* A node sends an explicit PREPARE message when applies to its neighbors, if required
* A node in the prepared state that has received a READY message from all its neighbors but one, sends to that neighbor a READY message and enters the ready state.
* A node in the prepared state that has received a READY message from all its neighbors sends a READY message to all it's neighbors, commits, and enters the committed state.
* A node in the ready state that receives a READY message from a neighbor sends a READY message to all other neighbors, commits, enters the committed state, and sends a COMMITTED message to that neighbor and all neighbors who sent COMMITTED messages during the commit process.
* A node in the committed state that has received COMMITTED messages from all it's neighbors enters the forgotten state.

####Abort Procedure
* A node that encounters an error during the prepare phase sends an ABORT message to all it's neighbors and enters the aborted state
* A node in the prepared or ready state that has received an ABORT message from a neighbor sends an ABORT message to all other neighbors, aborts, enters the aborted state, and sends an ABORTED message to that neighbor and all neighbors who sent ABORTED messages during the abort process.
* A node in the aborted state that has received ABORTED messages from all it's neighbors enters the forgotten state.
