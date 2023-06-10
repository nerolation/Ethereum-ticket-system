# Gas Ticketing - A Backstage Pass to Ethereum Blocks

**TL;DR - Users buy "tickets" from "coordinators" that, when redeemed, fund an unfunded EOA. This, while maintaining privacy, so that not even the coordinator knows who has bought the redeemed ticket.**

*Thanks to [Matt (lightclient)](https://twitter.com/lightclients), [Guillaume](https://twitter.com/gballet), [Matt Solomon](https://twitter.com/msolomon44), [Barnabe](https://twitter.com/barnabemonnot), [Chris Hager](https://twitter.com/metachris) and [Vitalik](https://twitter.com/VitalikButerin) for feedback and review!*

## Today's landscape

One of the challenges of today's Ethereum landscape is privacy. <br/>
Users fund their accounts, get doxxed, set up new accounts, repeat.

Managing numerous addresses is a complex task and it has become increasingly challenging to uphold the necessary level of privacy required for secure financial transactions.

This privacy issue extends to [stealth addresses](https://eips.ethereum.org/EIPS/eip-5564) as well. Stealth addresses can enhance privacy by creating unlinkable transactions, but they come with the challenge that the recipient's account isn't initially funded with ETH. If the sender doesn't attach sufficient ETH to the stealth address transaction, the recipient is unable to utilize it and must first fund the stealth address from another, potentially doxxed, account. Check out [Vitalik's blog post](https://vitalik.ca/general/2023/01/20/stealth.html) for more details on that topic.

<center>
<img src="https://github.com/nerolation/Ethereum-ticket-system/assets/51536394/a509950b-d5dd-4093-8995-572e9cb8081e" />
</center><br>


Some privacy challenges boil down to the fact that EOAs need to have a minimum amount of ETH to cover the gas costs before they can do something on Ethereum.

With EIP-1559 and the introduction of a protocol-enforced base fee it became impossible to submit zero gas transaction.
However, we can circumwent this by letting a (minimally trusted) external party, the coordinator, handle the funding an unfunded EOA by purchasing and redeeming a "gas ticket". While doing so, we're able to effecively break the link between the user's doxxed account (buying the ticket) and the user's unfunded account (redeeming the ticket), therby offering increased privacy.


## The challenge of maintaining privacy in an example

For the following, let's assume we are dealing with a user who has two accounts, `A` and `B`. 

Account `A` is funded and doxxed, while account `B` represents a just-created account that has not yet done anything on Ethereum.
The user, who holds the private keys to these accounts, has received a grant payment of 1000 DAI to address `B`. For privacy reasons, the user didn't want to receive this money to address `A`.

<center>
<img src="https://github.com/nerolation/Ethereum-ticket-system/assets/51536394/314054ac-9e6a-4e68-b834-56df5ef3bbb7" />
</center><br>

In this situation, the user can't do anything with the 1000 DAI because he has no ETH in the account to swap the DAI to ETH or transfer the DAI to a CEX.
The user wants to avoid funding that account through account `A`, as this would create an on-chain link between `A` and `B` and undermine the reason why account `B` was set up in the first place.


**The user now has different options:**
1. Funnel some funds through Tornado Cash (or some similar mixing protocol).
2. Transfer from a CEX and accept `B` getting doxxed in front of the CEX.
3. Use the *Gas Ticketing* solution.

Let's assume the user is from the US, so prohibited from engaging with Tornado Cash, while not being a hardcore rebellious libertarian at the same time. Then, using Tornado Cash would not be an option.<br/>
Apart from the risk of violating sanctions (or increasing the anonymity set of malicious parties), zk-SNARK-based mixers come with high gas costs (approx. 1m gas for depositing + withdrawing), as well as demanding some sophistication from the user as users are required to locally construct a Merkle proof for the withdrawal. So, option 1 is not applicable for this use case.

Option 2 assumes that a CEX can be entirely trusted — both in terms of security and reliability — with the knowledge that the user owns the funds in account "B". So the user discards this option as well.

There remains option 3...

Having established the context, we'll now delve into a Gas Ticketing concept in greater detail. This post is inspired by a [blog post](https://vitalik.ca/general/2023/01/20/stealth.html#stealth-addresses-and-paying-transaction-fees) from Vitalik, which outlines how such a system could function. One of the most attractive features of this ticketing concept is that it allows users to maintain their privacy while using unfunded EOAs.

## Gas Ticketing

The ticketing solution is based on David Chaum's ["Blind Signatures for Untraceable Payments"](https://link.springer.com/chapter/10.1007/978-1-4757-0602-4_18) concept from 1983, which cleverly enabled untraceable payments on centralized e-cash systems. Since then, the method has been used in various applications that demand privacy, such as digital voting, and, more cryptocurrency-related, [CoinJoin protocols](https://en.bitcoin.it/wiki/CoinJoin), such as [Wasabi Wallet](https://wasabiwallet.io/) on Bitcoin. Blind signatures come with some very cool properties that allow them to be used for building a gas fee ticketing solution.

**It works as follows:**
* The user buys a "*ticket*" from a coordinator for 0.001 ETH using account `A`. To do so, the user sends the ETH along with some blinded data to the coordinator. This "data" could theoretically be a hashed string saying "*This is my first ticket*". The "*blinding*" is a cryptographic technique to prevent the coordinator from seeing the provided data in clear text. The blinding requires a randomly generated number, the *blinding factor*, which is used to "*blind*" the data and is kept secret by the user.
* The coordinator verifies receipt of 0.001 ETH, signs the blinded data, and returns the *signed blinded data* to the user.
* The user, now using account `B`, can redeem the ticket by first "*unblinding*" the signed blinded data (effectively obtaining a valid signature from the coordinator) and then showing the signature to the coordinator, while at the same time providing the coordinator with the the to-be-funded account holding the 1000 DAI.
* If the signature is valid, the coordinator can fund the provided account, knowing that the user has paid for it via purchasing a ticket beforehand. In the last step, the coordinator records the used signature to prevent double redemption.




<center>
    <img src="https://hackmd.io/_uploads/Syi-XsWDn.png" />
    <p align="center">(1) User-generated random/arbitrary data and blinds it. (2) User sends it, together with some ETH, to the coordinator. (3) Coordinator confirms receipt and signs the blinded data. (4) Coordinator returns the signed blinded data. (5) User unblinds the signed blinded data and gets the coordinator's signature. (6) User, from an unfunded account, submits a to-be-funded address alongside the builder's signature and the ticket in clear text. (7) Builder verifies the signature and funds the provided address with the ticket value.</p>
</center>
<br>

Importantly, the coordinator never saw the unblinded data but gets suddenly confronted with a valid signature of itself, which the users extracted from the signed blinded data the coordinator returned. Only by unblinding the returned signed blinded data, the signature can be extracted.
Seeing the signature, the coordinator can be confident that this user (remember, using account `B`) has the right to get some account funded for redeeming a ticket, without knowing which exact ticket was redeemed.

To maintain the readability of this post without overloading it with code, I have chosen not to include concrete code examples. A PoC implemented in Python can be found in [this repository](https://github.com/nerolation/Ethereum-ticket-system/tree/main).

## Some Challenges Along the Road

Thinking more about this concept, one can identify several challenges, some of which might only be implementation details, whereas others a more fundamental in nature.

### Trust assumptions
As stated above, the ticketing requires some trust to be placed on coordinator, who could potentially rug-pull you and run away with the tickets (basically ETH) that have not been redeemed yet. While one might argue that it is always better to go with the trustless solution, I'd agree with the [assessment](https://vitalik.ca/general/2023/01/20/stealth.html) of Vitalik, that, even though some trust in searchers is required, the quantity of funds involved is low, thus the advantages outweigh the disadvantages.


Nevertheless, it remains challenging to identify trustworthy coordinators. Malicious coordinator could sell tickets without providing signatures or funding addresses, so we'd have to rely on some reputation system for coordinators.

### Coordinator profits
Coordinators could charge some fees for that service, therby opening up an additional, potentially lucrative, revenue stream. Users value their privacy and would be willing to pay more to get their unfunded accounts funded privatly.

### Ticket Size and handling Change
For privacy reasons, the ticketing service requires having tickets of fixed sizes. 

Broadly speaking, a ticket should carry enough value (in terms of ETH) so that only a few tickets are needed to compensate the coordinator for funding the unfunded EOA. For smaller ticket sizes, a larger number of tickets would need to be redeemed at once. However, this wouldn't pose a significant issue as users can purchase and redeem multiple tickets at the same time.

For example, a user could buy 10 tickets by sending 0.001 ETH to the builder, along with 10 pieces of blinded data.
There is no need to hanle "change" because the coordinator, after deducing some fees, could just use the whole ticket value to fund the address specified by the user.


### User - Coordinator Communication
Users should have the capability to purchase a ticket from the coordinator by sending ETH and supplying them with blinded data. This blinded data is rendered irrelevant for any other user without access to the blinding factor (a random number used for the blinding). As such, this blinded data can be appended to the transaction as calldata without concerns.

On the other hand, the builder must also establish communication with the user to deliver the signed blinded data after confirming the receipt of the ETH, a process that may present certain challenges.

**Off-chain communication** - One solution could be the use of a private communication channel. Users would send an API request to the coordinator to send the txhash including the ticket purchase and the blinded ticket.
After that, the user directly reveices back the signed blinded data (the coordinator verifys receipt) and unblind it using blinding factor, a process that involves the modular multiplicative inverse.

When the user wishes to redeem tickets by providing a builder's signature and the "data" in clear text, another API endpoint at the coordinator could be set up for that purpose to avoid exposing the "data" and the clear text ticket, because, if exposed, others could front run the user and redeem the ticket.


<center>
<img src="https://github.com/nerolation/Ethereum-ticket-system/assets/51536394/cc2bd9fc-07be-4ede-8bc0-d3371f5c7744" />
</center><br>

## Furture Outlook
In summary, there's still a lot of effort needed to solve the challenges outlined above.

The ticketing can be seen as a side channel, and despite side channels having a very bad reputation in the context of MEV (as they're a centralizing force), they can be used for the sake of increasing privacy, by making unlinkable transactions more convinient.

If you're planning to become a coordinator, contact me. I'm happy to help with the integration.
In particular, thinking of stealth addresses, the ticketing approach holds significant potential to improve UX - which is something I'd personally love to see.
Also, if you have any concerns or feedback, feel free to reach out on [twitter](https://twitter.com/nero_eth)/telegram - I'm happy to answer upcoming questions or engage in discussions.
