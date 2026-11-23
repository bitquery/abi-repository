# abi-repository

Contract ABIs for the Bitquery EVM stream processors. Every processor downloads the tarball of
branch `main` when it starts, so what is merged here is live at the next restart of each node.

There are two kinds of file, and the difference is the file name.

## `<protocol>/<Contract>.abi`: registered chain-wide

Every file whose name ends in `.abi`, **in any directory**, is registered by selector (methods) and
by topic0 (events) for every contract on every chain. Nothing in it is tied to an address.

- The first file to declare a selector keeps it. Files are read in path order, so `aerodrome/` is
  ahead of `uniswap/`.
- A selector is persisted on each node the first time it is seen, and deleting the file later does
  not remove it from that node.
- A method's selector does not include its outputs, and an event's topic0 does not include which
  arguments are indexed. Two files that declare one selector with different outputs or indexing
  cannot both be right, and only one of them is ever used.

So a chain-wide file carries only what is safe for every contract:

- Prefer a `.lite.abi`: the events, and the methods whose outputs no other protocol declares
  differently. `aerodrome/CLPool.lite.abi` leaves out `slot0()` and `ticks(int24)` because Uniswap
  v3 returns other tuples under the same selectors.
- Leave out generic events and getters another protocol may declare with another layout.
- Do not add an event that reuses a well-known **name** with other arguments (a `Mint` with an
  extra word, say) unless the processors are known to read that event by its full signature.

## `bound/`: bound to contracts of one chain

An ABI that cannot be chain-wide goes under `bound/` and is bound to the contracts it describes by
`bound/index.json`. A bound ABI is used for the contracts it is bound to and for nothing else, and
there it goes ahead of the chain-wide set.

**No file under `bound/` may end in `.abi`.** Processors released before the bound area existed
register every `.abi` file chain-wide, whatever directory it is in. Bound ABIs are `.json`.

```json
{
  "version": 1,
  "bindings": [
    {
      "name": "Aero Slipstream CLPool, Base",
      "abi": "bound/aerodrome/CLPool.json",
      "chain": 8453,
      "addresses": ["0x1642634c8cb1c9f1b84c9a1a08534f8c3f4e89f8"],
      "factories": ["0x9cb220581b91b0c83223012fe69688b92ad07a76"],
      "from_block": 0,
      "to_block": 0
    }
  ]
}
```

| field | meaning |
| --- | --- |
| `name` | What the contract is. For the reader and the log. |
| `abi` | Path of the ABI file from the repository root. A chain-wide `.abi` file may be named too: it is then registered chain-wide and bound. |
| `chain` | Network id. Required: the same address on two chains can be two contracts. |
| `addresses` | Contracts the ABI is the ABI of: a singleton (factory, manager, router), or the implementation that every clone of a pool delegates to. |
| `factories` | Contracts whose every creation has this ABI: the way to say "all pools of this factory" without listing them. |
| `from_block`, `to_block` | Heights the binding holds at, both inclusive. Optional; `0` or absent leaves that end open. Use them when a contract is upgraded: close the old binding and add one for the new ABI. |

Rules the processors enforce, each of which leaves the binding out and logs an error:

- `version` is `1`; an index of another version is not read at all.
- `chain` is set, and at least one of `addresses` and `factories` is not empty.
- Addresses are `0x` and 40 hex digits.
- The `abi` file is in the repository and parses.
- One contract of one chain has one ABI at any height: two bindings of the same address (or of the
  same factory) whose block ranges overlap are a mistake, and the later one is refused.

What a processor does with a binding:

- `addresses` is applied when a log or a call is decoded: the events and methods the ABI declares
  are decoded from it, with its argument names, its indexing and its outputs. A log emitted inside
  a delegated call is looked up under the emitting contract first and under the code address
  second, so binding a clone implementation covers every clone.
- `factories` is for code that already knows which factory created a contract. The block decoder
  does not know that, so **a `factories` entry alone does not change how a pool's logs and calls
  decode yet**. Bind the implementation address as well when the pools are clones.

## Adding or changing an ABI

1. Decide which kind it is. If any selector in it is declared elsewhere with other outputs or other
   indexing, it is a bound ABI, or a `.lite.abi` without those entries.
2. Run the staged tests of `streaming_processor_evm` against your checkout before pushing:
   `go test -run TestStagedAbiRepository ./evm/services/`, with the checkout beside the project or
   named by `ABI_REPOSITORY_DIR`. They fail on a selector two chain-wide files spell differently,
   on a `.abi` file under `bound/`, and on an index entry a processor would refuse.
3. After the merge, a processor logs `Loaded abis from the repository count=N bound=M` when it
   starts: `N` chain-wide files and `M` bindings.
