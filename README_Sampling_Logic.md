# Filplus Retrieval Sampling

## Background

The goal of filplus retrieval sampling is to offer an aggregated view of how each storage provider performs in
retrievability.
The Filecoin community would like to make this retrieval sampling logic public and transparent, so that participants
including storage providers, notaries, and data clients are aligned on best practices and how their reputation is
evaluated.

## Sampling Logic

All active verified deals in Filecoin network will be sampled randomly.

Newer deals will have a higher chance to be sampled than older deals. The chance is 4x for each year the deal is newer
determined by deal start date. This gives newer deal a higher chance to be sampled for retrieval testing.


## Retrieval Testing

### Graphsync

Graphsync is the default retrieval protocol for Filecoin. It is a libp2p protocol that is used to retrieve data from the
storage provider.

We look at the `label` field of the `PublishStorageDeals` message. If the `label` is a valid `CID` and it's
not `pieceCID`, then we **assume** it is the root CID of the deal.

We then make a graphsync retrieval request to retrieve only the root block.

### Bitswap

Bitswap is the retrieval protocol that's also used in IPFS. It can be enabled by
running [`booster-bitswap nodes`](https://boost.filecoin.io/bitswap-retrieval)

Similar to Graphsync retrieval, we assume the label is the rootCID of the deal. We then first query the multiaddr of the
storage provider to get the libp2p multiaddr that serves bitswap retrieval.
Then we will attempt to make a single block retrieval request to the storage provider using bitswap protocol for just
the root block.

### HTTP

HTTP is the retrieval protocol that can serve piece, file and block retrieval. It can be enabled by
running [`booster-http nodes`](https://boost.filecoin.io/http-retrieval)

Piece retrieval is by default enabled so instead of assuming the deal proposal has the correct RootCID set in the label,
we will use the `pieceCID` field of the deal proposal and make piece retrieval.

1. Connect to libp2p multiaddr of the provider that's published on the chain
2. Get HTTP multiaddr using /fil/retrieval/transport/1.0.0 protocol - The SP needs to handle this protocol and return
   HTTP endpoint. SP can use boost or other implementation that produces the same behavior

#### Piece range retrieval

1. Use the `pieceCID` field of the deal proposal and make piece retrieval with the HTTP endpoint
2. Make range retrieval for the first 100 bytes and verify it is a valid CAR V1/V2 header
    * If it is a [CAR V2 header](https://ipld.io/specs/transport/car/carv2/#header), then check the `data_size` in the
      header to calculate how much padding has been used. In the next step, we only need to perform range retrieval
      between `[data_offset, data_offset + data_length]`
3. Make ranges retrieval for a random offset of that piece, up to 8MiB length
    * We check if retrieved data is all zeroes. Overtime, we will get a ratio of how much datacap is under utilized by
      padding data with zeroes
    * Try to find `[varint, CID, block, varint, CID]`. This is a valid IPLD data block. A valid IPLD block size is <=
      4MiB so we should expect to get at least
      one [IPLD data block](https://ipld.io/specs/transport/car/carv1/#format-description) within that range
    * Calculate the compression ratio of the block bytes using zstd compression
        * High compression ratio / low entropy means the data is highly repetitive (i.e. repeating "hello world")
        * Low compression ratio / high entropy means the data is noisy (i.e. random bytes, already compressed or
          encrypted)
        * Useful data usually does not have an extremely high or low entropy and the compression ratio can be compared
          to the original data source
4. The purpose of this retrieval type is to make sure the clients are not padding too much zeroes or are actually
   storing data that is not useful. Since the retrieval is lightweight, most of the retrieval testing will be using this
   kind

#### Whole piece retrieval

1. Retrieve the whole piece and verify `PieceCID` and `PieceSize` matches the deal proposal
2. The purpose of this retrieval is to make sure the HTTP endpoint is in fact serving the correct piece. This retrieval
   is more expensive and will be used very rarely

#### File retrieval

1. If the client has provided a list of CIDs for files included in the dataset, we have the opportunity to retrieve the
   whole file
2. Retrieve the first 4k bytes of the file, check the file type
   using [libmagic](https://man7.org/linux/man-pages/man3/libmagic.3.html). Overtime, this will give us an overall
   picture of what types of files is this dataset composed of
3. With less sampling rate, retrieve the whole file and store it in an online storage (i.e. web3.storage). The file can
   be downloaded by notary to check the content

## Storage Provider Self-Testing
It is possible to check how good you, as a storage provider is performing in terms of retrievability.

You may run a one-off test with your storage provider by following below instruction:
```
go install storagestats/integration/oneoff@latest
oneoff <providerID> <dealID>
```

This will fetch relevant information about your miner and the deal and run retrieval testing against all currently available protocols.

The result will not be pushed to the reputation working group database.
