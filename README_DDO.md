
# 🛰️ Filecoin Retrieval DDO Collector

## 🔍 Project Overview

This project is a retrieval subsystem component for the Filecoin network, focused on real-time subscription and collection of **DDO (Data Deal Operations)** events. It captures lifecycle changes of storage deals, such as submission, precommitment, and activation. Compared to traditional f05 DealProposal scans, this provides significantly higher data efficiency and accuracy.

## 🎯 Project Objectives

- 📡 Subscribe to on-chain DDO-related events in real time;
- 📋 Accurately track the lifecycle states of storage deals;
- 🗃️ Persist structured DealEvent data for downstream usage (retrieval, validation, analytics);
- 🚫 Eliminate the need for daily f05 DealProposal dumps.

## ⚙️ Architecture and Principles

```
            +-----------------------------+
            |     Lotus FullNode RPC      |
            +-----------------------------+
                      ▲
                      | ChainNotify()
                      ▼
        +-------------------------------+
        |   DDO Event Subscription      |
        +-------------------------------+
        | - DealPreCommit               |
        | - DealActivated               |
        | - DealCompleted (optional)    |
        +-------------------------------+
                      ▼
        +-------------------------------+
        | Structured DealEvent Record   |
        +-------------------------------+
                      ▼
               +----------------+
               |   MongoDB      |
               +----------------+
```

## 🧱 Implementation (Golang)

### Initialize Lotus RPC connection and subscribe:

```go
lapi, closer, err := client.NewFullNodeRPC(ctx, "ws://127.0.0.1:1234/rpc/v0", api.FullNode, httpHeaders)
ch, _ := lapi.ChainNotify(ctx)
```

### Capture and persist full Deal object:

```go
for notif := range ch {
  for _, ev := range notif.Events {
    if ev.Type == api.HintDealActivated {
      log.Printf("[DDO] Activated: DealID=%d, RootCID=%s, Epoch=%d",
        ev.Deal.DealID, ev.Deal.Root, notif.Val.Height())
      collection.InsertOne(ctx, bson.M{
        "event": "Activated",
        "epoch_height": notif.Val.Height(),
        "created_at": time.Now(),
        "deal": ev.Deal,  // Store full deal object
      })
    }
  }
}
```

## 🧩 MongoDB Schema Example

```go
type DealEvent struct {
  Event     string         `bson:"event"`        // e.g., "Activated"
  Timestamp int64          `bson:"epoch_height"`
  CreatedAt time.Time      `bson:"created_at"`
  Deal      api.MarketDeal `bson:"deal"`         // Store full Deal object
}
```

### Sample MongoDB insert logic:

```go
collection := mongoClient.Database("ddo").Collection("events")
_, err := collection.InsertOne(ctx, DealEvent{
  Event:     "Activated",
  Timestamp: notif.Val.Height(),
  CreatedAt: time.Now(),
  Deal:      ev.Deal,
})
```

## ✅ Features

| Feature                  | Description |
|--------------------------|-------------|
| ✅ Real-time event stream | No need for daily dumps |
| ✅ Precise deal tracking  | Detect only sealed+activated data |
| ✅ Stores full Deal object | All relevant fields persisted |
| ✅ Supports resumption    | Events can be persisted and reprocessed |
| ✅ Easy integration       | Embeddable in retrieval/indexing systems |

## 🧪 Environment Dependencies

- Lotus FullNode (recommended v1.31+)
- Go 1.20+
- MongoDB 6.0+
- `go.mongodb.org/mongo-driver` for database access

## 📦 Grant Objectives

The project seeks funding support to:

- ✅ Complete end-to-end support for DDO lifecycle tracking;
- ✅ Extend activation/sealing event classification and aggregation;
- ✅ Launch a public-facing DDO event API service;
- ✅ Integrate with Boost, retrieval providers, and indexers for validation.
