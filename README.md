# BlockKeys

**WIP Elixir implementation of BIP44 Blockchain Wallet**

Do not use this, it's just a learning implementation.

## Installation

If [available in Hex](https://hex.pm/docs/publish), the package can be installed
by adding `bitcoin_addresses` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:block_keys, "~> 0.1.0"}
  ]
end
```

Documentation can be generated with [ExDoc](https://github.com/elixir-lang/ex_doc)
and published on [HexDocs](https://hexdocs.pm). Once published, the docs can
be found at [https://hexdocs.pm/bitcoin_addresses](https://hexdocs.pm/bitcoin_addresses).

## BIP32 Mnemonic

### Generate 24 word phrase

The mnemonic generator from random entropy is currently implemented. 
The BIP32Mnemonic module will output 24 mnemonic words corresponding to a 256bit entropy.

```
iex(1)> BlockKeys.Bip32Mnemonic.generate_phrase
"baby shadow city tower diamond magnet avocado champion crash evolve circle chair boring runway remain fantasy finger impose crumble profit excuse group twist purse"
```

The steps to generate the mnemonic words are the following:

- Create random sequence of 256bits
- Create the checksum by taking the first 8 bits of the entropy SHA256 hash
- Add checksum to the end of the random sequence (now 264bit)
- Split result into 11 bit sequences -> 24 sequences
- Map each 11 bit value to a word in the predefined dictionary

### Restore entropy given phrase

```
iex(1)> BlockKeys.Bip32Mnemonic.entropy_from_phrase("fade joy announce clever yellow special near expand bus jealous memory usual just daughter bring oppose tone bind cloud mosquito route warfare engage champion")
<<81, 207, 16, 37, 21, 79, 241, 161, 228, 226, 129, 30, 238, 242, 43, 248, 23,
  150, 111, 135, 12, 220, 228, 66, 200, 175, 200, 11, 201, 238, 18, 145>>
```

The steps to restore the entropy given the phrase are:

- Split the string into individual words
- Look up the index in the word dictionary for each word and create a list
- Convert each element in that list to a binary string of 0s and 1s
- Remove checksum (last 8 bits)
- Split the bitstring into groups of 8 bits
- Convert each chunk into a byte (8 bits)
- Convert the byte list into a binary

### Generate seed given entropy and optional password

To generate the final seed that will in turn derive our tree of private in public addresses we will use a key stretching function called PBKDF2.
The hashing function we will use is HMAC-SHA512 with 2048 iterations. 

```
iex(1)> ent = BlockKeys.Bip32Mnemonic.entropy_from_phrase("future skate cash mountain senior maze spoil supply flee front ocean organ pause patrol define box save exhibit bike blush length globe jungle bacon ")
<<94, 153, 64, 141, 72, 76, 59, 19, 52, 150, 206, 88, 203, 170, 99, 206, 58, 23,
  66, 78, 96, 212, 191, 201, 240, 88, 140, 72, 0, 198, 158, 64>>
iex(2)> BlockKeys.Bip32Mnemonic.generate_seed(ent)
"3b8831d9372b6bccbdb18b013c1ad88d6d33650d82865bbd84dfaa9080a9f67f7bad64251a304960488fcdea85fd015c2899ed15cb221ebb3f373340e37409f1"
iex(3)> BlockKeys.Bip32Mnemonic.generate_seed(ent, "supersecurepassword")
"b156a85e86341a28503951bc4fa543b73ed203ef241ceee06386a1f9d15229a8e7c4a1709eca5306266f9a7195101219cfa5f145f574fe53a3dab481a2ea848b"
```

Note that there is no such thing as a right or wrong password. If you use the wrong password you will generate a completely different seed that will in turn
create a completely different tree of wallet addresses.

Here are the steps:

- Create a salt. If no password is chosen, the salt will be comprised of the string "mnemonic"
- Create initial PBKDF2 round by passing the entropy as the key and the salt as the data.
- Repeat this 2048 times. The entropy parameter stays the same, while the data is the xor of the current and previous result
- The final binary is encoded in hex