##  web3.js基本开发环境搭建及常用查询函数

### 1.环境准备

 - 安装nodejs
```  
node -v
npm -v
```  
 - 安装Visual Studio Code
 - 在vs code中安装code runner插件

### 2. 安装web3.js
```  
npm install web3 
```  

### 3. web3.js常用查询  
 web3.js中文文档： https://learnblockchain.cn/docs/web3.js/  
 
```  
# 连接区块链
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider("区块链接口"));
``` 

以下列举的都是适用于chain33的常用查询接口  
 - Web3.modules: 返回包含所有子模块类的对象，主要用到Eth，Net，Personal子模块，Shh, Bzz这些用不上
   - Eth: 用来和区块链网络交互
   - Net: 网络属性相关
   - Persion: 区块链账户相关
 - Web3.version: 所引用的Web3的版本号  
 - web3.currentProvider：返回当前通信服务提供器
 - web3.eth.currentProvider：同上，chain33兼容eth的连接
 - web3.eth.getChainId(): 返回区块链节点的chainid
 - web3.eth.net.getId(): 获取当前网络的ID
 - web3.eth.net.isListening(): 当前节点是否处于监听连接状态，是的话返回true
 - web3.eth.net.getPeerCount(): 获取对等节点的数量
 - web3.eth.net.isListening(): 当前节点是否处于监听连接状态，是的话返回true
 - web3.eth.getHashrate():  返回区块的difficulty值
 - web3.eth.getGasPrice():  返回当前gas价格
 - web3.eth.getBlockNumber():  返回当前最大的区块高度
 - web3.eth.getBlock("区块高度/区块hash值", true/false): 返回区块信息,true/false: 要不要带交易详情
 - web3.eth.getBlockTransactionCount(区块高度): 返回对应区块中交易数量
 - web3.eth.getTransaction("交易hash"): 返回对应交易hash的交易对象
 - web3.eth.getTransactionReceipt("交易hash"): 返回交易收据
 - web3.eth.getTransactionCount("要查询的交易地址"): 返回交易数量

##  web3.js账户操作和发送交易

### 1. 账户信息操作
- web3.eth.getAccounts():  返回当前节点控制的账户，如果本地节点没有创建账户, 则报错返回：ErrAccountNotExist
- 节点上通过chain33-cli命令行工具创建钱包
```  
- web3.eth.personal.newAccount("账户别名") ： 返回创建的账户地址，如果别名有重复，则报错返回：ErrLabelHasUsed，注意：参数和web3.js原生的有区别
- web3.eth.personal.importRawKey("用户私钥", "账户别名")： 将给定的私钥导入节点上的钱包，返回导入私钥的账户地址
- web3.eth.getBalance("用户地址"): 返回用户地址下的余额， 精度10的18次方

### 2. 签名发送转账交易
方式一（不推荐）：使用保存在节点上的私钥签名并发送交易上链。  私钥存在节点上不是安全的做法，容易导致私钥泄漏。   
方式二（推荐）： 利用私钥在链下签名，签名完成后再通过接口发送到区块链上。  

- web3.eth.accounts.signTransaction： 本地（链下）签名交易
- web3.eth.sendSignedTransaction： 发送已经签好名的交易到区块链上

## web3.js合约部署

### 1. 合约编写
- 编写一个ERC1155的智能合约，继承了OpenZeppelin ERC1155和Ownable合约
- 合约中有两个私有映射变量 _tokenSupply 和 _tokenURIs，分别用于存储每个ERC1155代币的供应量和URI
- 在构造函数中，传入了基础URI，用于确定代币的元数据
- mint 函数可供所有者调用来铸造新的ERC1155代币。该函数调用了 _mint 函数将新代币的供应量添加到特定帐户中
- _tokenSupply 和 _tokenURIs 映射都会更新，并触发 TokenMinted 事件。 该事件记录了代币 ID，代币供应量和代币 URI。tokenId 参数使用 indexed 修饰符进行标记，以便在事件日志中可以更快速地搜索并访问该参数。 
```  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.3.0/contracts/token/ERC1155/ERC1155.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.3.0/contracts/access/Ownable.sol";

