# Liquidity

What is Liquidity?

It's a concept sometimes called various names in Lightning:

- Capacity
- Balance
- Bandwidth

The Lightning Network is a network of payment channels. In payment channels you can send back and forth as much capital as has been committed to the Blockchain at the start of the channel. This is because every updated balance is represented by a signed transaction that can be published to the Blockchain and the Blockchain consensus will reject transactions that pay out more money than was paid in.

Within that starting boundary capital can flow back and forth between you and your peer. Your capital at the moment dictates how much you can send towards your peer, their capital dictates how much they can send in your direction.

Because Lightning is passing funds across from peer to peer but network node balances are private, your local capital dictates an upper bound of how much you can send out across the network to arbitrary destinations and does not dictate a lower bound. For example if two peers are only connected to each other, their ability to send to the greater Lightning Network is zero.

Liquidity is therefore generally discussed in the context of an abstract greater network rather than in directly observable balances. Liquidity is the capital amount that you can use to effectively send and receive across the broadly connected Lightning Network.

Liquidity is both a qualitative and quantitative measurement since the ability to send is subject to prohibitively expensive routing fees, slow peer relays, poorly connected nodes, etc. There are also two types of liquidity in this context: capital that is pointed in your direction that can be used for receiving and capital that is pointed outwards that can be used for sending.

You can call receiving liquidity "inbound" and sending liquidity "outbound".

Liquidity objectives:

- Get *inbound* liquidity when you want to be able to *receive* funds.
- Get *outbound* liquidity when you want to be able to *send* funds.
- Get both *inbound* and *outbound* when you want to forward payments.

Of course, you can also use the Lightning Network payment channels for a non-networked use case where you only send and receive with close peers. That is a lot easier but lacks the benefits of distributed transfer potential.

## Outbound

The outbound liquidity of your Lightning node is not just the balance of your channels, it is the balance of your channels modified by the ability of your peers to forward to the greater Lightning Network.

That forwarding ability is a constantly changing but invisible factor, so you can only get a general sense of it when you are sending and more generally you'll need to reply on a rough reputational equation where you assume some nodes have a high probability of being able to forward your payments at any given time.

Increasing your outbound liquidity means increasing the local balance of good liquidity channels:

- Create new channels with well-connected routing peers
- Receive funds off-chain on well-connected channels
- Circular rebalance funds from poorly connected to well connected peers

This begs the question: how do I know what a well-connected peer is? The intuitive and often popular answer to evaluate the graph of nodes and look for highly connected nodes is also the wrong answer. The reason for this is that the graph doesn't show you routing performance and it doesn't show you capital allocation. A node can appear well connected but have no capital or provide poor forwarding services.

So how do you evaluate good connectivity of routing peers?

- Observe passively your past payment activity. Who routed for you?
- Actively attempt probe payments. Who can deliver probes to their destination?
- Evaluate social proofs of nodes. Is the operator known? Are they recommended?

Improving outbound liquidity can actually involve closing channels because not all peers can deliver good routing service to the greater network.

Minimize your costs of increasing outbound liquidity:

- Create outbound channels in batch transactions with many outputs, few inputs
- Only use the Blockchain during low-fee periods such as weekends
- Try and receive regular inflows of money off-chain instead of on-chain
- Serve as a routing peer to nodes that are receiving more than they are sending

Getting good outbound liquidity is generally fairly easy since most routing nodes will be glad to accept a new incoming channel from you.

## Inbound

Your inbound liquidity represents the capital of other people. Because this is literally other people's money, it's not quite as straightforward to control as your local outbound.

Just like outbound liquidity, the quality of inbound liquidity is difficult to ascertain and is constantly changing as capital flows in different ways through the Lightning Network.

Increasing your inbound liquidity means increasing the remote balance of good liquidity channels. This generally requires some kind of recompense to various remote parties that must hold capital in your direction.

How to get inbound liquidity:

- Peers earning fees from forwards to you may add liquidity in your direction
- You can engage in a [submarine swap][1] to transfer off-chain funds for on-chain
- You can purchase an inbound channel from someone
- You can arrange to create a dual funded channel with someone
- Try and send regular outflows of money off-chain instead of on-chain
- Serve as a routing peer for nodes that send out more than they receive
- Circular rebalance poor inbound liquidity to good inbound liquidity

High quality inbound liquidity is a scarce resource of Lightning. For routing nodes, it is an asset that is being translated into routing fees. For merchants, it is required to receive payments.

[1]: https://wiki.ion.radar.tech/tech/research/submarine-swap
