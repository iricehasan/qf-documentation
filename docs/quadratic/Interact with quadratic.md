
# Deploy the Quadratic Funding Contract

To store the contract: 

```
RES=$(wasmd tx wasm store artifacts/nft.wasm --from wallet $TXFLAG -y --output json -b block)
# Otherwise, you will have to type in the following command to upload the wasm binary to the testnet:
RES=$(wasmd tx wasm store artifacts/nft.wasm --from wallet --node https://rpc.malaga-420.cosmwasm.com:443 --chain-id malaga-420 --gas-prices 0.25umlg --gas auto --gas-adjustment 1.3 -y --output json -b block)
```


Then, the following gives the code id of the deployed contract

```
QF_CODE_ID=$(echo $RES | jq -r '.logs[0].events[-1].attributes[0].value')
echo $QF_CODE_ID
```


Now, the instantiate msg for the quadratic funding contract is:

Write your wallet address to with the same contract address as the nft contract admin. Note that the amount provided in instantiation is the budget of the quadratic funding round, and the nft_address is the nft contract address that is deployed.

```
INIT='{"admin": ..address..., "nft_address": ..address.. "leftover_addr": ..address.., "voting_period": { at_height: "257600" }}, "proposal_period": { at_height: "257600" }, "budget_denom": "ucosm", "algorithm": { capital_constrained_liberal_radicalism: {parameter: "param"}} '
```

 

```
wasmd tx wasm instantiate $QF_CODE_ID "$INIT" --amount 1000000ucosm --from wallet --label "quadratic funding" $TXFLAG -y --no-admin
```


We can check the contract details

```
CONTRACT=$(wasmd query wasm list-contract-by-code $QF_CODE_ID $NODE --output json | jq -r '.contracts[-1]')
echo $CONTRACT
```
# Create Proposals

Creating proposals with a title, description and metadata when the height is smaller than the proposal_period given in instantiation. The owner address should be the address that executes this transaction so only the proposal owner can create proposals. fund_address is the address that funds will be sent to when the distribution function is triggered (after the proposal and voting period ended).

```
CreateProposal {
    title: String,
    owner: String,
    description: String,
    metadata: Option<Binary>,
    fund_address: String,
}
```

```
CREATE_PROPOSAL='{"title": "title1", "owner": ..address.., "description": "desc", "fund_addresss": "cosmos146sch227m6erjytl4ax5fl78gh4fw03qmaehfh"}'
wasmd tx wasm execute $CONTRACT "$CREATE_PROPOSAL" --from wallet $TXFLAG -y
```


# Vote Proposals

Each non-transferable token holder can vote within their voting-power during the voting_period (given in instantiation where height is smaller than the voting period height) to a poroposal with proposal_id and the desired sent_vote. They can distribute their votes to more than one proposal within their total voting power.
It is assumed that there is only one non-transferable token per one address. So, if an address has more than one non-transferable token minted to their address, they can only use the first one to vote.

```
VoteProposal {
    proposal_id: u64,
    sent_vote: u128,
}
```

Assuming this wallet address has a token with voting power more than 5:

```
VOTE_PROPOSAL='{"proposal_id": "1", "sent_vote":"5"}'
wasmd tx wasm execute $CONTRACT "$VOTE_PROPOSAL" --from wallet $TXFLAG -y
```


# Change Vote

A non-transferable token holder can change their vote on already voted proposals by providing a proposal_id and the new desired vote as sent_vote during the voting_period within their total voting power.

```
ChangeVote {
    proposal_id: u64,
    sent_vote: u128,
}
    
```

Assuming this wallet address has a token with voting power more than 10, their vote will be updated to 10 for this proposal.

```
CHANGE_VOTE='{"proposal_id": "1", "sent_vote":"10"}'
wasmd tx wasm execute $CONTRACT "$CHANGE_VOTE" --from wallet $TXFLAG -y
```

# Distribution of Funds

After the proposal and voting periods end, the admin of the quadratic funding contract (which should also be the minter of the nft contract) can trigger the distribution function to distribute budget to proposals according to the algorithm given in the instantiation.

```
TriggerDistribution {}
```

```
TRIGGER='{"trigger_distribution": {}}'
wasmd tx wasm execute $CONTRACT "$TRIGGER" --from wallet $TXFLAG -y
```

# Query Functions

There are two query functions in the quadratic funding contract. One can query a single proposal by its proposal id or query all of the proposals.
```    
ProposalByID { id: u64 },
AllProposals {},
```

```
QUERY_PROPOSAL='{"proposal_by_i_d": {"id": "1"}}'
wasmd query wasm contract-state smart $CONTRACT "$QUERY_PROPOSAL" $NODE --output json

QUERY_ALL='{"all_proposals": {}}'
wasmd query wasm contract-state smart $CONTRACT "$QUERY_ALL" $NODE --output json
```