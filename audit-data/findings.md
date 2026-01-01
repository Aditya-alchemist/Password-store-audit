

## [H-1] Storing the password on-chain makes it publicly readable and no longer private

### Description

All data stored on-chain is publicly accessible and can be read directly from the blockchain. The variable `PasswordStore::s_password` is intended to be private and only accessible through the `PasswordStore::getPassword` function, which itself is intended to be callable only by the contract owner.

However, despite this intention, any user can read the password directly from contract storage without interacting with the contract logic. One such off-chain method is demonstrated below.

### Impact

Any user can read the supposedly private password, completely breaking the confidentiality guarantees of the protocol and rendering the password mechanism ineffective.

### Proof of Concept

The following steps demonstrate how private on-chain data can be read directly:

1. Start a local blockchain:

```bash
make anvil
```

2. Deploy the contract:

```bash
make deploy
```

3. Read the storage slot containing the password:

```bash
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 1 --rpc-url http://127.0.0.1:8545
```

Example output:

```text
0x6d7950617373776f726400000000000000000000000000000000000000000014
```

4. Decode the `bytes32` value into a string:

```bash
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

5. Output:

```text
myPassword
```

This demonstrates that the password can be retrieved directly from on-chain storage without any authorization.

### Recommended Mitigation

The contract architecture should be redesigned to avoid storing sensitive information in plaintext on-chain. One possible approach is to encrypt the password off-chain and store only the encrypted value on-chain. The user would then be responsible for securely managing the decryption key off-chain.

Additionally, the `getPassword` view function should likely be removed, as returning the decrypted password on-chain defeats the purpose of encryption and risks accidental disclosure via transactions.

---

## [H-2] `PasswordStore::setPassword` lacks access control, allowing anyone to overwrite the password

### Description

The `PasswordStore::setPassword` function is declared as `external`, and according to its natspec documentation, it is intended to allow **only the owner** to set a new password. However, no access control is implemented.

```solidity
function setPassword(string memory newPassword) external {
    s_password = newPassword;
    emit SetNetPassword();
}
```

### Impact

Any user can call `setPassword` and overwrite the stored password, completely bypassing the intended access control and compromising the protocol.

### Proof of Concept

```solidity
function test_Anyone_can_set_password(address random) public {
    vm.assume(random != owner);
    vm.prank(random);

    string memory newPassword = "hackedPassword";
    passwordStore.setPassword(newPassword);

    vm.prank(owner);
    string memory actualPassword = passwordStore.getPassword();

    assertEq(actualPassword, newPassword);
}
```

This test demonstrates that a non-owner can successfully overwrite the password.

### Recommended Mitigation

Restrict access to `setPassword` by adding an `onlyOwner` modifier (or equivalent access control mechanism).

```solidity
function setPassword(string memory newPassword) external onlyOwner {
    s_password = newPassword;
    emit SetNetPassword();
}
```

---

## [I-1] Incorrect natspec documentation in `PasswordStore::getPassword`

### Description

The natspec documentation for `PasswordStore::getPassword` references a parameter that does not exist in the function signature.

```solidity
/*
 * @notice This allows only the owner to retrieve the password.
 * @param newPassword The new password to set.
 */
function getPassword() external view returns (string memory) {
```

The function does not accept any parameters, making the natspec misleading.

### Impact

Incorrect documentation can confuse developers, auditors, and integrators, potentially leading to incorrect assumptions or usage of the contract.

### Recommended Mitigation

Remove the invalid natspec parameter annotation.

```diff
- @param newPassword The new password to set.
```

---