contract MyERC1155 is ERC1155, Ownable {

    mapping(uint256 => uint256) private _tokenSupply;
    mapping(uint256 => string) private _tokenURIs;

    event TokenMinted(uint256 indexed tokenId, uint256 supply, string uri);

    constructor(string memory uri) ERC1155(uri) {}

    function mint(uint256 tokenId, uint256 supply, string memory uri) public onlyOwner {
        _mint(msg.sender, tokenId, supply, "");
        _tokenSupply[tokenId] += supply;
        _tokenURIs[tokenId] = uri;
        emit TokenMinted(tokenId, supply, uri);
    }

    function tokenSupply(uint256 tokenId) public view returns (uint256) {
        return _tokenSupply[tokenId];
    }

    function uri(uint256 tokenId) public view override returns (string memory) {
        return _tokenURIs[tokenId];
    }
}
```  

## web3.js合约读写

### 1. 向合约内写入数据
#### 通证mint
mint函数调用流程  

```  
// 1. 读取ABI文件
var abi = JSON.parse(fs.readFileSync(path.join(__dirname, "../contract.abi")).toString())

// 2. provider连接
var web3 = new Web3(new Web3.providers.HttpProvider("http://172.22.16.19:8545"));

// 3. 获得合约地址
var contractAddress = '合约地址';

// 4. 创建合约instance
var tx = new web3.eth.Contract(abi, contractAddress);

// 5. mint操作
var mintTx = tx.methods.mint(tokenId, supply, uri);
const mint = async () => {
  const createTransaction = await web3.eth.accounts.signTransaction(
    {
      to: contractAddress,
      data: mintTx.encodeABI(),
      gas: await mintTx.estimateGas({from: "来源方地址"}),
    },
    "来源方私钥"
  );
  const createReceipt = await web3.eth.sendSignedTransaction(createTransaction.rawTransaction);
  console.log(createReceipt);
};
mint();
```  

mint函数并传入以下三个参数(tokenId: 通证的编号，用整数表示; supply: 通证的供应量，整数; uri: 通证元数据信息）。 其中uri代表通证元数据信息，它的标准参考： https://eips.ethereum.org/EIPS/eip-1155#metadata, 以下是一个uri示例，但实际的URI格式各有区别，具体取决于通证发行者的需求。 
```  
{
  "name": "MyToken",  -- 通证名称
  "description": "My custom ERC1155 token",  -- 通证的描述
  "image": "https://example.com/token-image.png",  -- 通证图像的URL
  "external_url": "https://example.com/token",  -- 指向通证相关网站的URL，没有的话，这个字段可以删除
  "attributes": [                               -- 通证的属性数组
    {
      "trait_type": "color",  -- 属性的类型
      "value": "blue"         -- 属性的值
    },
    {
      "trait_type": "size",
      "value": "medium"
    }
  ]
}
```  

#### 通证tranfer
- 通过合约实例使用methods.safeTransferFrom函数并传入参数(from: 通证来源方地址; to: 通证去向方地址: id: 通证ID, amount: 要转移的通证数量)来生成转移通证的交易
- 再通过from地址的私钥，签名上述交易并发送到链上，实现通证转移

transfer函数调用流程  
```  
// 转账操作
var transferTx = tx.methods.safeTransferFrom("来源地址", "去向地址", 通证ID, 转让数量, "0x");

