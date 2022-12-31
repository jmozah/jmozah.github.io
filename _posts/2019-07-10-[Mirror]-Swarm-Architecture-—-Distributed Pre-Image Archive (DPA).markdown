---
layout: post
title:  "[Mirror] Swarm architecture - Distributed Pre-Image Archive (DPA)"
date:   2019-07-19 10:36:43
categories: Swarm
---

As announced in my [previous post](https://jmozah.github.io/2019/06/26/Mirror-Swarm-Architecture-a-view-from-above.html), this blog is the first in a series of blogs that will take an in-depth look at the architecture of each of the modules that make up the Swarm network. Today, we’ll look at the DPA layer of Swarm, which is responsible for parcellating files into chunks and putting them back together, along with encrypting & decrypting them. In this blog, I will try to depict these four functionalities and the inner workings of the DPA. The image below shows where the DPA fits in the Swarm architecture.

Note: Please read the [Swarm Architecture](https://jmozah.github.io/2019/06/26/Mirror-Swarm-Architecture-a-view-from-above.html) post before reading this one to get an idea about Swarm, its components and functionalities as a whole.

![image](https://user-images.githubusercontent.com/940575/210151461-70211cd2-d228-46da-a51c-a8b572cb3e06.png)
Swarm implements decentralised storage by splitting files into smaller pieces, called chunks, and strewing them all over the Swarm network. It does this by using specific methods that systematically scatter (sync) and un-scatter (join) chunks. DPA is the interface between the external “world” representation of storage (i.e. files) and internal “Swarm” representation of storage (i.e.chunks). It is the module that enables people to upload or download files to and from Swarm.

But, before I delve any deeper into how DPA works it is essential to understand what happens when a file is uploaded to or downloaded from Swarm.

![image](https://user-images.githubusercontent.com/940575/210151471-93e27b29-fd12-49af-82ca-2ed7b66cbdab.png)

### Upload
When a file is uploaded to a local Swarm node, the following things happen:

- The chunker splits the file into fixed-sized 4K chunks.
- Optionally, each of the chunks is encrypted.
- The chunks are stored in the local store levelDB.
- The syncer is triggered and syncs these new chunks to the Swarm network, meaning, the network becomes “aware” that new files have been uploaded.

### Download
When a file is downloaded by supplying a Swarm hash, the next sequence of events is triggered:

- The joiner first retrieves the root chunk and all the subsequent tree chunks pointed to by the data present in the tree chunks.
- The syncer then sends a request to the network and gets the respective data chunks (client can also request a portion of a file).
- The chunks are decrypted if they are encrypted.
- These chunks are stitched together accordingly and then given to the client.

### Chunks
An important thing to know is that in Swarm a file is represented as a Merkle tree. The leaf nodes are data nodes that store actual data, while the rest of the nodes in the Merkle tree are tree nodes. The swarm hash of the file will point to the root chunk of that Merkle tree.
![image](https://user-images.githubusercontent.com/940575/210151507-3ad8e026-e72c-4943-8f60-9656b07e2021.png)

There are also two kinds of chunks: a data chunk and a tree chunk. Data chunks are the ones which store the actual file data as part of their payload. Tree chunks are intermediary chunks which store pointers to lower level chunks of the Merkle tree and can store up to 128 32-byte chunk addresses. Tree chunks also store the file size of their children sub-tree.

Having files split into chunks and arranged as a Merkle tree has many advantages, over and above the cryptographic ones. It can maintain file integrity, it allows for easy access to data in a random file position etc.

### Chunker & Joiner
![image](https://user-images.githubusercontent.com/940575/210151645-97f241c7-115c-47d9-bc76-e517f5708073.png)

The current implementation of the chunker is called the pyramid chunker because it forms a pyramid out of chunks, created from the uploaded file (see Picture 4). Data chunks, the ones that contain the actual data, form the base of the pyramid.

The Level 0 tree chunks in Picture 4 are formed by packing the chunk hashes of 128 data chunks. A chunk hash is nothing but a Keccak 256 hash of the payload in the chunk. Along with 128 32-byte hashes (green blocks ), every tree chunk will also have an 8-byte field called “sub-tree size” (blue blocks). This is only the size of the actual file contents under a given tree chunk. This field is useful when we’re looking for the location of a given file.

For every 128 data chunks in the pyramid base, a tree chunk is formed in level 0 of the pyramid. If the number of Level 0 tree chunks is greater than 1, then Level 1 tree chunks are formed, and so on until you accommodate all the lower levels in a single tree chunk. The hash of that root tree chunk is the hash of that file in Swarm.

The joiner, on the other hand, is exactly the opposite of the chunker. It implements a LazyChunkReader interface which reads the data only when the client demands it. Given a root hash of a file and a file position to read, the joiner walks down the tree until it reaches the data chunk which contains the required content. The joiner walks down by opening the tree chunks, picking up the hash of the lower level chunk from that and following until it reaches the required data chunk which holds the data of the original file.

### Encryption & Decryption
Encryption of a file is optional in Swarm. When a file is uploaded, a flag indicates whether the file should be encrypted or not.
This is how the upload command with encryption looks from the command line:


swarm up — encrypt "file to upload"


Encryption happens after the pyramid chunker chunks the data. A temporary encryption key is created for every upload and is used to encrypt the data. The unencrypted chunk is encrypted with a temporary encryption key. The rest of the chunk-storing process remains the same. The swarm hash (32 bytes) of the encrypted tree is appended with the decryption key (32 bytes) to form the final swarm hash (64 bytes) for the file.

Inversely, decryption happens when the chunk is joined and before handing over the chunk to the client. The decryption key is part of the file hash and is used to decrypt all the chunks of that Merkle tree.
  
### Code Commentary
The diagram below shows the upload path of the Swarm codebase. I picked upload as an example, but a download of a file also follows a similar path. In the comments below, you can also read what each step means and does and which functions it employs to achieve that.
  
![image](https://user-images.githubusercontent.com/940575/210151678-f27bd3ac-7c1a-4648-9d80-b7b1e4b7f578.png)

#### HTTP Request
A file can be uploaded or downloaded from Swarm using:

- HTTP POST — endpoint is exposed by a Swarm server
- Or by using the rpc command “swarm up <filename>” or “swarm down <swarm hash>”
Internally the rpc command also uses the same HTTP endpoint to upload or download the files. The client wraps the file or directory to be uploaded as a tar stream and sends it through the HTTP request.

#### server.go
This file implements the HTTP server which exposes the following HTTP endpoints:

- HandlePostFiles() function takes care of the normal file upload.
- HandlePostRaw() function takes care of the RAW file upload. RAW files are files which do not have a Swarm manifest attached to it.

Both these functions call the Store() API described below.

#### api.go
For normal file or collection upload, the UploadTar() function does two things. First, it unpacks the tar and stores every file in the tar (preserving the path) using the filestore API and then creates a manifest for that collection and stores the manifest too, using the same filestore API described below.
For the Raw file storage, it just uses the same filestore’s Store() API and stores the file.

#### filestore.go
This file exposes the Swarm filesystem API Store() and Retrieve() which stores and retrieves the files from Swarm. The Store() function is also responsible for tagging the upload. Tags are used to monitor the upload to localstore, as well as to the network. These tag counters are used to show the progress of the upload.

#### pyramid.go
This file implements the chunker functionality. It gets the file stream as an argument and splits the file into chunks and forms a Merkle tree.

Split() function implements the chunker functionality.
  
prepareChunks() goes through the file stream and splits it into 4K data chunks. It also starts several processor() which sends these chunks to the network to be stored.

buildTree() is called by prepareChunks() after every 128 data chunks are created. This function creates a tree chunk and inserts it into the Merkle tree at the appropriate position.

#### hasherstore.go
Put() receives the chunks (both data and tree chunks) from the pyramid chunker, encrypts them (optionally), and queues them for storage.

storeChunk() function implements the queue wherever chunks are queued. Then, for every chunk, a go routine is spawned to push it to the actual storage. The queue acts here as a back pressure to pyramid chunker.

#### netstore.go
Put() gives the chunk to localstore and notifies the syncer client to sync this new chunk to the Swarm network.

#### mode_put.go (localstore)
Put() is the function that actually stores the chunk in the local levelDB.

Localstore has a concept of Index built on top of levelDB. Every index has a key and value which can be a composite of many values. When a chunk is stored, it actually adds in many indexes.

- retrievalDataIndex: This index is where the actual data is stored. The key of the Index is the Address and the value is a composite of StoreTimestamp|BinID|Data
- pullIndex: This index is used in the pull syncing process. The key is the Proximity Order and the value is the BinID|Hash
- pushIndex: This index is used to track the push syncing process. Any entry added in this index indicates that this chunk needs to be pushed somewhere. The key of the index is StoreTimestamp and the value is hash|Tags

### Conclusion
In the next blog of the series I will take a look at Hive, so stay tuned.

