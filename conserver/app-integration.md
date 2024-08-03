---
description: Using the Conserver with your Application
---

# App Integration

Using the Conserver with your application is straightforward:

* Setup your application database as a "storage" for the conserver.   As it process vCons, the vCons will send copies of the vCons into the storage, similar to a database follower.
* When you create, update and delete vCons, use the Conserver REST API.  This assures that the vCons will be processed, tracked and protected like the rest.
* For real time processing, you can use the use the WebHook link to notify your application of new vCons in the database.  Also, since the conserver is a client to your application's database, all of the native notifications, such as Mongo OP-Log tails, REDIS key space events or S3 events can also be used.

<figure><img src="../.gitbook/assets/Untitled (12).jpg" alt=""><figcaption><p>Application Integration with the Conserver</p></figcaption></figure>

### Application Integration, Step by Step

1. vCons are sent to the Conserver using the REST API to first create new vCons.  These vCons will now be saved in the REDIS database using the vcon: keyspace.   For instance, a vCon with a uuid of _018796f4-ece7-8f28-9dd8-dd37220d739c_ will be stored in REDIS JSON with a key of _vcon:018796f4-ece7-8f28-9dd8-dd37220d739c._
2. To process this vCon using a chain, add this vCon UUID to it using the REST API.  These vCons uuid will be added to the list with matching name and processed in the main event loop.&#x20;
3. Once the chain has finished processing the vCon, it will put the vCon into the storages configured for that chain.  In the diagram above, it will be stored in both S3 and MongoDB.
4. REDIS is used as a caching system, and vCons eventually expire from REDIS.  When a REST API call refers to an expired vCon, it will be replaced in the cache.&#x20;
5. Optionally, webhook links can be used at the end of a chain to notify external systems. Alternatively, applications  can leverage the native notifications on their systems.
6. To find a vCon, use the application databases (in the figure shown as Mongo)  native functionality.  For instance, to find all vCons where a party is "george.washington@whitehouse.gov", the application would use the Mongo command: \
   \
   `db.conversations.find_all({$.parties.email: "george.washington"}), assuming the vCons were kept in a collection called "conversations".`&#x20;
7. To maintain security, logging and synchronization of vCons, updates to the vCon should be made using the REST API.  For instance, when a vCon is deleted using the REST API, it will also be deleted in any of the storages.&#x20;





1. Then using the REST API to trigger the execution of a "chain".   Inside the conserver, the v
