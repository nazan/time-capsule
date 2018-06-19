# TICTOC

## Application overview

In the text ahead, I will refer to this application as 'tictoc' short for TIme Capsule sTOCk. Now that we have naming taken care of.. I will try to explain this as brief as possible. Idea behind this application is very simple. Perhaps, by example would be the way to go.

Suppose you have a message that is best kept secret. Any message will get intercepted by one or more recipients eventually, otherwise there is no purpose for that message to exist in the first place. Say you have this message, but you would rather have it a secret for a certain duration of your choosing. At the expiry of that duration you would have it revealed to anyone who has some sort of reference (say a URI) to the encrypted form of the original message. An important point to note is, up until the expiry point, absolutely no one should be able to decipher, let alone read the contents of the original message. That includes the entity who runs/operates/maintains the repository of time-locked messages. In fact, this aspect of absolute secrecy is the very thing that will sell this product to potential users.

Three obvious characteristics of the solution to the above system requirements are:

1. It will hide the message by some sort of encryption. The encrypted cipher text will be either publicly available or accessible to a limited group of recipients chosen by the author of the message.

2. Storage of the message itself need not be handled by this system. This application is not of the storage variety as in Dropbox or Google Drive. This product has to do with keeping secrets for a limited period of time chosen by its users.

3. The storage of the unlock keys of the messages will be handled by a distributed network of computers. No single entity should hold the whole cipher text of any unlock key.


## A solution

In this particular solution, the following entities exist.

1. Reservor node - Works in unison with other nodes of its kind to store a symetric key which was used to encrypt an end-user message. More accurately, for a given symetric key, a single node will hold only a slice of the key. Once a reservoir node intercepts a slice of a key, it will keep that slice in its storage until it is called upon (by a controller node) to participate in the collaborative effort to reconstitute the whole key. This effort is triggered by a controller node, which is explained in more detail below. Each reservoir node has a reference to at least one controller node. It can send messages to the controller node via that reference. Each reservoir node also subscribes to the public channel 'ch0' to which the controller node may or may not broadcast a message on.

2. Controller node - This type of node has 2 more features in addition to all the features of a reservoir node. Only one controller node will be included in each instance of system deployment.
    1. Acts as a registry of reservoir nodes. Responds with a random list of reservoir nodes from its registry once called upon by an end-user for such a list.
    2. Storage of first slice of any symetric key will always be handled by the controller node. Which qualifies it to perform the second additional feature. That is, to trigger and coordintate the collaborative effort to reconstitute a symetric key once the expiry time of that key is detected. The result of the reconstitution will be collected and stored in the controller node itself.

Note that all communication described below must be performed over SSL to protect against various attack methods.

### Encryption

1. Bob wants to encrypt message M with sysmetric key SK for duration T.
2. Bob performs the encryption with a standard sysmetric key encryption algorithm giving him cipher text CT.
3. Bob splits symetric key SK to n slices SK1,SK2..SKn.
4. Bob asks tictoc controller node tictoc.ctrl0 for a list of n-1 reservoir nodes.
5. Bob intercepts the list of reservoir nodes RNp where p is the index from 1 to n-1. Note that RNp is a random list of reservoir nodes.
6. Bob prepends tictoc.ctrl0 to the list of RNp resulting RNq where q is 1 to n.
7. Bob generates a global ID, xyz.
8. Bob generates expiry point by adding T to current UTC time, giving ET.
9. Bob makes a hash (with a standard hashing algorithm like sha256) of SKn, giving #SKn and subsequently sends the following data message to RNq-1.
    > ["op":0, "key_gid":xyz, "slice_id":n-1, "total_slices":n, "slice":SKn-1, "next_slice_wuc_code":#SKn, "expires_on":ET]
10. Bob repeats step 9 until all slices are sent and stored at its corresponding tictoc node.
11. Bob stores cipher text CT at a third party website abc.com along with its associated sysmetric key gid xyz (note that symetric key itself is not included, only the gid of the key).
12. Bob discards symetric key SK.

### Decryption

1. Alice wants to read contents of message M after duration T.
2. Alice retrieves CT and xyz from abc.com.
3. Alice asks tictoc for symetric key with gid xyz by sending the following message. This request propagates and initially hits the controller node tictoc.ctrl0.
    > ["op":1, "key_gid":xyz]
4. tictoc.ctrl0 looks up its data store and given the current time is greater than value of field "expires_on" of record with gid xyz, tictoc broadcasts the following message WK on channel ch0 via the PUBSUB construct of the system.
    > ["op":2, "key_gid":xyz, "code":#SK1]
5. Each and every reservoir node intercepts the broadcast message and performs the following.
    1. Looks up its data store and find the record that has a matching "key_gid" of xyz.
    2. Performs the same standard hashing algorithm (like sha256) on the value of field "slice" of the matched record resulting #SKP.
    3. If #SK1 matches #SKP and current UTC time (as indicated by local machine) is greater than value of field "expires_on" of matched record then this reservoir node will send the following message to tictoc.ctrl0.
        > ["op":1, "key_gid":xyz, "slice_id":1, "slice":SK1, "code":#SK2]
        > which triggers tictoc.ctrl0 to manipulate this message as follows and broadcast it to ch0.
        > ["op":2, "key_gid":xyz, "code":#SK2]
        > which in turn will trigger step 5 to repeat until all slices of key_gid xyz is intercepted by tictoc.ctrl0 where it will be reconstituted and stored as a whole.
6. Alice awaits while the above process completes, concluding with her intercepting the whole symetric key SK.
7. Alice either asks abc.com to decrypt and display contents of M by sending it SK or given Alice knows how to perform the decryption algorithm Alice does the decryption herself without the need to make further calls to abc.com.