# Better Contracts Design

### More code, more bugs

- More code does not necessarily mean more vulnerabilities – prioritize quality over quantity.
  - Note: Keep the codebase simple and avoid unnecessary complexity.
- Vulnerabilities tend to increase in a parabolic manner as code complexity grows.
  - Note: Use modular design and thorough testing to limit risks.
- Focus on clean, well-designed contracts to mitigate risks.
  - Note: Regular audits and refactoring are key.

#### Use fewer storage variables

- Remove unnecessary storage variables.
  - Note: Only retain state variables critical to functionality.
- It will help in less solidity code to less charge for auditing firms.
  - Note: Optimized storage can lower audit and execution costs.
- Less mismatches.
  - Note: Ensure variable consistency and clear naming conventions.

#### logic which can be done off chain ?

- Ensure if the computatation is very high then it must be performed off chain!
  - Note: Offload compute-intensive tasks to off-chain systems to save gas costs.

#### Make the code as simple as possible

### Use for-loops

using for loops : leads to dos, there can be o(1) used for mapping

#### use parallel data structures

like enumerablemapping and enumberableset
