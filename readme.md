# TICTOC

## Application overview

In the text ahead, I will refer to this application as 'tictoc' short for TIme Capsule sTOCk. Now that we have naming taken care of.. I will try to explain this as brief as possible. Idea behind this application is very simple. Perhaps, by example would be the way to go.

Suppose you have a message that is best kept secret. Any message will get intercepted by one or more recipients eventually, otherwise there is no purpose for that message to exist in the first place. Say you have this message, but you would rather have it a secret for a certain duration of your choosing. At the expiry of that duration you would have it revealed to anyone who has some sort of reference (say a URI) to the encrypted form of the original message. An important point to note is, up until the expiry point, absolutely no one should be able to decipher, let alone read the contents of the original message. That includes the entity who runs/operates/maintains the repository of time-locked messages. In fact, this aspect of absolute secrecy is the very thing that will sell this product to potential users.

Three obvious characteristics of the solution to the above system requirements are:

1.  It will hide the message by some sort of encryption. The encrypted cipher text will be either publicly available or accessible to a limited group of recipients chosen by the author of the message.

2.  Storage of the message itself need not be handled by this system. This application is not of the storage variety as in Dropbox or Google Drive. This product has to do with keeping secrets for a limited period of time chosen by its users.

3.  The storage of the unlock keys of the messages will be handled by a distributed network of computers. No single entity should hold the whole string of any unlock key.

## A solution

In this particular solution, the following entities exist.

1.  Reservoir node - Works in unison with other nodes of its kind to store a symetric key which was used to encrypt an end-user message. More accurately, for a given symetric key, a single node will hold only a slice of the key. Once a reservoir node intercepts a slice of a key, it will keep that slice in its storage until it is called upon (by a controller node) to participate in the collaborative effort to reconstitute the whole key. This effort is triggered by a controller node, which is explained in more detail below. Each reservoir node has a reference to the controller node. It can send messages to the controller node via that reference. Each reservoir node also subscribes to the public channel 'ch0' to which the controller node may or may not broadcast a message on.

2.  Controller node - This type of node has 2 more features in addition to all the features of a reservoir node. Only one controller node will exist in each instance of system deployment.
    1.  Acts as a registry of reservoir nodes. Responds with a random list of reservoir nodes from its registry once called upon by an end-user for such a list.
    2.  Storage of first slice of any symetric key will always be handled by the controller node. Which qualifies it to perform the second additional feature. That is, to trigger and coordintate the collaborative effort to reconstitute a symetric key once the expiry time of that key is detected. The result of the reconstitution will be collected and stored in the controller node itself.

Note that all communication described below must be performed over SSL to protect against various attack methods.

### Encryption

