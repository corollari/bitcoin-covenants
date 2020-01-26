## A compilation of research on Bitcoin Covenants {.subtitle}

## What are covenants?
Covenants, also known as spending constraints, is the name given to hipothetical bitcoin scripts that when attached to UTXOs would constrain the way these can be spent, for example restricting the addresses where such coins can be sent.

Covenants which replicate themselves into the UTXOs where they are spent to are called recursive.

## Why are covenants interesting?
Generalized covenants would enable a ton of new applications for scripts, such as:
- Scripts with unbounded length (including arbitrary sized multisigs)
- Turing complete contracts
- Transaction-level MAST (a really coarse-grained version of MAST that would split code paths at the script/transaction level)
- Drivechain-like two-way pegs

Several more use cases are described [on the OP\_CTV page](https://utxos.org/uses/).

## Why was this created?
Because I spent a whole month researching ways to create covenants with the current capabilities of Bitcoin's Script and, after thinking I had found a beakthrough, I discovered that the same system I had thought about had already been described a few months ago in one of the thousands of posts of the bitcoin mailing list.
The goal of this website is to provide information on bitcoin covenants so that other people don't end up wasting their time like I did.

In other words:

> Those who don't know history are doomed to repeat it

## Opcode based covenants
### OP\_CHECKOUTPUTVERIFY
One of the earliest covenant proposals, outlined in [this paper](https://maltemoeser.de/paper/covenants.pdf) describes a new opcode to be added to the Bitcoin protocol, this new opcode takes three arguments `(index, value, pattern)` and returns true if and only if the following conditions holdfor the `index`-th output of the transaction that is trying to spend the UTXO:
- The amount of bitcoin spent on that output is equal to `value`
- The script attached to the output is equal to `pattern` except for the parts of `pattern` that contain placeholders for public keys and hashes.

A technically-wrong but exemplary implementation of the opcode in python would be as follows:
```python
def CheckOutputVerify(tx, index, value, pattern):
	output = tx.outputs[index]
	if value != 0:
		if output.value != value:
			return False
	if pattern != "":
		sanitizedPattern = pattern.replace("PubKeyPlaceholder", "00000000").replace("HashPlaceholder", "00000000")
		mask = replaceLocationsOfPlaceholdersWithZeroes("1"*len(sanitizedPattern), ["PubKeyPlaceholder", "HashPlaceholder"])
		if (output.script & mask) != sanitizedPattern:
			return False	
	return True
```
**Note**: This implementation is wrong due to the fact that the binary operations are replaced with string operations, which are themselves wrong in lots of ways (a mask of bytes is not constructed as a string of 1's...) but as a pseudocode example it helps get the concept across

This opcode would enable all kind of covenants, but nowadays it has been superseded by other systems and the probability of it ever getting deployed is null.

### OP\_CHECKOUTPUTSHASHVERIFY
Very similar to OP\_CHECKOUTPUTVERIFY, see the [BIP](https://github.com/JeremyRubin/bips/blob/op-checkoutputshashverify/bip-coshv.mediawiki) for more details.

### OP\_CHECKTEMPLATEVERIFY

For more information, see [BIP119](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki) and [the website dedicated to it](https://utxos.org/). 

## Signature based covenants
### A primer on signatures

### OP\_CHECKSIGFROMSTACK

For more information, see the [paper](https://fc17.ifca.ai/bitcoin/papers/bitcoin17-final28.pdf) and [article](https://blockstream.com/2016/11/02/en-covenants-in-elements-alpha/) about it.

**Note**: This opcode has been implemented inside Elements, the blockchain layer upon which Blockstream's Liquid sidechain is built. Furthermore, an opcode very similar to it called [OP\_CHECKDATASIG](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/op_checkdatasig.md) has been implemented and deployed in Bitcoin Cash.

### Signature costruction 
Approaches based on constructing a public key so that only a single signature for it is known.

### Multi-party Computation variant

---

This text is open source (MIT licensed), and available [on github](https://github.com/corollari/bitcoin-covenants). Contributions are welcome.
