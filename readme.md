# Introduction to Zero Knowledge Proof with Merkle Trees Proof of Membership

I recently came across [Semaphore](https://semaphore.appliedzkp.org/docs/introduction)

> Semaphore is a zero-knowledge protocol that allows you to cast a signal (for example, a vote or endorsement) as a provable group member without revealing your identity. Additionally, it provides a simple mechanism to prevent double-signaling. Use cases include private voting, whistleblowing, anonymous DAOs and mixers.

_But how do you prove that you are a member of a group without revealing your identity?_

At the heart of the protocol is the Semaphore [circuit](https://github.com/semaphore-protocol/semaphore/tree/main/packages/circuits). It uses Merkle Proof as Proof of Membership. Each group is represented by a Merkle root. A member of the group can prove that he is a member of the group by providing a Merkle Proof. I.e. he knows the leaf, the pathIndices and the siblings that lead to the _public_ Merkle root, without revealing the leaf.

Let's run the circuit [merkle_tree_proof.circom](merkle_tree_proof.circom) which is derived from [tree.circom](https://github.com/semaphore-protocol/semaphore/blob/main/packages/circuits/tree.circom), this is not the actual circuit used in Semaphore, but it is a Merkle tree part of it.

## Merkle Tree

Very details on [what is a Merkle Tree](https://decentralizedthoughts.github.io/2020-12-22-what-is-a-merkle-tree/)

## Proving-verifying

### Circom and snarkjs installation

Follow the [installation guide](https://docs.circom.io/getting-started/installation/)

### Create the circuit

- Add `circomlib` by running `yarn add circomlib`
- The circuit is defined in [merkle_tree_proof.circom](merkle_tree_proof.circom)

```js
pragma circom 2.0.0;

include "./node_modules/circomlib/circuits/poseidon.circom";
include "./node_modules/circomlib/circuits/mux1.circom";

template MerkleTreeInclusionProof(nLevels) {
    signal input leaf;
    signal input pathIndices[nLevels];
    signal input siblings[nLevels];

    signal input root;

    component poseidons[nLevels];
    component mux[nLevels];

    signal hashes[nLevels + 1];
    hashes[0] <== leaf;

    for (var i = 0; i < nLevels; i++) {
        pathIndices[i] * (1 - pathIndices[i]) === 0;

        poseidons[i] = Poseidon(2);
        mux[i] = MultiMux1(2);

        mux[i].c[0][0] <== hashes[i];
        mux[i].c[0][1] <== siblings[i];

        mux[i].c[1][0] <== siblings[i];
        mux[i].c[1][1] <== hashes[i];

        mux[i].s <== pathIndices[i];

        poseidons[i].inputs[0] <== mux[i].out[0];
        poseidons[i].inputs[1] <== mux[i].out[1];

        hashes[i + 1] <== poseidons[i].out;
    }

    root === hashes[nLevels];
}
component main{public [root]}  = MerkleTreeInclusionProof(3);
```

- Hash function is Poseidon
- `nLevels` is the number of levels in the tree, or the depth of the tree
- `leaf` is the hash of the original data
- `pathIndices` is the position of the leaf in the tree, and array with length = nLevels, each element is either 0 or 1
- `siblings` is the hashes of the nodes on the path from the leaf to the root. The number of siblings = nLevels
- Given the `leaf`, the `pathIndices` and the `siblings`, the circuit whould be able to compute the root
- An example: for the leaf H(E), `pathIndices` is [0,0,1] and the `siblings` is [H(F), H(GH), H(ABCD)]
  ![Merkle tree](merkle_tree1.png "Merkle tree")

- The circuit computes the root and make sure it is equal to the root input

### Compile the circuit

```bash
circom merkle_tree_proof.circom --r1cs --wasm
```

### Compute the witness

#### Compute input

As a prover, we need to know _everything_, namely: `leaf`, `pathIndices`, `siblings` and `root`.

Let follow the example showed in the diagram above, proving leaf H(E) belongs to the tree, with [A,B,C,D,E,F,G,H] = [1,2,3,4,5,6,7,8]

Our input should be in this format

```json
{
  "leaf": H(E),
  "siblings": [
    H(F), H(GH), H(ABCD)
  ],
  "pathIndices": ["0","0","1"],
  "root": H(ABCDEFGH)
}
```

Details in [input.json](input.json) with the help of [Python script](poseidon_py/zok_poseidon.py) to compute Poseidon hash, taken from [ingonyama.ctfd](https://ingonyama.ctfd.io/)

#### Compute and display the witness

- Run the following command to compute the witness

```bash
node merkle_tree_proof_js/generate_witness.js merkle_tree_proof_js/merkle_tree_proof.wasm input.json witness.wtns
```

- To display the witness in .json format

```bash
snarkjs wtns export json witness.wtns witness.json
```

### Generate the proof

#### Download the trusted setup from [trusted-setup-pse](https://www.trusted-setup-pse.org/)

```bash
wget https://storage.googleapis.com/trustedsetup-a86f4.appspot.com/ptau/pot14_final.ptau
```

#### Generate the verification key

```bash
snarkjs plonk setup merkle_tree_proof.r1cs pot14_final.ptau merkle_tree_proof.zkey
```

#### Generate verification key in json format

```bash
snarkjs zkey export verificationkey merkle_tree_proof.zkey verification_key.json
```

#### Generate the proof

```bash
snarkjs plonk prove merkle_tree_proof.zkey witness.wtns proof.json public.json
```

### Verify the proof

```bash
snarkjs plonk verify verification_key.json public.json proof.json


[INFO]  snarkJS: OK!
```
