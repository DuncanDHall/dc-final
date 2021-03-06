# Technical Breakdown and Documentation
This technical detail was last updated May 16th, 2017

*This document will be transferred to inline comments*

## AODVEvent

This class represents any of the AODVEvents that occur in our system. This currently includes route request messages (RREQ), route reply messages (RREP), and data transfer messages. Each AODV event takes place across one edge in the network at a specific time.

#### Fields

| Type   | Name        | Descripion |
| ------ | ----------- | ---------- |
| int    | time        | The timestep at which this event occurs |
| int    | msgType     | The type of AODVEvent: 0. Data, 1. RREQ, 2. RREP |
| Node   | from        | The node on the sending side of the edge |
| Node   | to          | The node on the receiving side of the edge |
| Node   | source      | The node which was the origin of the message (the sender in the case of a RREQ or Data message, the receiver in the case of a RREP message) |
| Node   | destination | The node which is the destination of the message (the receiver in the case of a RREQ or Data message, the sender in the case of a RREP) |
| String | floodID     | The unique string associated with the pathfinding flood |
| String | message     | Only relevent for Data messages, otherwise empty string "" |
| int    | hopCount    | Tracks the number of edges which the message has propagated along |


#### Methods

| Return Type | Name            | Arguments and Types | Description |
|-------------|-----------------|---------------------|-------------|
| constructor | AODVEvent       | int time, int msgType, Node from, Node to, Node source, Node destination, String floodID, String message, int hopCount | Basic constructor with arguments for all fields |
| constructor | AODVEvent       | AODVEvent e, Node from, Node to | Secondary constructor to ease creation of new events prpagated from existing ones. All fields are copied from AODVEvent e except Node from and Node to which are overwritten, and int time and int hopcount which are incremented by 1 |
| String      | toString        | *empty* | Overridden method for easier identification |
| boolean     | equals          | Object o | Overridden method for comparisons. The logic present here is likely redundant with compareTo, and should invoke that method instead |
| int         | compareTo       | Object o | Overridden method for use within a legacy priority queue in AODVhelper.expandEvents. |
| *various*   | *getters*       | *empty*  | Getters for all private fields. |



## AODVHelper

This class handles all AODV routing algorithm processing and simulation.

#### Methods

