# Unexpected Balances in Smart Contracts

Unexpected balances in smart contracts can lead to significant accounting issues when the contract's logic relies on the balance for its operations. Two prominent examples of such issues are "force-feeding a contract" and inflation attacks.

## Force-Feeding

Force-feeding is a technique where Ether is sent directly to a smart contract without invoking any of its functions. This can disrupt the contract's internal accounting mechanisms, particularly if the contract relies on balance checks for its logic.

In typical smart contract operation, Ether is sent to a contract via a transaction that calls a payable function or invokes the `receive()` or `fallback()` functions. If a contract lacks these functions, transactions sending Ether to it will normally be reverted, ensuring the contract does not inadvertently receive funds.

Force-feeding bypasses this by sending Ether in a manner that doesn't require calling the contract's functions, thereby avoiding the checks and logic coded in Solidity.

### Force Feeding Methods

#### Block Rewards and Coinbase

In Proof of Stake systems like Ethereum, validators earn block rewards for successfully adding blocks to the blockchain. These rewards are sent to an address known as the **coinbase address**. Validators typically set this address to their own wallets, but an attacker-validator can set it to a target smart contract’s address.

Since block reward transfers are handled at the protocol level, they bypass Solidity-level checks. As a result, the target contract receives Ether directly as part of the block reward, regardless of any Solidity-coded restrictions.

## Inflation Attacks

Consider an inflation attack on a standard ERC-4626 vault. For the attack to work, the attacker has to send funds directly to the vulnerable ERC-4626 vault. This action disrupts the accounting mechanism and for this matter, the attacker is able to get away with funds belonging to other users of the vault.

Detailed explanation of inflation attacks can be found [here](https://www.youtube.com/watch?v=3IMw7xbxJgY)


## Mitigation Strategies

### 1. Avoid Using the Contract’s Balance Directly

Instead of relying on `address(this).balance` for critical logic, maintain an internal accounting system. For example:

```solidity
/// DO NOT USE IN PRODUCTION
/// ONLY MEANT TO SERVE AS AN EXAMPLE

mapping(address => uint256) internalBalances;

function deposit() external payable {
    internalBalances[msg.sender] += msg.value;
}

function withdraw(uint256 amount) external {
    require(internalBalances[msg.sender] >= amount, "Insufficient balance");
    internalBalances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);
}

function getBalance(address user) external view returns (uint256) {
    return internalBalances[user];
}
```

### 2. Immediate Funds Transfer

Instead of holding funds within the contract, immediately transfer received Ether to a secure, off-contract storage or another smart contract that handles funds securely:

```solidity
address payable public safeAddress = payable(0xSafeAddress);

receive() external payable {
    safeAddress.transfer(msg.value);
}
```

### 3. Event-Based Balance Tracking

Use events for deposit and withdrawal tracking, enabling off-chain monitoring and cross-verification of the contract’s balance:

```solidity
event Deposit(address indexed from, uint256 amount);
event Withdrawal(address indexed to, uint256 amount);

function deposit() external payable {
    internalBalances[msg.sender] += msg.value;
    emit Deposit(msg.sender, msg.value);
}

function withdraw(uint256 amount) external {
    require(internalBalances[msg.sender] >= amount, "Insufficient balance");
    internalBalances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);
    emit Withdrawal(msg.sender, amount);
}
```

Off-chain systems can then monitor these events and compare them with the on-chain state to ensure integrity.

## Sources
- [Solidity coinbase address](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#block-and-transaction-properties)
