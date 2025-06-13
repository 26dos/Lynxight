
# SubPiece Inclusion Proof System for Filecoin Retrieval Checker

## 🔍 What Are We Trying to Solve?

In Filecoin's retrieval checking system, checker nodes need to validate that storage providers (SPs) are correctly storing and serving data by requesting random subpieces of stored pieces (commP / PieceCID). Specifically:

- A checker node has access to CommPs (PieceCIDs) for full pieces stored on Filecoin;
- The checker node requests a specific byte range (subpiece) from the SP;
- The SP returns data claimed to be from that byte range.

### ❌ Problem:
There is currently no cryptographic way for the verifier to ensure:
- The returned data actually belongs to the original piece (known CommP);
- The returned data is from the exact requested byte range.

---

## ✅ Proposed Solution

We propose the following protocol:
- The checker requests a byte range (offset + length) from the SP along with a Merkle inclusion proof;
- The SP returns:
  - Raw data for the requested range;
  - A Merkle proof linking this data to the known CommP;
- The checker verifies the inclusion proof using the known CommP.

This enables cryptographic verification without relying on IPNI or external indexing systems.

---

## 📦 Use Case

This system is designed to:
- Randomly select a subpiece (byte range) from a CAR file;
- Generate a Merkle proof for the subpiece;
- Allow the checker to download the subpiece and validate its inclusion in the complete PieceCID.

---

## ⚙️ Technical Overview

### 📘 SubPiece Definition
- A SubPiece = a specific byte range (offset + length) from the CAR file;
- The range corresponds to serialized IPLD data blocks;
- Inclusion proof = Merkle path proving the data is part of the complete Piece (CommP);
- The checker reconstructs the Merkle root and compares it to the known PieceCID.

---

## 🛠️ Proof Generation Logic (SP Node)

1. Load the full CAR file;
2. Parse and sort all blocks by CID or order;
3. Construct the Merkle tree;
4. Generate inclusion proof path for the target byte range;
5. Return `SubPieceInclusionProof`.

```rust
pub fn generate_subpiece_inclusion_proof(
    car_path: &str,
    offset: usize,
    length: usize
) -> Result<SubpieceInclusionProof, String> {
    // Load CAR file, parse blocks
    let blocks = load_blocks_from_car(car_path)?;
    let merkle_tree = build_merkle_tree(&blocks)?;
    let proof = merkle_tree.generate_proof_for_range(offset, length)?;
    Ok(proof)
}
```

---

## 🔎 SubPiece Validation Logic (Checker Node)

1. Request: `GET /piece?offset=8192&length=4096&proof=true&pieceCid=<piece>`
2. Receive the subpiece + Merkle proof;
3. Hash the data to construct leaf node;
4. Use the proof to reconstruct the Merkle root;
5. Compare with expected PieceCID.

```rust
pub fn verify_subpiece_inclusion(
    data: &[u8],
    proof: &SubpieceInclusionProof,
    expected_root: &[u8]
) -> bool {
    let leaf_hash = sha256(data);
    let computed_root = proof.compute_root_from_leaf(&leaf_hash);
    computed_root == expected_root
}
```

---

## 🧱 Architecture Diagram (Text-Based)

```text
        ┌────────────┐
        │  Checker   │
        └────┬───────┘
             │
   GET /ipfs?offset=X&length=Y&proof=true&rootCid=...
             │
             ▼
       ┌─────────────┐
       │ Storage Prov│
       └────┬────────┘
            │
    GenerateSubpieceProof(offset, length)
            │
            ▼
       ┌─────────────┐
       │ CAR Merkle  │
       │ Tree Builder│
       └────┬────────┘
            │
       Returns: Data + Merkle Proof
            │
            ▼
       Checker Validates:
       ┌────────────────────────────┐
       │ verify_subpiece_inclusion()│
       └────────────────────────────┘
```

---

## 📌 Benefits
- Verifiable integrity without full CAR download;
- No reliance on IPNI;
- Flexible integration for retrieval markets and on-chain deal verifications.