| Return Type | Name            | Arguments and Types | Description |
|-------------|-----------------|---------------------|-------------|
| AODVEvent[] | expandEvents    | RawEvent[] eventsTable, Graph<Node, Link> network, HashMap<Integer, Node> nodeLookup | Relates a set of RawEvents to the network graph using a nodeLookup table (RawEvents encode node id's which are integers, and this allows ascociation with the actual node objects). The expansion of these raw events into AODVEvents is computed with discrete time steps. At each new time step, new AODVEvents are generated for the next round based on the current AODVEvents (the logic here could be modified to get different AODV algorithem varients like AOMDV), and new AODVEvents are also generated by polling the RawEvents eventsTable. |



## CSVHelper

This class is designer to compartmentalize the saving of a CSV file from an array of AODVEvents.

#### Methods
| Return Type | Name            | Arguments and Types | Description |
|-------------|-----------------|---------------------|-------------|
| void        | writeCSV        | AODVEvents[] events | Saves the AODVEvents array to a csv on disk for manual inspection. Currently this is always as "./out.csv", and the format of the csv is not ideal. It should be re-written to be better formatted and also better filed. |


## Global

This class is the starting point for the program when it runs, calling on all other classes as needed. It's methods (main and otherwise) outline the structure of the program.

#### Fields

| Type                   | Name        | Descripion |
| ---------------------- | ----------- | ---------- |
| Graph<Node, Link>      | network     | A JUNG graph object representing the network nodes and edges |
| HashMap<Integer, Node> | nodeLookup  | Maps Node.id to Node objects so that RawEvents only reference nodes by id |


#### Methods

| Return Type       | Name                        | Arguments and Types | Description |
|-------------------|-----------------------------|---------------------|-------------|
| *constructor*     | Global                      | int numNodes, int numEdges, boolean dynamic | A constructor for Global objects which generates a random parameterized network. |
| *constructor*     | Global                      | Graph<Node, Link> network | A secondary constructor for a Global state object which allows external construction of the network. Useful for testing |
| RawEvent[]        | generateDataTransfersEvents | int n, int e, int interval | Generates an array of e RawEvents between random selections from n nodes with a time interval |
| Graph<Node, Link> | generateSparseGraph         | int n, int s | Used by the first constructor to generate a random connected graph with n nodes and s edges. Note: the graphs generated don't accurately represent an ad hoc network because there is no location tied to the nodes. Edges are added at random to the graph after all nodes have been connected |
| void              | main                        | String[] args | This is the main method of the program. Invoke this with `java Global` from the command line, and then follow the prompts. First the program generates a random connected graph, and prompts the user for the number of nodes, then the average degree of each node (the number of edges generated is floor(degree * nodes / 2)). Make sure that the input degree is between 2(n-1)/n and n(n-1)/2 because there are no catches implemented for input degrees that will yield too many or too few edges for a connected graph. The next two prompts have to do with the data messages to be sent. The program will ask for the number of desired messages, and the desired interval between messages (shorter intervals/larger networks will result in messages from different pathfinding sessions taking place concurrently, but messages from different sessions do not interfere with one another). A Global starting state is then created from those parameters, raw events generated, and those events are expanded into a full AODVEvents array, and logged to a csv file. A basic implementation of the Visualizer class allows the viewing of the topology of the network generated, though nodes are currently unlabeled, and the image is not animated with the messages as we aim for eventually. |



## Link

This class is hardly used, though having it around allows for further expansion of edge functinality with minimal change across the code base.

#### Fields

| Type | Name    | Descripion |
| ---- | ------- | ---------- |
| int  | weight  | A representation of the length of the edge (assumed to be proportional to the time a message would take to traverse it). This is not currently used anywhere, but in future implementations we would like to see messages taking varied amounts of time traverse edges |
| int  | id      | An unused identification of the link. |



## Node

This class represents a node in the network. It contains critical functionality to the AODV algorithem because of it's table which allows distributed storage of established paths from one node to another. It also includes a private helper class TableEntry which stores this path data (Node destination, Node nextHop, and int hopcount).

#### Fields

| Type                      | Name           | Descripion |
| ------------------------- | -------------- | ---------- |
| HashMap<Node, TableEntry> | routingTable   | The table which stores routing information on all paths which this node is a part of. This maps destination Node to TableEntry (see specs above) |
| int                       | id             | An integer id which uniquely identifies the node |
| int                       | floodID        | An integer id which is incremented every time the node initiates a path discovery. This value is combined with the node id to create a unique flood id for each new path discovery process |
| HashSet<String>           | knownFloods    | Keeps track of which path discovery RREQ floods the node has alread heard from so that it only propagates the first RREQ it receives |
| HashMap<String, String>   | unsentMessages | Keeps track of data messages that are queued to be sent when their path discover completes |

#### Methods

| Return Type       | Name                 | Arguments and Types | Description |
|-------------------|----------------------|---------------------|-------------|
| *constructor*     | Node                 | int id              | Basic constructor for a node |
| String            | genFloodID           | String message      | Generates a unique RREQ flood id by incrementing the floodID field and combining it with the node id. The message is stored in unsentMessages for later retreival |
| boolean           | knowsFlood           | String id           | Return true if the node already has received a RREQ with the same floodID (using knownFloods), otherwise remember the floodID and return false |
| void              | addTableEntry        | Node destination, Node nextHop, int hopCount | Creates a new table entry for the destination recording nextHop and hopCount. Inline comment is legacy and is no longer relevent |
| Node              | getNextHop           | Node destination    | Looks for established path to the destination node and returns the recorded nextHop, or null if no entry is present for the destination node |
| int               | getHopCount          | Node destination    | Performs a similar lookup to getNextHop, but instead returns hopCount |
| String            | getMessageForFloodID | String floodID      | Retreives the unsent message corresponding to the specified floodID. This could be changed to remove the corresponding entry from unsentMessages entirely |
| boolean           | equals               | Object o            | Overridden method for custom comparison of objects |
| String            | toString             | *empty*             | Overridden method for easier identification |



## RawEvent

This class represents a raw event (a sort of seed message) which requires a path to be generated. For example: node 1 sends node 5 message 'hi there, 5'. This would then be expanded into AODVEvents which represent the path discovery process, the reply process, and then finally the data transfer (of 'hi there, 5') across the network.

#### Fields

| Type      | Name           | Descripion |
| --------- | -------------- | ---------- |
| int       | time           | The time at which the RawEvent is initiated |
| int       | nodeFrom       | The id of node which is the sender of the message |
| int       | nodeTo         | The id of node which is the receiver of the message |
| String    | message        | The actual message being sent |

#### Methods

| Return Type       | Name                 | Arguments and Types | Description |
|-------------------|----------------------|---------------------|-------------|
| *constructor*     | RawEvent             | int time, int nodeFrom, int nodeTo, String msg | A basic constructor which assigns values to all fields |
| *various*         | *getters*            | Getter methods for all fields |



## Visualizer

A class to which all visualization functionality is delegated.

#### Methods

| Return Type       | Name                 | Arguments and Types | Description |
|-------------------|----------------------|---------------------|-------------|
| void              | visualizeNetwork     | Graph<Node, Link> network | Creates a static image of the input network and displays it to the user. This should be rewritten to include node labels |
| void              | visualize            | AODVEvents[] fullEvents | Not implemented yet, but this is the method that will provide the full animated version of the AODVEents. It will also need to be modified to take in the network topology |




## Tests

We included a couple of tests of the AODV algorithm in this repository to exemplify and verify the program. Below are summaries of each test file.


### GraphTests

This test group only has one test and is not automatic. It is solely a demonstration of the graph visualization technique the visualizer uses currently.


### LinearNetworkTests

This test group includes tests for linear networks only.


#### makLinearNetwork(int numNodes)

This helper function creates a linear network with the specified number of nodes, and nothing more!


#### testLinearNetwork1 (Test)

This first test is for only two nodes connected by a single edge, with one RawEvent expanded into 3 AODVEvents.


#### testLinearNetwork2 (Test)

This test is identical to the first, but with an extra node in between, with one RawEvent expanded into 6 AODVEvents.


#### testLinearNetwork3 (Test)

This test used the same topology as the second, but the RawEvent is sent from the first node to the second, with the third never recieving any messages (one RawEvent, 3 AODVEvents).


