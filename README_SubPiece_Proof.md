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

##  What is a SubPieceInclusionProof?

A **SubPieceInclusionProof** is a Merkle Tree-based inclusion proof that demonstrates a specific subpiece of data is indeed part of a larger dataset (e.g., a full Piece corresponding to a CommP).

In Filecoin, each data Piece is ultimately converted into a CommP (PieceCID), which is the root hash of a binary Merkle tree constructed over a 0-padded unsealed CAR file. When a specific byte range (subpiece) is requested, the system must prove that this range is a subset of the original Merkle tree structure.

---

## Proof Generation Principles (on SP side)

📂 **Steps:**

**Step 1: Load the full CAR file**
- Read all IPLD data blocks;
- Data may include padding for sector alignment.

**Step 2: Construct Merkle Tree in order**
- Each data block (or subsegment) becomes a leaf node;
- The CAR is divided into fixed-size chunks (e.g., 4KiB);
- Compute `SHA256(data)` for all leaf nodes;
- Recursively merge: `H = SHA256(left || right)` to form parent nodes up to the Merkle Root (which is the CommP).

**Step 3: Locate target subpiece and extract Merkle path**
- Determine which leaf node(s) the requested `[offset, offset + length)` falls into;
- Extract the Merkle proof path from the leaf node to the root;
- The proof includes sibling hashes at each level needed to reconstruct the Merkle root.

**Step 4: Package the subpiece and the proof**
- Return the data and the Merkle path to the requester.

---

## Cryptographic Explanation (Merkle Inclusion)

Assume:
- Full data: `D = [B0, B1, ..., Bn]`;
- Each `Bi` is 4KiB;
- `leafHash(i) = SHA256(Bi)`;
- `parent(i,j) = SHA256(leafHash(i) || leafHash(j))`;
- The final `MerkleRoot = rootHash`, which is the CommP.

### Example:

If the request targets block `B3`, the SP returns:
```
Data = B3  
Proof = [H(B2), H(B01), H(B4567)]
```

The reconstruction process is:
```
H3 = SHA256(B3)  
H23 = SHA256(H(B2) || H3)  
H0123 = SHA256(H(B01) || H23)  
Root = SHA256(H0123 || H4567)
```

If `Root == CommP`, the proof is valid.

---

## Verification Process (on Checker side)

Received:
- `data`: raw subpiece bytes;
- `proof`: Merkle path;
- `pieceCID`: CommP.

### Verification steps:
```rust
leaf_hash = sha256(padded_data)
for sibling_hash in proof.path:
    if is_left:
        leaf_hash = sha256(leaf_hash || sibling_hash)
    else:
        leaf_hash = sha256(sibling_hash || leaf_hash)
```

### Final comparison:
```rust
assert leaf_hash == piece_cid_digest
```

If the assertion holds, then:

✅ The data is from the correct location within the full Piece, and its integrity is cryptographically protected by the Merkle Root (CommP).

---

## Consistency with Filecoin CommP

- CommP is calculated using: `unsealed data → FR32 → Merkle Tree → Digest`;
- Data must be aligned using **127-byte per 128-byte padding** (FR32 Padding);
- Filecoin uses a **custom multibranch Merkle structure** (mixed fan-out);
- Validation must use the same **hash function** and **tree structure**;
- In production systems, both SP and Checker should use the same Filecoin-standard libraries (e.g., `filecoin-proof`, `rs-carv2-merkle`) to generate and validate the Merkle structure.

---

## 🛠️ Proof Generation Logic (SP Node)

```rust
pub fn generate_subpiece_inclusion_proof(
    car_path: &str,
    offset: usize,
    length: usize
) -> Result<SubpieceInclusionProof, String> {
    let blocks = load_blocks_from_car(car_path)?;
    let merkle_tree = build_merkle_tree(&blocks)?;
    let proof = merkle_tree.generate_proof_for_range(offset, length)?;
    Ok(proof)
}
```

---

## 🔎 SubPiece Validation Logic (Checker Node)

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
   GET /piece?offset=X&length=Y&proof=true&pieceCid=...
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
