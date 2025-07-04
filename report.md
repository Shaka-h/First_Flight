---
title: Protocol Audit Report
author: Miriam Shaka
date: May 7, 2025
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
    {\Large\itshape Miriam Shaka\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle


# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Core Invariant](#core-invariant)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Wrong calculation of `payPerTeacher` at `LevelOne::graduateAndUpgrade` which breaks the payment structure.](#h-1-wrong-calculation-of-payperteacher-at-levelonegraduateandupgrade-which-breaks-the-payment-structure)
    - [\[H-2\] System upgrade can be done even when no review has been made on the students, this breaks one of the invariants](#h-2-system-upgrade-can-be-done-even-when-no-review-has-been-made-on-the-students-this-breaks-one-of-the-invariants)
    - [\[H-3\] There is no check for `LevelOne::cutOffScore` for students who are qualified to graduate, this breaks one of the invariants](#h-3-there-is-no-check-for-levelonecutoffscore-for-students-who-are-qualified-to-graduate-this-breaks-one-of-the-invariants)
  - [Low](#low)
    - [\[L-1\] There is no check for `sessionEnd` before an upgrade is done which allows the System to upgrade before school's `sessionEnd` has reached](#l-1-there-is-no-check-for-sessionend-before-an-upgrade-is-done-which-allows-the-system-to-upgrade-before-schools-sessionend-has-reached)

# Protocol Summary

Welcome to Hawk High, enroll, avoid bad reviews, and graduate!!!

You have been contracted to review the upgradeable contracts for the Hawk High School which will be launched very soon.

These contracts utilize the UUPSUpgradeable library from OpenZeppelin.

At the end of the school session (4 weeks), the system is upgraded to a new one.

# Core Invariant

- A school session lasts 4 weeks
- For the sake of this project, assume USDC has 18 decimals
- Wages are to be paid only when the graduateAndUpgrade() function is called by the principal
- Payment structure is as follows:
  - principal gets 5% of bursary
  - teachers share of 35% of bursary
  - remaining 60% should reflect in the bursary after upgrade
- Students can only be reviewed once per week
- Students must have gotten all reviews before system upgrade. System upgrade should not occur if any student has not gotten 4 reviews (one for each week)
- Any student who doesn't meet the cutOffScore should not be upgraded
- System upgrade cannot take place unless school's sessionEnd has reached

# Disclaimer

This report is provided on a best-effort basis. While significant effort was made to uncover vulnerabilities, this audit does not guarantee the complete absence of issues. The audit focuses on the Solidity smart contracts and does not constitute an endorsement of the business model.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.


## Scope 

- In Scope:
```
./src/
#--LevelOne.sol
#--LevelTwo.sol
```


## Roles

- Principal: In charge of hiring/firing teachers, starting the school session, and upgrading the system at the end of the school session. Will receive 5% of all school fees paid as his wages. can also expel students who break rules.
- Teachers: In charge of giving reviews to students at the end of each week. Will share in 35% of all school fees paid as their wages.
- Student: Will pay a school fee when enrolling in Hawk High School. Will get a review each week. If they fail to meet the cutoff score at the end of a school session, they will be not graduated to the next level when the Principal upgrades the system.

# Executive Summary

This audit reviewed the Hawk High core contracts: LevelOne and LevelTwo. A total of 4 issues were identified. Most issues are solvable with minor code changes

## Issues found


| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 0                      |
| Low      | 1                      |
| Info     | 0                      |
| Gas      | 0                      |
| Total    | 4                     |

# Findings

## High

### [H-1] Wrong calculation of `payPerTeacher` at `LevelOne::graduateAndUpgrade` which breaks the payment structure.

**Description:** The `graduateAndUpgrade` function is responsible for upgrading the system to level two and paying wages to the principal at 5% and 35% shared among the teachers evenly. However the function does not share the 35% evenly to every teacher

```js
    function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }

        uint256 totalTeachers = listOfTeachers.length;

        uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;

        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo); // @audit this seems to nothing

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
    }
```


**Impact:** This causes the contract to break the following invariant
- Payment structure is as follows:
  - `principal` gets 5% of `bursary`
  - `teachers` share of 35% of bursary
  - remaining 60% should reflect in the bursary after upgrade

<details>
<summary> Proof of Code </summary>

Add the following in the `LevelOneAndGraduateTest.t.sol`

```js
    function test_can_pay_wages() public schoolInSession{
        levelTwoImplementation = new LevelTwo();
        levelTwoImplementationAddress = address(levelTwoImplementation);
        uint256 remainingBursary = (levelOneProxy.bursary() * 60) / levelOneProxy.PRECISION();

        bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

        vm.prank(principal);
        levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);

        assertEq(remainingBursary, usdc.balanceOf(address(levelOneProxy)));
    }
```
</details>

**Tools Used** Manual Review and Foundry

**Recommended Mitigation:** This update should be made on the `graduateAndUpgrade` function

```diff
    function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }

        uint256 totalTeachers = listOfTeachers.length;

-       uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
+       uint256 payPerTeacher = ((bursary * TEACHER_WAGE) / PRECISION)/totalTeachers;

        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo); // @audit this seems to nothing

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
    }
```

### [H-2] System upgrade can be done even when no review has been made on the students, this breaks one of the invariants

**Description:** One of the invariants states `Students must have gotten all reviews before system upgrade. System upgrade should not occur if any student has not gotten 4 reviews (one for each week)`. However this check is not made on the `LevelOne::graduateAndUpgrade` 

**Impact:** This allows system upgrade to be made on an going session without students receiving reviews for graduation

**Proof of Concept:**
1. Add teachers
2. Enroll students
3. Start session
4. Upgrade system without making any review


<details>
<summary> Proof of Code </summary>

Add the following in the `LevelOneAndGraduateTest.t.sol`

```js
    
    function test_cant_upgrade_without_all_reviews() public schoolInSession {
        vm.warp(block.timestamp + 4 weeks);

        levelTwoImplementation = new LevelTwo();
        levelTwoImplementationAddress = address(levelTwoImplementation);

        bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

        vm.prank(principal);
        levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);

    }
```
</details>

**Tools Used** Manual Review and Foundry

**Recommended Mitigation:** Add a check on the `graduateAndUpgrade` function to ensure that all enrolled students are done a review before upgrading the system


### [H-3] There is no check for `LevelOne::cutOffScore` for students who are qualified to graduate, this breaks one of the invariants

**Description:** One of the invariants states `Any student who doesn't meet the `cutOffScore` should not be upgraded`. However this check is not made on the `LevelOne::graduateAndUpgrade` 

**Impact:** This allows any student to graduate even without passing the cutoffscore

**Proof of Concept:**
1. Add teachers
2. Enroll students
3. Start session
4. Upgrade and graduate without checking students cutoffscore


<details>
<summary> Proof of Code </summary>

Add the following in the `LevelOneAndGraduateTest.t.sol`

```js
    
    function test_cant_upgrade_without_all_reviews() public schoolInSession {
        vm.warp(block.timestamp + 4 weeks);

        levelTwoImplementation = new LevelTwo();
        levelTwoImplementationAddress = address(levelTwoImplementation);

        bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

        vm.prank(principal);
        levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);

    }
```
</details>

**Tools Used** Manual Review and Foundry

**Recommended Mitigation:** Use `cutOffScore` to add a check on the `graduateAndUpgrade` function to ensure that only students who pass the cutOffScore graduate


## Low 

### [L-1] There is no check for `sessionEnd` before an upgrade is done which allows the System to upgrade before school's `sessionEnd` has reached

**Description:** One of the invariants states `System upgrade cannot take place unless school's `sessionEnd` has reached`. However this check is not made on the `LevelOne::graduateAndUpgrade` 

**Impact:** This allows the System to upgrade before school's `sessionEnd` has reached

**Proof of Concept:**
1. Add teachers
2. Enroll students
3. Start session
4. Upgrade and graduate without checking session has ended


<details>
<summary> Proof of Code </summary>

Add the following in the `LevelOneAndGraduateTest.t.sol` which fails since the time `graduateAndUpgrade` is called does not match `sessionEnd`

```js
    
    function test_cant_upgrade_before_sessionEnd() public schoolInSession {
        uint256 sessionEnd = levelOneProxy.getSessionEnd();

        levelTwoImplementation = new LevelTwo();
        levelTwoImplementationAddress = address(levelTwoImplementation);

        bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

        vm.prank(principal);
        levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);
        assertEq(sessionEnd, block.timestamp);

    }
```
</details>

**Tools Used** Manual Review and Foundry

**Recommended Mitigation:** Add a check on the `graduateAndUpgrade` function to ensure that the sessionEnd has reached before upgrading the system
