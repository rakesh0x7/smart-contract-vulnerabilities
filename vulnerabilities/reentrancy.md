## Reentrancy

Reentrancy is an attack that can occur when a bug in a contract may allow a malicious contract to reenter the contract unexpectedly during execution of the original function. This can be used to drain funds from a smart contract if used maliciously. In fact, reentrancy was the attack vector used in the DAO hack.

### Single function reentrancy

A single function reentrancy attack occurs when a vulnerable function is the same function that an attacker is trying to recursively call.

```
// UNSECURE
function withdraw() external {
    uint256 amount = balances[msg.sender];
    (bool success,) = msg.sender.call{value: balances[msg.sender]}("");
    require(success);
    balances[msg.sender] = 0;
}
```

Here we can see that the balance is only modified after the funds have been transferred. This can allow a hacker to call the function many times before the balance is set to 0, effectively draining the smart contract.

### Cross-function reentrancy

A cross-function reentrancy attack is a more complex version of the same process. Cross-function reentrancy occurs when a vulnerable function shares state with a function that an attacker can exploit.

```
// UNSECURE
function transfer(address to, uint amount) external {
  if (balances[msg.sender] >= amount) {
    balances[to] += amount;
    balances[msg.sender] -= amount;
  }
}

function withdraw() external {
  uint256 amount = balances[msg.sender];
  (bool success,) = msg.sender.call{value: balances[msg.sender]}("");
  require(success);
  balances[msg.sender] = 0;
}
```

In this example, a hacker can exploit this contract by having a fallback function call `transfer()` to transfer spent funds before the balance is set to 0 in the `withdraw()` function.

### Read-only Reentrancy

Read-only reentrancy is a novel attack vector in which instead of reentering into the same contract in which state changes have yet to be made, an attacker reenters into another contract which reads from the state of the original contract.

```
// UNSECURE
contract A {
	// Has a reentrancy guard to prevent reentrancy
	// but makes state change only after external call to sender
	function withdraw() external nonReentrant {
		uint256 amount = balances[msg.sender];
		(bool success,) = msg.sender.call{value: balances[msg.sender]}("");
		require(success);
		balances[msg.sender] = 0;
	}
}

contract B {
	// Allows sender to claim equivalent B tokens for A tokens they hold
	function claim() external nonReentrant {
		require(!claimed[msg.sender]);
		balances[msg.sender] = A.balances[msg.sender];
		claimed[msg.sender] = true;
	}
}
```

As we can see in the above example, although both functions have a nonReentrant modifier, it is still possible for an attacker to call `B.claim` during the callback in `A.withdraw`, and since the attackers balance was not yet updated, execution succeeds.

### Reentrancy prevention

The simplest reentrancy prevention mechanism is to use a [`ReentrancyGuard`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol), which allows you to add a modifier, e.g. `nonReentrant`, to functions which may otherwise be vulnerable. Although effective against most forms of reentrancy, it's important to understand how read-only reentrancy may be used to get around this and to always use the **checks-effects-interactions pattern**.

For optimum security, use the **checks-effects-interactions pattern**. This is a simple rule of thumb for ordering smart contract functions.

The function should begin with *checks*, e.g. `require` and `assert` statements.

Next, the *effects* of the contract should be performed, i.e. state modifications.

Finally, we can perform *interactions* with other smart contracts, e.g. external function calls.

This structure is effective against reentrancy because when an attacker reenters the function, the state changes have already been made. For example:

```
function withdraw() external {
  uint256 amount = balances[msg.sender];
  balances[msg.sender] = 0;
  (bool success,) = msg.sender.call{value: balances[msg.sender]}("");
  require(success);
}
```

Since the balance is set to 0 before any interactions are performed, if the contract is called recursively, there is nothing to send after the first transaction.


Examples from: https://medium.com/coinmonks/protect-your-solidity-smart-contracts-from-reentrancy-attacks-9972c3af7c21


### Sources

- https://consensys.github.io/smart-contract-best-practices/attacks/reentrancy/
- https://medium.com/@gus_tavo_guim/reentrancy-attack-on-smart-contracts-how-to-identify-the-exploitable-and-an-example-of-an-attack-4470a2d8dfe4
- https://medium.com/coinmonks/protect-your-solidity-smart-contracts-from-reentrancy-attacks-9972c3af7c21