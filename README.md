# CHORD

Project for the Distributed Systems (SDIS) class of the Master in Informatics and Computer Engineering (MIEIC) at the Faculty of Engineering of the University of Porto (FEUP). 

## Author

Rui Jorge Leão Guedes <br>
* Student Number: 201603854
* E-Mail: up201603854@fe.up.pt

## Description

Althought the project itself included many other aspects and consisted in a distributed replication system this repository contains only the chord related files. Each file is properly documented to allow an easy understanding. The following sections are retrieved from the project report which explains briefly how chord works.

## Scalability

For a scalable distribution of chunks across peers, at the design level, we used the chord DHT. Each peer, upon its initialization, joins the chord network by contacting another peer that already joined the network or, in the case of him being the network’s first peer, it simply creates the network. This is done throughout the Chord class that is responsible for creating the node associated to the peer itself and then join the network. 
All operations related to the management of the node in the network are encapsulated inside the chord package. Inside this package the class Node is responsible for initialize all needed structures to ensure the scalability desired.
After the peer node is created, the Chord class is responsible for joining it to the network. The process of joining the network varies. If the node is the network’s first peer it simply creates the network, otherwise it communicates with the contact peer in order to retrieve its successor and successfully join the network (chord.Node.joinNetwork,  lines 84-95). Having joined the network, the node must initialize other components in order to maintain its correctness in the network. The needed components are: chord.NodeListener, chord.Stabilizer, chord.FixFinger and chord.PredecessorPolling.
The chord.NodeListener, as previously stated, is used to establish a UDP connection with the other nodes in order to receive requests from them and reply accordingly.
The chord.Stabilizer guarantees that nodes are added to the chord network in a way that preserves reachability of existing nodes, even in the face of concurrent joins and lost and reordered messages. Basically, in other words, this component is responsible for periodically notifying its successor of its presence, to ensure that its successor has the right predecessor. 
The chord.FixFinger is responsible for periodically selecting a random entry of the finger table and updating its value accordingly to the network’s current state. As time passes this component ensures the finger table correctness and integrity.
Last but not least, the chord.PredecessorPolling periodically ensures that the node's predecessor is online. This way, it is possible to avoid the problem of considering invalid (non active) nodes as predecessors, which can then be successfully updated in the future by the chord.Stabilizer component.
Aside from these four main components, there are two other components which are also important for the chord protocol: chord.TransferKeys and chord.Status. The first one is responsible for maintaining the correctness of the chunks backup. After a node joins the network, it is possible that some chunks’ successor has changed, and therefore should be transferred to the new node. This is ensured by the chord.TransferKeys component. The second one, if initialized, periodically displays node information: IP address, server port and node finger table, to keep track of the network’s state and evolution.
All combined, these components ensure a reliable and robust implementation of a chord DHT.

To ensure that the file operations scale well with their size, chunk-wise operations such as their backup, download and deletion are executed concurrently. This concurrency was implemented using AsynchronousFileChannel from Java NIO, to read/write the chunks to a file asynchronously and CompletableFuture from the Java concurrent package. The CompletableFuture allows a Future to be marked as complete, therefore allowing the usage of a CompletionHandler with the AsynchronousFileChannel to handle the operations to be performed after reading or writing a chunk from/to the file, while keeping track of the completion of each chunk-wise task. This concurrent mechanism is implemented at peer.FileManager, more specifically at lines 194-226, 95-118 e 151-167.

## Fault-Tolerance

With the goal of avoiding single-points of failure we, having a decentralized design, implemented Chord’s fault-tolerance features. The implemented solution, for achieving the pretended goals, consists on replicating each chunk/key not only on its respective peer, where the chunk/key is supposed to be stored, but also in its predecessor and successor. This was possible since each peer, after joining the chord network, knows about both its successor and predecessor. 
At first, the idea was replicate the chunk/key at r successors and predecessors. Choosing r = log2(N) would suffice the lookup correctness even when 50% of the nodes present in the network fail. 
However, since fault tolerance was not one of the extra features considered for the grade ceiling, in order to avoid having to keep track of how many peers are currently in the network, we decided to consider r = 1, so, for each chunk/key backup, the chunk/key is replicated on both the predecessor and successor node. In the same manner, when the expected peer does not contain a chunk, its predecessor and successor nodes are used to retrieve the chunk. The information about the node where the chunk/key should be backed up and its successor and predecessor is given by the Query class (chord.Query). This query is performed in middleware.ChunkTransfer, in the backup, download and delete chunk methods.
