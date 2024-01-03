# PasswordStore_Audit


<br/>
<p align="center">
<img src="./password-store-logo.png" width="400" alt="password-store">
</p>
<br/>

#### A smart contract application for storing a password. Users should be able to store a password and then retrieve it later. Others should not be able to access the password. 
💻 Security Review code: https://github.com/Cyfrin/3-passwordstore-audit/tree/onboarded

## Findings 👇🏽

### [H-1] Storing variables on chain makes it visible to anyone, thus no longer private

**Description:** All data stored on chain is visible to anyone, and it can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be private and only accessed through the `PasswordStore::getPassword` function, which is intended
to be only called by the owner of the contract.

We show one such method of reading any data of chain below.

**Impact:** Anyone can read the private password, severely breaking the functionality of the protocol

**Proof of Concept:**(Proof of code)
the below example shows how can anyone read the password from the blockchain.

1. Create a locally running chain

```bash
make anvil
```

2. Deploy the contract to the chain

```
make deploy
```

3. Run the storage tool

We use `1` here because that's the storage slot of `s_password` in the contract.

```
cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
```

You will get and output that looks like this
`0x6d7950617373776f726400000000000000000000000000000000000000000014`

Then you can parse that hex to a string with:

```
cast --parse-bytes23-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

And get an output of

```
myPassword
```

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and the store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password on-chain. However we would also likely want to remove the view function as we won't want the user to accidentaly send a transaction with the password that decrypts your password.

### [H-2] `PasswordStore::setPassword` has no access control, meaning a non-owner could change the password

**Description:** The `PasswordStore::setPassword` function is set to be `external` function, however, the natspec of the function says `This function allows only owner to set a new password.`

```javascript

    function setPassword(string memory newPassword) external {
 @>       // @audit access control bug
        s_password = newPassword;
        emit SetNetPassword();
    }

```

**Impact:** Anyone can set/change the password, severly breaking the contract's intended functionality.

**Proof of Concept:** Add the following to `PasswordStore.t.sol` test file.

```javascript
    function test_anyone_can_Set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        string memory expectedPassword = "myNewPassword";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }

```

**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.

```javascript
if(msg.sender!=s_owner){
    revert PasswordStore__NotOwner();
}

```

### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect.

**Description:** The `PasswordStore::getPassword` function signature is `getPassword()` while the natspec says should be `getPassword(string)`

```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
@>   * @param newPassword The new password to set.
     */

    function getPassword() external view returns (string memory) {}
```



**Impact:** Natspec is incorrect

**Recommended Mitigation:** Remove the incorrect natspec line.

```diff
-  * @param newPassword The new password to set.
```



