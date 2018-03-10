## Cassandra  Single node disaster recovery and surprises
This repository has been created for sharing my experience while performing singe node disaster recovery.

Some surprises will come up during replacing a dead node in Cassandra cluster due to various reasons described below:

**Issue 1 : Unable to replace in case rest of your cluster restarts since node failure.**

Let's assume you have 3 node in your cluster node1, node2 and node3. your node3 got crashed and you want to replace node3 with brand new node node3_replace by following the standard process described in [documentation](https://docs.datastax.com/en/cassandra/3.0/cassandra/operations/opsReplaceNode.html) .

So you followed each steps carefully and updated cassandra-env.sh of node3_replace

    JVM\_OPTS="$JVM\_OPTS -Dcassandra.replace_address=<address of dead node node3>

and starting the ndoe3_replace node.

Now here, at this point, you will get your first surprise with below exception  in case node1 and node2  got  somehow restarted since the time  node3  failure.

    ERROR \[main\] 2017-11-15 13:57:59,043 CassandraDaemon.java:583 - Exception encountered during startup
    java.lang.RuntimeException: Cannot replace_address /10.61.216.133 because it doesn't exist in gossip

**Issue 2 : Unable to replace as while resolving above issue 1 you ignored a word in resolution provided .**

Your node3_replace node is not able start  and you are confused. So you googled and found out a good solution on Stackoverflow [link](https://stackoverflow.com/questions/23982191/cant-replace-dead-cassandra-node-because-it-doesnt-exist-in-gossip).  This link explains the reason that the cluster lost the gossip information of dead node3 and suggest a 3 steps resolution:

1.  Discover the node ID of the failed node (from another node in the cluster)
     nodetool status | grep DN
1.  And remove it from the cluster:
    nodetool removenode (node ID)
1.  Now you can clear out the data directory of the failed node, and bootstrap it as a brand-new one.

So you followed above steps quickly as you want to get rid of this issue asap and another surprise for you in case you ignored the word **brand-new** in step 3 of the resolution step :)

    java.lang.RuntimeException: cassandra does not use new-style tokens!

You are more confused now and again googling but this time even after spending some time, you don’t find any site which talks about this issue, now here your confusion becomes frustration.

Actually what went wrong here is **you forgot to comment out replace_address  from cassnadra-env.sh of node3_replace node.**

**Issue 3: Issue 2 + An another node in your cluster of your node was down**

This is the second version of Issue 2 but this time node1 was also down so you ignored node1 and you were more interested in starting up node3_replace.

But here you are surprised with another Cassandra exception encountered during startup of node

    java.lang.RuntimeException: A node required to move the data consistently is down (/<node1 ip>). If you wish to move the data from a potentially inconsistent replica, restart the node with -Dcassandra.consistent.rangemovement=false

This time you did not google as the exception message is quite explanatory. so you check node 1 was down and you started node1 thinking that this will resolve the problem but now you are again got back to Reason 2

**Lesson learned :**  Don’t ignore a word while following an administrative steps :)

**Summery:**
There is conceptual difference between replacing a dead node and adding a brand new node. In replace the tokens will be assigned to new node  while in addition there will a token shuffling.

The administrative steps of adding a new node has nothing to do with  replace_address (this is purely for replacing a dead node) and unfortunately cassandra does not ignore this parameter if you put during adding a new node.

One last point : Don’t forget to run nodetool clean after adding a new node.


