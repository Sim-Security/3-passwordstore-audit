### [H-1] Storing Variable on Chain Makes It Visible to Everyone and No Longer Private

**Description:** All data stored on-chain is visible to everyone and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword` function, which is intended to be called only by the owner of the contract.

We demonstrate one such method of reading any data off-chain below.

**Impact:** Anyone can read the private password, severely breaking the functionality of the protocol.

**Proof of Concept:**
The following test case shows how anyone can read the password directly from the blockchain.

1. **Create a Locally Running Chain**
    ```bash
    make anvil
    ```

2. **Deploy the Contract to the Chain**
    ```bash
    make deploy
    ```

3. **Run the Storage Tool**

    We use `1` because that's the storage slot of `s_password` in the contract.

    ```bash
    cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
    ```

    You'll get an output that looks like this:
    ```
    0x6d7950617373776f726400000000000000000000000000000000000000000014
    ```

    You can then parse that hex to a string with:

    ```bash
    cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
    ```

    And get an output of:
    ```
    myPassword
    ```

**Recommended Mitigation:** Due to this vulnerability, the overall architecture of the contract should be rethought. One approach is to encrypt the password off-chain and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. Additionally, you'd likely want to remove the view function to prevent users from accidentally sending a transaction with the password that decrypts your password.

---

### [H-2] `PasswordStore::setPassword` is Callable by Anyone 

**Description:** The `PasswordStore::setPassword` function is set to be an `external` function. However, the NatSpec of the function and the overall purpose of the smart contract indicate that **"This function allows only the owner to set a new password."**

```javascript
function setPassword(string memory newPassword) external {
@>  // @audit - There are no access controls here
    s_password = newPassword;
    emit SetNetPassword();
}
```

**Impact:** Anyone can set/change the password of the contract.

**Proof of Concept:** 

Add the following to the `PasswordStore.t.sol` test suite.

<details>
<summary>Code</summary>

``` javascript
function test_anyone_can_set_password(address randomAddress) public {
    vm.prank(randomAddress);
    string memory expectedPassword = "myNewPassword";
    passwordStore.setPassword(expectedPassword);
    vm.prank(owner);
    string memory actualPassword = passwordStore.getPassword();
    assertEq(actualPassword, expectedPassword);
}
```

</details>

**Recommended Mitigation:** Add an access control modifier to the `setPassword` function. 

``` javascript
if (msg.sender != s_owner) {
    revert PasswordStore__NotOwner();
}
```

---

### [I-1] The `PasswordStore::getPassword` NatSpec Indicates a Parameter That Doesn't Exist, Causing the NatSpec to Be Incorrect

**Description:** 

``` javascript
/*
 * @notice This allows only the owner to retrieve the password.
@> * @param newPassword The new password to set.
 */
function getPassword() external view returns (string memory) {
```

The NatSpec for the function `PasswordStore::getPassword` indicates it should have a parameter with the signature `getPassword(string)`. However, the actual function signature is `getPassword()`.

**Impact:** The NatSpec is incorrect.

**Recommended Mitigation:** Remove the incorrect NatSpec line.

```diff
-     * @param newPassword The new password to set.
```