const transfer = async () => {

  const transferTransaction = await web3.eth.accounts.signTransaction(
    {
      to: contractAddress,
      data: transferTx.encodeABI(),
      gas: await transferTx.estimateGas({from: "来源地址"}),
    },
    "来源地址钥"
  );

  const createReceipt = await web3.eth.sendSignedTransaction(transferTransaction.rawTransaction);
};
transfer();
```  

### 2. 合约信息查询
```  
// 查询操作
const getInfo = async () => {
  // 查询from地址下的余额
  const fromValue = await tx.methods.balanceOf("地址", 通证ID).call();
  console.log(`The current balance of from is: ${fromValue}`);

  // 查询to地址下的余额
  const toValue = await tx.methods.balanceOf("地址", 通证ID).call();
  console.log(`The current balance of to: ${toValue}`);

  // 查询URI信息
  const uri = await tx.methods.uri(tokenId).call();
  console.log(`The current balance of to: ${uri}`);
};
getInfo();
```  
##  web3.js合约事件

事件是Solidity合约中的一种通信机制，它允许合约在执行过程中发出通知，以便应用程序或其他合约可以监听这些通知并采取相应的行动。

事件通常用于以下情况：
1. 交易完成后向外部系统发送通知。
2. 监听合约状态变化，例如合约中某个值的更新。
3. 记录合约的行为，以便后续分析或审计。

以前面介绍的NFT合约为例：
```  
event TokenMinted(uint256 indexed tokenId, uint256 supply, string uri);

function mint(uint256 tokenId, uint256 supply, string memory uri) public onlyOwner {
        _mint(msg.sender, tokenId, supply, "");
        _tokenSupply[tokenId] += supply;
        _tokenURIs[tokenId] = uri;
        emit TokenMinted(tokenId, supply, uri);
    }
```  
在上面的代码中，TokenMinted事件用于记录mint函数中的值更改，并向应用程序发出通知。indexed关键字用于指定事件参数在事件日志中进行索引，这使得应用程序可以更轻松地搜索和过滤事件日志。

通过监听事件，应用程序可以自动更新状态或响应合约的状态变化，这使得合约更加灵活和可扩展。

### 1. 查询合约历史事件  
web3.eth.getPastLogs()  
过滤器对象：
- fromBlock: 起始区块（最小值支持从1开始）
- toBlock: 终止区块， 这个参数不带代表一直到最大区块
- address: 合约地址
- topics: 日志项中的主题值数组

示例： 
```  
web3.eth.getPastLogs({
    address: contractAddress,
    topics: ['0x30b042f6d29a39b7c10c2dfbd81016db37ea3d11e72ec5764fb0d0f7a1012a3a'],
    fromBlock: 71,
})
.then(console.log);

[
  {
    address: '0x61b35E8804F87589fBb7da3A6B68c563C05Bb2f1',
    topics: [
      '0x30b042f6d29a39b7c10c2dfbd81016db37ea3d11e72ec5764fb0d0f7a1012a3a',
      '0x0000000000000000000000000000000000000000000000000000000000002718'
    ],
    data: '........',
    blockNumber: 79,
    transactionHash: '0xc5bb3cd573e04e5561d262c394218bc3ad3457f6f0aa2f8101464ebb108700fb',
    transactionIndex: 0,
    blockHash: '0x7b0b108c8b55bdefb5edd6a1ce1ec05180f3cd0b58aa373ff88e9682ab694b7a',
    logIndex: 1,
    removed: false,
    id: 'log_bab13c4d'
  }
]
```  

### 2. 订阅区块链中的指定事件  
web3.eth.subscribe(type [, options] [, callback]);  
- String - 订阅类型
- Mixed - (可选) 依赖于订阅类型的可选额外参数
- Function - (可选) 可选的回调函数，其第一个参数为错误对象，第二个参数为结果

```  
web3.eth.subscribe('logs', {
    address: contractAddress,
    topics: ['0xc3d58168c5ae7397731d063d5bbf3d657854427343f4c083240f7aacaa2d0f62'],
}, function(error, result) {
    if (!error) {
        console.log(result);
    } else {
        console.error(error)
    }
})
.on("connected", subscriptionId => {
    console.log(`Subscribed with ID: ${subscriptionId}`);
  });
```  

### 3. 根据事件签名计算topics的值  
```  
var eventName = 'TokenMinted(uint256,uint256,string)'
var topic = web3.eth.abi.encodeEventSignature(eventName)
```  