1.  Alice wants to encrypt message M with sysmetric key SK for duration T.
2.  Alice performs the encryption with a standard sysmetric key encryption algorithm giving her cipher text CT.
3.  Alice splits symetric key SK to n slices SK1,SK2..SKn.
4.  Alice asks tictoc controller node tictoc.ctrl0 for a list of n-1 reservoir nodes.
5.  Alice intercepts the list of reservoir nodes RNp where p is the index from 1 to n-1. Note that RNp is a random list of reservoir nodes.
6.  Alice prepends tictoc.ctrl0 to the list of RNp resulting RNq where q is 1 to n.
7.  Alice generates a global ID, xyz.
8.  Alice generates expiry point by adding T to current UTC time, giving ET.
9.  Alice makes a hash (with a standard hashing algorithm like sha256) of SKn, giving #SKn and subsequently sends the following data message to RNq-1.
    > ["op":0, "key_gid":xyz, "slice_id":n-1, "total_slices":n, "slice":SKn-1, "next_slice_wuc_code":#SKn, "expires_on":ET]
10. Alice repeats step 9 until all slices are sent and stored at its corresponding tictoc node.
11. Alice stores cipher text CT at a third party website abc.com along with its associated sysmetric key gid xyz (note that symetric key itself is not included, only the gid of the key).
12. Alice discards symetric key SK.

### Decryption

1.  Bob wants to read contents of message M after duration T.
2.  Bob retrieves CT and xyz from abc.com.
3.  Bob asks tictoc for symetric key with gid xyz by sending the following message. This request propagates and initially hits the controller node tictoc.ctrl0.
    > ["op":1, "key_gid":xyz]
4.  tictoc.ctrl0 looks up its data store and given the current time is greater than value of field "expires_on" of record with gid xyz, tictoc broadcasts the following message WK on channel ch0 via the PUBSUB construct of the system.
    > ["op":2, "key_gid":xyz, "code":#SK1]
5.  Each and every reservoir node intercepts the broadcast message and performs the following.

    1.  Looks up its data store and find the record that has a matching "key_gid" of xyz.
    2.  Performs the same standard hashing algorithm (like sha256) on the value of field "slice" of the matched record resulting #SKP.
    3.  If #SK1 matches #SKP and current UTC time (as indicated by local machine) is greater than value of field "expires_on" of matched record then this reservoir node will send the following message to tictoc.ctrl0.

        > ["op":1, "key_gid":xyz, "slice_id":1, "slice":SK1, "code":#SK2]

        which triggers tictoc.ctrl0 to manipulate this message as follows and broadcast it to ch0.

        > ["op":2, "key_gid":xyz, "code":#SK2]

        which in turn will trigger step 5 to repeat until all slices of key_gid xyz is intercepted by tictoc.ctrl0 where it will be reconstituted and stored as a whole.

6.  Bob awaits while the above process completes, concluding with him intercepting the whole symetric key SK.
7.  Bob either asks abc.com to decrypt and display contents of M by sending it SK or given Bob knows how to perform the decryption algorithm Bob does the decryption himself without the need to make further calls to abc.com.

## What if a node goes down unexpectedly?

Where a node disappears unexpectedly, redundancy is the proposed solution. Any reservoir node of the system can go down at any time, and for this reason there should be a graceful recovery mechanism in place. Note that, since reservoir nodes are run by the community, the uptime of those servers are at the discretion of whoever maintains/owns it.

In summary the redundancy mechanism involves copying each slice of each key to a set number of other reservoir nodes. Let's say each slice will be copied over to 2 other nodes resulting 3 different nodes (holding each and every slice of each and every key). Those 3 nodes will situate themselves in a ring formation; each of those 3 nodes would have a pointer to the next, and the last in the line will point to the first, thereby closing the loop. All three would periodically check whether its next is alive. Any node that finds its 'next' to be unresponsive, would initiate replication of itself which would conclude patching up the broken link in the ring formation.

### Replication

#### Assumptions

- Two redundant copies will be made for each slice of each key. Each slice of each key would be held by 3 different nodes in total. For simplicity and brevity I assume 3 redundant copies although 3 is not an ideal number of copies since its crucial that each redundant node should only be aware of part of the ring formation (as opposed to the entire ring).
- Pointers that make up the ring formation are:
  - Each node will have a pointer to its immediate previous node.
  - Each node will have a pointer to its "next" node.
  - Each node will have a pointer to its "next next" node.
- Any slice of any key has the ability to make a redundant copy of itself onto another node (suggested by the controller node).
- Each redundant node will reveal its "next" if one of 2 things satisfy.
  - Asking node is the "previous" of self node.
  - Or, asking node has a verification code that matches.

#### Pointers table

| Node | Previous | Next | Next Next |
| ---- | -------- | ---- | --------- |
| R1   | R3       | R2   | R3        |
| R2   | R1       | R3   | R1        |
| R3   | R2       | R1   | R2        |

#### Relication steps

If any node in the ring of redundant nodes finds that its next is unresponsive then it will initiate the replication process. Replication would proceed as follows. Let's assume node R2 is down for some reason.

1.  R1 gets a blank node by requesting for it from the controller node (tictoc.ctrl0).
2.  R1 calls and asks "next next" node (R3) to prepare for the replication of its "previous" node (R2). R1 generates a random key denoted by ABC and hands it over to R3.
3.  Assuming R1 is the caller, "next next" of R1 which is R3 will check if its "previous" is responsive, if so then invalidate the replication and delete the key ABC given at step 2. Process ends here if "previous" is alive.
4.  R1 tells blank node:
    1.  Here's my "next next", set it as your "next".
    2.  Here's me, set it as your "previous".
    3.  To ask your "next" to give its "next"; here's a verification code ABC.
5.  Blank node sets as its "next next", whatever success response given by its "next" R3.
6.  R1 transmits its data (includes slice and next slice wake up code) to blank node and asks to set that as its data.
7.  R1 sets its "next" as the newly set up blank node.
