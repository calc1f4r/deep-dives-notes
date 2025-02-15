### Issues which can happen due to external call

1. Reentrancy: CEI pattern and non-reentrant considerations

   - Check for multiple contract reentrancy
   - For read-only reentrancy, ensure view functions reading state are safe

2. DOS - Denial of Service attack

   - Avoid sending ether to a contract with no payable function

3. Return value vulnerabilities:

   - Validate all possible return values
   - Check for unexpected bytes
   - Be aware of return bump attacks

4. Gas: If not trusted, note that external calls receive 63/64 of the available gas.
   - If the external call consumes all the forwarded gas, the call will fail.
   - Ensure proper error handling for cases where the external call fails due to insufficient gas.
   - Consider setting a proper gas stipend or using alternative patterns if more gas is needed.

Recomendation :
Enforce post check invarinets
