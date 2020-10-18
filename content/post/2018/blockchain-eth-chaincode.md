---
title: "以太坊智能合约开发入门"
date: 2018-05-07
categories: ["区块链"]
tags: ["区块链", "以太坊"]
---

## 搭建智能合约开发环境

智能合约的开发环境依赖于node/npm，所以在构建开发环境之前需要确保开发机器已经安装好了node/npm的环境了。

如何还没有安装，那么在mac下面可以通过brew命令非常快速的安装
`brew install node`
或者自己下载node的安装包安装。
然后分别安装下面的几个开发智能合约需要用到的工具:
1.  Web3 JS - 开发以太坊客户端的javascript框架
`npm install web3`
2.  truffle - 以太坊开发框架
`npm install -g truffle`
3.  Metamask  - 以太坊钱包，基于Chrome的插件
4.  Ganache - 用于本地的虚拟以太坊区块链， 并且自动创建了用于测试的10个本地的区块链账号
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203449.png) 
5.  Remix - [https://remix.ethereum.org/](https://remix.ethereum.org/) 以太坊官方推荐的智能合约开发IDE,可以在浏览器中快速部署测试智能合约。
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203514.png) 

配置Metamask连接Ganache本地环境，Ganache默认的监听端口为7545，所以需要设置Metamask，如下，点击 “Custom RPC”，然后设置本地虚拟区块链的地址

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203635.png)



 ![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203715.png)



配置完之后，就可以导入Ganache默认创建的账号，

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203741.png) 

点击右侧的钥匙(show key), 从弹出的界面中拷贝账号的token，然后通过点击Metamask右上角的头像，通过"Import Account"，粘贴刚才复制的token，导入，就可以看到新导入的账号，并且该账号已经默认的有了100个ETH了。

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203753.png)

 

## 创建智能合约

在Remix编辑器中编写一个简单的智能合约 - HelloWorld
```
pragma solidity ^0.4.0;

contract HelloWorld {
   string public name;

   function HelloWorld() public {
       name = "Liping";
   }

   function setName(string _name) public {
       name = _name;
   }
}
```
合约编好之后，点击 "Run" 标签，在“Environment”中选择"Web3 Provider"

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203817.png) 

点击“OK”后，填入"http://localhost:7545"，连接我们在本地运行的Ganache虚拟区块链

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203831.png) 

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203845.png) 

连接成功之后，在“Account”下拉列表中会列出区块链中已经存在的账号：

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203858.png) 

点击"Create"，则刚刚编写的智能合约代码就部署到了虚拟区块链上了

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203914.png) 

下一步，将通过Web3 JS 和 Node.js 来编写改智能合约的客户端。

## 通过Web3 JS访问智能合约

通智能合约的通行，可以通过Web3来实现。Web3 JS通过RPC远程调用与以太坊节点进行连接。然后通过Web3中的eth对象与以太坊进行交互。其过程主要为：

1.  初始化Web3
2.  设置区块链账号
3.  设置智能合约Abi
4.  设置智能合约的地址
5.  创建智能合约对象
6.  通过智能合约对象与部署好的合约进行交互

完整的代码(javascript)如下
```
 // 初始化 Web3
  if (typeof web3 !== 'undefined') {
      web3 = new Web3(web3.currentProvider);
  } else {
      web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:7545'));
  }

  // 设置默认账户
  web3.eth.defaultAccount = web3.eth.accounts[0];
  // 设置合约 ABI
  var contractAbi = [
      {
          "constant": false,
          "inputs": [
              {
                  "name": "_name",
                  "type": "string"
              }
          ],
          "name": "setName",
          "outputs": [],
          "payable": false,
          "stateMutability": "nonpayable",
          "type": "function"
      },
      {
          "inputs": [],
          "payable": false,
          "stateMutability": "nonpayable",
          "type": "constructor"
      },
      {
          "constant": true,
          "inputs": [],
          "name": "name",
          "outputs": [
              {
                  "name": "",
                  "type": "string"
              }
          ],
          "payable": false,
          "stateMutability": "view",
          "type": "function"
      }
  ];

  // 设置合约地址
  var contractAddress = '0x01ee312a2055f212e09aa43e1b8d61f5634ccfc3';
  // 获取智能合约对象
  var contract = web3.eth.contract(contractAbi).at(contractAddress);
  // 调用合约对象的方法，返回name
  contract.name(function(err, value) {
      $('#name').html(value);
  })

  // 提交name的值
  $('form').on('submit', function(event) {
      event.preventDefault;
      contract.setName($('#input_name').val());
  });
```
代码比较简单，用到了web3和jQuery.

重点要解释的是合约的ABI，以及如何获取合约Abi和合约地址。

什么是合约ABI呢？ABI是Application Binary Interface的缩写，是是能合约的二进制接口，可以通俗的理解为描述智能合约的接口说明。当合约被编译后，那么它的abi也就确定了。

如何获取合约ABI和合约地址呢？

这个通过Remix 浏览器IDE可以非常容易就获取到，在“Compiler”标签，点击“Details”按钮，就可以在弹出的页面上看到“ABI”的这个区域，点击右侧的复制图标，就可以将ABI复制到粘贴板了。

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018203936.png) 

ABI的内容可以从上面的代码中看到，一个简单的JSON描述文件。

如何获取合约的部署地址呢？

同样的通过Remix 浏览器IDE来获取，上面我们是在“Run”标签页中通过“Create”来部署智能合约的，当点击“Create”按钮后，在标签页下方会显示合约名称 at XXXXX的下拉列表框，这个就是我们部署好的合约，点击复制图标，就可以将该合约的地址复制到粘贴板，

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018204010.png) 

通过合约的ABI和合约的地址后，就可以构建合约对象了，然后通过该合约对象，就可以完成智能合约的一些操作。

Web3 API中文文档  http://web3.tryblockchain.org/index.html

## 创建界面

通过上面的代码，已经初始化了web3，创建了contract 对象，所以只要写一个简单的HTML文件就可以完成一个非常简单的界面了

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018204028.png)



​                            ![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018204049.png) 

 

在输入框中输入其他的内容，点击"Say Hello To" 按钮，可以看到合约返回的name值相应的变成了输入框中输入的值了。

通过Ganache的Blocks和Transactions界面，可以看到每次的操作都会形成相应的区块和事务记录。

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018204124.png) 



![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201018204145.png)


