# Overview

Distributed database for multiple web apps.

+ **In process (embedded):** It will run inside of rust as async (probably tokio) tasks.
+ **Generic:** The actual data can be anything you want
+ **Immediately Consistent:** A write will not succeed until all available nodes have it
+ **In memory:** All of the data lives purely in memory (an external process can get snapshots)
+ **Dual TLS:** All nodes and external processes talk through Dual TLS and use a trusted CA to authenticate peers

## Benefits

+ Having it in-process means less overhead for data consistency locks and communication
+ Immediately consistent (AFAIK) makes node and split brain easier

## Tasks

### Node management task

#### Messages

+ **ping pong:** A ping pong process will run in each node, keeping track of known siblings
  and will have the following message types on its queue
  - The pong message will include the first ever and last write UUIDs.
    This way we'll be able to know which nodes are good for reading.
    1. If a sibling has an earlier first ever write UUID than us, we are *no good for reading*
    2. If a sibling has a later write than us, we are *no good for reading*
    3. We just track the *Good for reading* flag. It is the job of the [](<overview#Launcher task>) task to catch us up.
+ **Get sibling list:**
+ **Add node:** From the node scanner process
+ **Good for reading:** Returns true if we have the same amount or more data than our peers (or if we're the only peer)

### Node finder task

#### Activities

+ Nodes will be found by a DNS search, eg. node1.mydomain.com, node2.mydomain.com, etc.
+ When it finds a new node, it sends a message to the Node management task
+ It has no incoming messages

### Data consistency task

#### Overview

+ Data is kept consistent through tracking a collection of writes, and a snapshot
+ Each write and each snapshot has a uuid with time encoding incuded
+ The data consistency processes broadcast writes, and make sure all nodes receive it
+ It uses a read-write lock to keep its write access to the data when handling a message
+ We keep track of the first ever write uuid
+ Normally the data is read-accessible by the whole process (using the same read-write lock)
  - There should be function that provides a guarded read to the data

#### Messages

+ **Broadcast write:** broadcasts a write to all nodes and writes it in its own data too.
  - Also records the write in a list of writes since last snapshot
  - Write message contents are generic, and it's up to the library user to interpret them
  - Write messages are any change, and can be an insert, update, or delete
+ **Get snapshot:** Makes a snapshot of the data and sends it out (make sure to not copy the whole of memory)
  - Inculdes the last write ID in the snapshot
+ **Get writes:** If an external process has a snapshot alread and knows the last write in that snapshot,
  It can just ask for the newer writes.
+ **Snapshot created :** Once the external process has made a snapshot, it sends this message with the
  time encode uuid of the last write it had.
  - Our task then broadcasts this to all other nodes.
  - It wipes out all writes that are in the snapshot, but leaves writes that didn't get snapshotted yet
+ **Overwrite:** This is called when restoring a backup. It'll include a last write ID.
  - We delete all queued writes earlier than the last write id
  - We overwrite all our data with the incoming chunk
  - We apply all the write operations left over

### Launcher task

This is a short lived task that runs once at start.

#### Activities

+ When a new node is launched, it'll have zero data. This process will bring it up to speed.
+ All of the above processes are launched as per normal
+ We can get data either from a backup source (eg. S3) - this will be written by the library user
  or polling peers and requesting a node list
  or it'll just have no data to start with
+ We poll the peers (using [](<overview#Node finder task>)), at the same time as the backup source.
+ Depending on preferences, whichever responds first becomes the data source.
+ Once we get the whole data blob, we move it to the [](<overview#Data consistency task>) node via the **Overwrite** message
