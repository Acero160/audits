![Project Logo](./password-store-logo.png)

# Audit Report for Project PasswordStore





 May 14, 2025

---





<!-- Your report starts here! -->

Prepared by: [Gabriel Acero](https://www.linkedin.com/in/gabriel-acero-52a508167/)
Lead Security Researcher:

- Gabriel Acero


# Table of Contents
- [Audit Report for Project PasswordStore](#audit-report-for-project-passwordstore)
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Storing the password on-chain makes it visable to anyone, and no longer private.](#h-1-storing-the-password-on-chain-makes-it-visable-to-anyone-and-no-longer-private)
    - [\[H-2\] ``PasswordStore::setPassword`` has no access controls, maening a non-owner could change the password.](#h-2-passwordstoresetpassword-has-no-access-controls-maening-a-non-owner-could-change-the-password)
  - [Informational](#informational)
    - [\[I-1\] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect](#i-1-the-passwordstoregetpassword-natspec-indicates-a-parameter-that-doesnt-exist-causing-the-natspec-to-be-incorrect)

# Protocol Summary

PasswordStore is a protocol dedicated to storage and retrieval of a user´s passwords. The protocol is designed to be used by a single user, and is not
designed to be used by multiple users. Only the owner should be able to set and access this password.

# Disclaimer

Gabriel Acero  makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.








# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

I use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

````
2e8f81e263b3a9d18fab4fb5c46805ffc10a9990
````

## Scope

```
./src/
#-- PasswordStore.sol
```

## Roles

- Owner: The user who can set the password and read the password.
- Outsiders: No one else should be able to set or read the password.

## Issues found

| Severity | Number of issues found |
|----------|-------------------------|
| High     |    2                      |
| Medium   |    0                      |
| Low      |    0                      |
| Info     |    1                      |
| Total    |    3                      |

# Findings

## High

### [H-1] Storing the password on-chain makes it visable to anyone, and no longer private.


**Description:** All data stored on-chain is visible to anyone, and can be read directly 
from the blockchain. The `PasswordStore::s_password` variable is intented to be a private variable and only 
accessed through the `PasswordStore::s_password` function, which is intented to be only called by the owner of the contract.

We show one such method of reading any data off chain below



**Impact:** Anyone can read the private password, severly breaking the functionality of the protocol.

**Proof of Concept:** (Proof of Code)

The below test case shows how anyone can read the password directly from the blockchain.

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought.
One could encrypt the password off-chain, and then store the encrypted password on-chain.
This would require the user to remember another password off-chain to decrypt the password. However,
you'd also likely want to remove the view function as you wouldn't want the user to accidentally send
a transaction with the password that decrypts your password.


---

### [H-2] ``PasswordStore::setPassword`` has no access controls, maening a non-owner could change the password.


**Description:** The `PasswordStore::setPassword` function is set to be and `external` function, however,
the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`

```javascript
    function setPassword(string memory newPassword) external {
@>       // @audit - There are no access controls
        s_password = newPassword;
        emit SetNetPassword();
    }
```


**Impact:** Anyone can set/change the password of the contract, severly breaking the contract intented functionality.


**Proof of Concept:** Add the following to the `PasswordStore.t.sol` test file.

<details>
<summary> Code </summary>

```javascript
     function test_anyone_can_set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        string memory expectedPassword = "myNewPassword";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }
```

</details>

**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.

```javascript
    if (msg.sender != owner) {
        revert PasswordStore_NotOwner();
    }
```

---

## Informational

### [I-1] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect


**Description:**

```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
     // @audit their is no newPassword parameter!
     * @param newPassword The new password to set.
     */  
    function getPassword() external view returns (string memory) {}
```

The `PasswordStore::getPassword` function signature is `getPassword()` which the natspec says it should be `getPassword(string)`.

**Impact:** The natspec is incorrect.

**Recommended Mitigation:** Remove the incorrect natspec line.

```diff
-    * @param newPassword The new password to set. 
```