---
layout: post
title:  "Extracting sensitive data from Ethereum logs"
date:   2021-02-01 15:11:51 +0530
categories: CTF blockchain
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, I will explain how one can extract sensitive information from ethereum [event logs][ethereum-logs]. This challenge was a part of the Offshift CTF, a CTF focused on Ethereum smart contracts.

<h4><u>Given smart contract</u></h4>
{% highlight ruby %}
pragma solidity ^0.6.0;

contract secure_enclave {
    event pushhh(string alert_text);
    struct Secret {
        address owner;
        string secret_text;
    }
    mapping(address => Secret) private secrets;
    function set_secret(string memory text) public {
        secrets[msg.sender] = Secret(msg.sender, text);
        emit pushhh(text);
    }
    function get_secret() public view returns (string memory){
        return secrets[msg.sender].secret_text;
    }
}
{% endhighlight %}

<h4><u>Solution</u></h4>
a. The contract source code has two functions defined: set_secret() and get_secret(). The set_secret() function stores the message sent by the msg.sender (caller of the function), and then emits the event pushhh.  
b. I initially thought that if I spoof my msg.sender to be the address of the contract creator, I will get the flag with the get_secret() function, but it wasn't the case. The solution is to get all the logs of the contract.  
c. To get all the logs, we have to interact with the contract using web3 (a JS library). My code for the challenge is as follows:

```javascript
let Web3 = require('web3');
const web3 = new Web3(new Web3.providers.HttpProvider('https://rinkeby.infura.io/v3/<KEY>'));

//CONNECT TO THE DEPLOYED CONTRACT USING THE ABI OF THE CONTRACT
const instance = new web3.eth.Contract(
	[
        {
          "anonymous": false,
          "inputs": [
            {
              "indexed": false,
              "internalType": "string",
              "name": "alert_text",
              "type": "string"
            }
          ],
          "name": "pushhh",
          "type": "event"
        },
        {
          "inputs": [],
          "name": "get_secret",
          "outputs": [
            {
              "internalType": "string",
              "name": "",
              "type": "string"
            }
          ],
          "stateMutability": "view",
          "type": "function"
        },
        {
          "inputs": [
            {
              "internalType": "string",
              "name": "text",
              "type": "string"
            }
          ],
          "name": "set_secret",
          "outputs": [],
          "stateMutability": "nonpayable",
          "type": "function"
        }
      ],
	'0x9B0780E30442df1A00C6de19237a43d4404C5237'
);	
instance.getPastEvents('pushhh', {fromBlock: 0, toBlock: 'latest'}, function(err, events){
	if(err)
		console.log(err);
	else
		console.log(events);
});
```

d. Now when you run the contract, all the logs will be fetched and printed. The flag is in one of them.     
flag : flag{3v3ryth1ng_1s_BACKD00R3D_0020}             

[remix-link]: https://remix.ethereum.org/
[ethereum-logs]: https://medium.com/linum-labs/everything-you-ever-wanted-to-know-about-events-and-logs-on-ethereum-fec84ea7d0a5
