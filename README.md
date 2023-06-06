# Ethereum-ticket-system
The following code is a PoC of the ticket-system descibed by Vitalik in [this](https://vitalik.ca/general/2023/01/20/stealth.html) blog post.

Essentially, users register a ticket at searchers (or builder) by sending them small abounts of funds together with a blinded message. The searcher verifies that the funds arrived and then signes the (chaumian) blinded message (blinded ticket) and returns it to the user. The user can then broadcast a transaction not paying any fees (within the transaction). The searcher would pick it up, include it into a block and invalidate the used blinded ticket/signature.
