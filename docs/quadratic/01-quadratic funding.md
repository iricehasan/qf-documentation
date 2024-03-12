# What Is Quadratic Funding?

Quadratic Funding is a mechanism for allocating funds in a decentralized and democratic manner. It is designed to support public goods, such as open-source software, research, and community initiatives, by leveraging contributions from both large and small donors. The key idea behind Quadratic Funding is to match donations to projects not linearly, but quadratically, amplifying the impact of smaller contributions. Check out [Wtfisqf](https://www.wtfisqf.com/) for in depth explanation.

1. **Fairness and Inclusivity**
 
Quadratic Funding aims to create a fair and inclusive funding environment by giving smaller donors a proportionally larger voice. This ensures that projects with widespread support from the community receive adequate funding, regardless of their initial popularity or backing from large donors.

2. **Democratization**

By enabling anyone to contribute, regardless of the size of their donation, Quadratic Funding democratizes the funding process. It empowers individuals to support projects they believe in, fostering a sense of ownership and engagement within the community.

3. **Transparency**

Transparency is fundamental to Quadratic Funding. All contributions and allocations are recorded on a public ledger, ensuring accountability and trust in the allocation process. This transparency also enables stakeholders to monitor the distribution of funds and hold administrators accountable for their decisions.

# Contracts

There are two contracts, the quadratic funding contract and the NFT contract extending [`cw721-base`](https://github.com/CosmWasm/cw-nfts/tree/main/contracts/cw721-base) for the use of non-transferable NFT owners instead of whitelist addresses for the following reasons:

1. NFT ownership is recorded onchain which is transparent and immutable. On the contrary, whitelists may be stored off-chain, so NFT usage allows a more transparent and verifiable representation of ownership directly on the blockchain, ensuring greater visibility, accountability, and trust in the management.

1. In addition, NFTs used during the funding round is interoperable in contrast to the whitelists so that they could be used for different projects or platforms that support the same NFT standard.

1. NFT contract uses [`cw_ownable`](https://github.com/larry0x/cw-plus-plus/tree/main/packages/ownable) to assure only admin of the NFT contract can execute logic of the contract. NFTs are non-transferable so, the voting power associated with each NFT remains constant and is determined at the time of issuance (mint). This simplifies the voting power calculating as it does not change with time. Moreover, NFT ownership remains stable and is limited to the original recipient. This means that there is no risk of external parties acquiring NFTs solely for the purpose of influencing voting outcomes. This may enhance the integrity of the voting system.

1. The voting power is stored in the metadata field for the use purpose in the quadratic funding formula. This metadata can be further enriched with visuals and additional information which enables an enhanced user experience. NFTs offer more visibility and engagemenet in the user side in contrast to the whitelist addresses.

The quadratic funding contract logic has following functionality:

- After non-transferable NFTs (a single NFT per address) with voting power functionality are minted to the users via the NFT contract, the voting power will be used on the Quadratic Funding formula.
- The proposal owners can create proposals and only the NFT owners can vote on proposals.
- Each Proposal have an address that funds should be sent to.
- Proposal periods & voting periods are either defined in advance by contract parameters, or are explicitly triggered via function calls from contract creator or admin. If periods are triggered via function calls, minimum proposal periods and voting periods should be set upon contract instantiation.
- Voters vote on proposals by using the voting power of the NFTs they own in a function call referencing a proposal. Therefore, each proposal receive funds at the end of matching according to the quadratic funding formula from the budget but not funds from voters.
- There is a functionality to change the votes on a proposal by its id, given that the change in vote is within the total voting power of the sender.
- Once voting period ends, contract creator / admin triggers the distribution of funds to proposals according to the capital constrained liberal radicalism quadratic funding formula.

