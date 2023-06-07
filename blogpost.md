# Fee Ticketing - A Backstage Pass to Ethereum Blocks

**TL;DR - Users buy "tickets" from searchers/builders that, when redeemed, pay for inclusion. Such tickets can be used to interact with the chain using EOAs that aren't funded with ETH. This, while maintaining privacy, so that not even the searcher knows who has bought the redeemed ticket.**


## Today's landscape

One of the challenges of today's Ethereum landscape is privacy. <br/>
Users fund their accounts, get doxxed, set up new accounts, repeat.

Already Satoshi [recommended](https://bitcoin.org/bitcoin.pdf) to use new accounts for every single transaction. <br/>
Vitalik [mentioned](https://vitalik.ca/general/2023/01/20/stealth.html#conclusions) application-specific addresses for privacy reasons as well.
So what's the problem?

Well, managing numerous addresses is a complex task. Moreover, the temptation to trade one's privacy for additional functionalities/features has become more prevalent. 
Users love to link their accounts to [ENS](https://ens.domains/) names or own NFTs that they also use as Twitter profile pictures. 
This, coupled with major privacy protocols [facing sanctiones](https://home.treasury.gov/news/press-releases/jy0916), has contributed to a steady weakening of on-chain privacy.

Today, it has become increasingly challenging to uphold the necessary level of privacy required for secure financial transactions.
For example, if it's known that your EOA holds 1 million dollars in ETH, how much do you think would it cost to hire someone to forcibly obtain those funds and still make some nice profits?

This privacy issue extends to [stealth addresses](https://eips.ethereum.org/EIPS/eip-5564) as well. Stealth addresses can enhance privacy by creating unlinkable transactions, but they come with their own set of challenges. One such challenge is that the recipient's account isn't initially funded with ETH. If the sender doesn't attach sufficient ETH to the stealth address transaction, the recipient is unable to utilize it (aside from proving ownership) and must first fund the stealth address from another, potentially doxxed, account. Check out [Vitalik's blog post](https://vitalik.ca/general/2023/01/20/stealth.html) for more details on that topic.

<p align="center">
<img src="https://github.com/nerolation/Ethereum-ticket-system/assets/51536394/a509950b-d5dd-4093-8995-572e9cb8081e" />
</p>


Some privacy challenges boil down to the fact that accounts (not talking about [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337) SM wallets here) need to have a minimum amount of ETH to cover the computational costs they impose on the entire network - the gas costs - before they can do something on Ethereum. The same challenges apply for users that just want to hold their favorite NFTs without being bothered with our magic internet money at all or those that receive a grant in DAI, want to swap it to pay for their rent, but find themselves unable to do so without paying for at least one transaction to a CEX.

Fortunatelly, with the introduction of [PBS](https://ethresear.ch/t/proposer-block-builder-separation-friendly-fee-market-designs/9725), we can solve this issue by letting (minimally trusted) external parties, searchers and builders, handle this task. PBS split up the roles of block proposing and block building, effectively separating (light-weight) proposers from (heavy-weight) builders and searchers. By opening up a side-channel to pay for transaction inclusion of a transaction of one account but paying for that inclusion from another account, we enable unfunded EOAs to participate in the ecosystem.

## The challenge of maintaining privacy in an example

For the following, let's assume we are dealing with a user who has two accounts, `A` and `B`. 

Account `A` is funded and doxxed, while account `B` represents a just created account that has not yet done anything on Ethereum.
The user, who holds the private keys to these accounts, has received a grant payment of 1000 DAI to address `B`. For privacy reasons, the user didn't want to receive this money to address `A`.

<p align="center">
<img src="https://github.com/nerolation/Ethereum-ticket-system/assets/51536394/314054ac-9e6a-4e68-b834-56df5ef3bbb7" />
</p>

In this situation, the user can't really do anything with the 1000 DAI because he has no ETH in the account to swap the DAI to ETH or transfer the DAI to a CEX.
The user wants to avoid funding that account through account `A`, as this would create an on-chain link between `A` and `B` and undermine the reason why account `B` was set up in the first place.


**The user now has different options:**
1. Funnel some funds through Tornado Cash (or some similar mixing protocol).
2. Transfer from a CEX and accept `B` getting doxxed in front of the CEX.
3. Use the *Transaction Fee Ticketing* solution.

Let's assume the user is from the US, so prohibited from engaging with Tornado Cash, while not being a hardcore rebellious libertarian at the same time, so option 1 is not applicable.

Option 2 assumes that a CEX can be trusted(!) to be both safe and reliable, knowing that the user owns the money in account "B". So the user discards this option as well.

There remains option 3...

Having established the context, we'll now delve into a Transaction Fee Ticketing system in greater detail. This post is inspired by a [blog post](https://vitalik.ca/general/2023/01/20/stealth.html#stealth-addresses-and-paying-transaction-fees) from Vitalik, which outlines how such a system could function. One of the most attractive features of this ticketing concept is that it allows users to maintain their privacy while using unfunded EOAs.

## Fee Ticketing

The ticketing solution is based on David Chaum's ["Blind Signatures for Untraceable Payments"](https://link.springer.com/chapter/10.1007/978-1-4757-0602-4_18) concept from 1983, which cleverly enabled untraceable payments on centralized e-cash systems. Since then, the method has been used in various applications that demand privacy, such as digital voting, and, more cryptocurrency-related, [CoinJoin protocols](https://en.bitcoin.it/wiki/CoinJoin), such as [Wasabi Wallet](https://wasabiwallet.io/) on Bitcoin. Blind signatues come with some very cool properties that allows it to be used for the purpose of builing a gas fee ticketing solution.

**Basically it works as follows:**
* The user buys a "*ticket*" from a searcher/builder (e.g. builder0x69) for 0.0001 ETH using account `A`, while simultaneously providing the builder with some blinded information. The "blinding" is a cryptographic technique to prevent the builder from seeing the information in clear text. The information could theoretically be a hashed string saying "This is my first ticket" and only the user can unblind it again. The blinding requires a randomly generated number, the blinding factor, which is used to "blind" the information and is never shared.
* The builder verifies the receipt of 0.0001 ETH, signes the blinded info and returns the resulting singed blinded info to the user.
* The user, now using account `B`, can redeem the ticket by first unblinding the singed blinded info (effectively obtaining a valid signature from the builder) and then showing the signature to the builder, while at the same time providing the builder with the signed transaction sending 1000 DAI to a CEX.
* If the signature is valid, the builder can include the transaction on-chain, knowing that the user has paid for it via purchasing a ticket beforhand. In a last step, the builder records the used signature to prevent double-redemption.


<p align="center">
    <img src="https://github.com/nerolation/Ethereum-ticket-system/assets/51536394/f5aa830f-08bc-482a-a104-0f9c9e41e1bd" />
    <p align="center">(1) User generated random/arbitrary info and blinds it. (2) User sends it, together with some ETH, to the builder. (3) Builder confirms receipt and signes the blinded info. (4) Builder returns the singed blinded info. (5) User unblinds the singed blinded info and gets the builder's signature. (6) User, from an unfunded account, submits a zero-fee tx to the builder alongside the builder's signature. (7) Builder verifies die signature and includes the tx into a block.</p>
</p>
<br>

Importantly, the builder never saw the unblinded info but gets suddenly confronted with a valid signature of itself, which the users extracted from the singed blinded info the builder returned. Only by unblinding the returned singed blinded information, the signature can be extracted.
Seeing the signature, the builder can be confident that this user (remember, using account `B`) has the right to get a transaction included for redeeming a ticket, without knowing which exact ticket was redeemed.

To maintain the readability of this post without overloading it with code, I have chosen not to include concrete code examples. A PoC implemented in Python can be found in [this repository](https://github.com/nerolation/Ethereum-ticket-system/tree/main).

## Some Challenges Along the Road

Thinking more about this concept, one can identify several challenges, some of which might only be implementation details, whereas others a more fundatmental in nature.

#### Trust assumptions
As stated above, the ticketing requires some trust to be placed on searchers/builders, who could potentially rug pull you and run away with the tickets (basically ETH) that have not been redeemed yet. While one might argue that it is always better to go with the trustless solution, I'd agree with the [assessment](https://vitalik.ca/general/2023/01/20/stealth.html) of Vitalik, that, even though some trust in searchers is required, the quantity of funds involved is low, thus the advantages outweigh the disadvantages.

Even if users start purchasing tickets from many different builders/searchers, we're still talking about relatively modest amounts of money.
In fact, having tickets redeemable at the 4 largest builders (as of June '23: beaver, 0x69, flashbots and rsync) would give a probability of being included in the next block of >70%.

At this point, some might ask, "*Aren't you just suggesting to legitimize private order flows?*", and the answer is "*Yes, but with some some caveats*". It's important to note that this is not equivalent to the typical private order flows (POF) often found in the MEV context. These POFs usually involve private agreements between users (typically through wallet apps) and searchers with the intention of maximizing profits through exclusive access to MEV, which can threaten decentralization. The ticketing system, on the other hand, focuses on enhancing privacy for transaction that cannot be meaningfully front- or back runned. 

**It is more of a private POF that can do the job of giving a jump start to unfunded EOS.**<br/>
Of course, the concept can be expanded from there.

#### Builder vs. Proposer
The builder's block is only accepted and appended to the chain if the corresponding proposer selects it. 

The inclusion of empty-gas fee transaction comes with disadvantages for the searcher/builder because it would make their blocks less attractive for validators. The inclusion of zero-gas fee transactions could be disadvantageous for the searcher or builder as it might make their blocks less appealing to validators. For instance, if a block's entire fees are paid through such "side-channels", these blocks might never be selected by proposers, assuming the builder doesn't pass on the generated income to the block's coinbase.

However, the flip side of this situation is that builders securing this additional, potentially lucrative, revenue stream would have more resources to subsidize their own blocks, thereby creating a reverse effect. Users value their privacy and would be willing to pay more to have a more seamless yet confidential experience. Consequently, builders and searchers participating in the gas fee ticketing process could gain a competitive edge over others who choose not to participate.

#### User - Builder communication
Users should have the capability to purchase a ticket from builders by sending them Ethereum (ETH), which shouldn't pose a problem, and supplying them with blinded information. This blinded data is rendered irrelevant for any other user without access to the blinding factor (a random number used for the blinding). As such, this blinded info can be appended to the transaction as calldata without concerns.

On the other hand, the builder must also establish communication with the user to deliver the signed blinded info after confirming the receipt of the ETH, a process which may present certain challenges.

**Off-chain communication** - One solution could be the use of a private communication channel, such as a Telegram bot. Users would register with the builder's bot to receive the signed blinded information via a direct message. The hurdle here is that users need their blinding factor to unblind the received message content from the builder, a process that involves the modular multiplicative inverse. This is not particularly straightforward, in particularily not for everyday users.<br/>
It would probably be more straigt-forward to have an UI, hosted by the builder, that handles the communication part (users authenticate/login by signing a msg) and does doe math on the user's front-end.

**On-chain communication** - Another alternative would be similar to the ticket purchasing process but reversed: the builder could append the information to a self-send transaction (along with other signed blinded information) and have the users parse it and determine which signed blinded information is relevant to them. However, this method presents the same difficulties as the first example. Moreover, it adds additional costs to the process.

In fact, no additional encryption is needed in the communication above since without having the blinding factor, the signed blinded information would be meaningless to an observer.

A third point where communication is necessary occurs when the user wishes to redeem tickets by providing a builder's signature. Since this information is no longer hidden through the blinding factor, some additional encryption, perhaps utilizing the builder's public key, might be required by the user before appending it to a transaction and submitting it to the builder (assuming it is submitted as calldata through the peer-to-peer network). 
If the redemption is submitted along with the transaction through a private communication channel, additional encryption may not be necessary.

In general, establishing an appropriate communication channel between users and builder still seems to demand further effort at this stage.

#### Ticket Size and Change
For privacy reasons, the ticketing service requires having tickets of fixed sizes. 

Broadly speaking, a ticket should carry enough value (in terms of ETH) so that only a few tickets are needed to compensate the builder for successful inclusion. For smaller ticket sizes, a larger number of tickets would need to be redeemed at once. However, this wouldn't pose a significant issue as users can purchase and redeem multiple tickets at the same time.

For example, a user could buy 10 tickets by sending 0.001 ETH to the builder, along with 10 pieces of blinded information.
Dealing with change could be more complex since a link to the buyer can't be established at the time of redemption. Therefore, the change can only be directed to the same recipient specified during the redemption, or it can be collected by the searcher and/or forwarded to the proposer. 

Hence, the ticket size should be chosen to be small enough not to generate substantial amounts of change.

<p align="center">
<img src="https://github.com/nerolation/Ethereum-ticket-system/assets/51536394/cc2bd9fc-07be-4ede-8bc0-d3371f5c7744" />
</p>

## Furture Outlook
In summary, there're still a lot of efforts needed to solve the challenges outlined above.

The ticketing can be seen as a private private side-channel, and despite side-channels hiving a very bad reputation in the context of MEV (as they're a centralizing force), they can be used for the sake of increasing privacy, by making unlinkable transactions more convinient.

If you're a builder or builder that wants to experiment with it, contact me. I'm happy to help with the integration.
In particular, thinking of stealth addresses, the ticketing approach holds significant potential to improve UX - which is something I'd personally love to see.
Also, if you have any concerns or feedback, feel free to reach out on twitter/telegram - I'm happy to answer upcoming questions or engage in discussions.
