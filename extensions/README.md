# Non-Standard Extensions
If you have any suggestions, found an issue, or just confusing wording,
we are happy to accept pull requests, and / or issue tickets discussing these.

Standardization stages:
- open for proposals
- already some proposals: still accepting new proposals
- discussion: core concept fixed; designing extension
- remaining details: thinking about exact instruction encoding, exact definitions, ...
- under review: PR into the [ETC.a spec repo](http://github.com/eTC-A/etca-spec/) has been opened


## Draft Extensions
The goal of these draft extensions is to be standardized in the future.

These are likely to change in the future!

| name                           | abbr  | stage              | CPUID  | link |
| ------------------------------ | ----- | ------------------ | ------ | ---- |
| multi-core systems             | CORES | discussion         | CP2.8  | [WIP spec](./multi-core/spec.md)         |
| variable-vector-length vectors | VVEC  | already a proposal | CP2.9? | [overview](./variable-vectors/README.md) |


## Vendor Extensions
These are extensions that are / will be implemented by a hardware "vendor",
but are maintained in this repository.

| vendor | name                            | abbr  | stage              | CPUID  | link |
| ------ | ------------------------------- | ----- | ------------------ | ------ | ---- |
|        | network-on-chip                 | NOC   | discussion         | CP2.10 | [WIP spec](./vnd-noc/spec.md)       |
|        | async network-on-chip recv/send | ASNOC | already a proposal | ?      | [WIP spec](./vnd-async-noc/spec.md) |
