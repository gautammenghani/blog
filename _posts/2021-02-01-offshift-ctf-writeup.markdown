---
layout: post
title:  "Offshift CTF writeup (2021)"
date:   2021-02-01 15:11:51 +0530
categories: CTF blockchain
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
This post has writeups for two blockchain challenges : 'sanity check' and 'secure enclave'. 

`1. SANITY CHECK`  
This is an introductory challenge whose objective is to explain how the solidity smart contracts are compiled and also how to interact with a contract that is already deployed on the ethereum network.

a. Go to [Remix][remix-link]. Replace the 1_Storage.sol file contents with the source code of the challenge.  
b. In the compiler tab, select the compiler version 0.7.0, and click on compile.  
c. If the compilation was successful, go to the 'deploy and run transactions tab'. Paste the contract address given in the challenge description and click on 'At address' as shown below.
                <img src="{{ site.baseurl }}/assets/images/1_sanity_chk.png" alt="remix_ss">

  <br />
d. Expand the loaded contract and click on the welcome button. You will recieve the flag.
  
`2. SECURE ENCLAVE`  
The objective of this challenge is to explain how [events and logs][ethereum-logs] work in Ethereum.  

a. The contract source code has two functions defined: set_secret() and get_secret(). The set_secret() function stores the message sent by the msg.sender (caller of the function), and then emits the event pushhh.  
b. I initially thought that if I spoof my msg.sender to be the address of the contract creator, I will get the flag with the get_secret() function, but it wasn't the case. The solution is to get all the logs of the contract.  
c. To get all the logs, we have to interact with the contract using web3 (a JS library). My code for the challenge is as follows:

{% highlight ruby %}
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
{% endhighlight %}

d. Now when you run the contract, all the logs will be fetched and printed. The flag is in one of them.     flag : flag{3v3ryth1ng_1s_BACKD00R3D_0020}             

[remix-link]: https://remix.ethereum.org/
[ethereum-logs]: https://medium.com/linum-labs/everything-you-ever-wanted-to-know-about-events-and-logs-on-ethereum-fec84ea7d0a5
