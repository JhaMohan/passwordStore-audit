---
title: Protocol Audit Report
author: Aditya Mohan Jha
date: December 28, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Protocol Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Aditya Mohan\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Aditya](https://www.linkedin.com/in/aditya-mohan-884131b4/)
Lead Security Researcher: 
- Aditya

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Storing the password on-chain make visible to anyone.](#h-1-storing-the-password-on-chain-make-visible-to-anyone)
    - [\[H-2\] `PasswordStore::setPassword` has no access controls,meaning a non-owner could change the password](#h-2-passwordstoresetpassword-has-no-access-controlsmeaning-a-non-owner-could-change-the-password)
  - [Informational](#informational)
    - [\[I-1\] The  `PasswordStore::getPassword` natspec indicate a parameter that doesn't exist,causing the natspec to be incorrect.](#i-1-the--passwordstoregetpassword-natspec-indicate-a-parameter-that-doesnt-existcausing-the-natspec-to-be-incorrect)

# Protocol Summary

PasswordStore is a protocol designed to store and retrieve password. This protocol is designed to be used by a single user.Only owner can set and access the password.

# Disclaimer

The YOUR_NAME_HERE team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

**The findings described in this document correspond the following commit hash**

Commit Hash:
```
2e8f81e263b3a9d18fab4fb5c46805ffc10a9990
```

## Scope 
```
./src/
#- PasswordStore.sol
```


## Roles

- Owner: The user who can  set the password and read the password.
- Outsiders: No one else should set and read the password.


# Executive Summary
## Issues found
| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 2                      |
| Medium   | 0                      |
| Low      | 1                      |
| Info     | 0                      |
| Total    | 3                      |



# Findings

## High

### [H-1] Storing the password on-chain make visible to anyone.

**Description:** All the data stored on-chain is visible to anyone,and can be read directly from blockchain.The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword` function, which is intended to be only called by the owner of the contract.

**Impact:** Anyone can read the private password, severely breaking the functionality of the protocol.

**Proof of Concept:** 

The below test case shows how anyone can read the password directly from the blockchain.

1. Create a locally running chain
```bash
make anvil
```

2. Deploy the contract to the chain
```bash
make deploy
```

3. Run the storage tool
```bash
forge inspect PasswordStore storage
```
We use `1` because that's the storage slot of `s_password` in the contract.

```bash
cast storage <CONTRACT_ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
```

You'll get the output which will look like this:

`0x6d7950617373776f726400000000000000000000000000000000000000000014`

You can then parse that hex to string with:

```bash
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

And get an output of:

```
myPassword
```

**Recommended Mitigation:** Due to this,the overall architecture of the contract should be rethought.One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password.However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password.



### [H-2] `PasswordStore::setPassword` has no access controls,meaning a non-owner could change the password

**Description:** The `PasswordStore::setPassword` function is set to be an `external` function,however, the natspec of the function and overall purpose of smart contract is that `This function allows only the owner to set a new password.`

```javascript
 function setPassword(string memory newPassword) external {
  @>    // @audit - There are no access controls
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set/change the password of the contract,severely breaking the contract intended functionality.
 
**Proof of Concept:** Add the following to the `PasswordStore.t.sol` test file.
<details>
<summary>Code</summary>

```javascript
function test_anyone_can_set_password(address randomUser) public {
        vm.assume(randomUser != owner);
        vm.prank(randomUser);
        string memory expectedPassword = 'newPassword';
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }
```

</details>


**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.

<details>
<summary>Code</summary>

```javascript
      if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
```
</details>





## Informational

### [I-1] The  `PasswordStore::getPassword` natspec indicate a parameter that doesn't exist,causing the natspec to be incorrect.

**Description:** 

```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
@>   * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
```

The `PasswordStore::getPassword` function signature is `getPassword()` which the natspec say it should be `getPassword(string)`

**Impact:** The natspec is incorrect

**Recommended Mitigation:** Remove the incorrect natspec line

```diff
-  * @param newPassword The new password to set.
```