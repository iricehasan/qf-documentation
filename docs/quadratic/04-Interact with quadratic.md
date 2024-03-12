
# Deploy the Quadratic Funding Contract

To store the contract: 

```
RES=$(wasmd tx wasm store artifacts/cw_quadratic_funding.wasm --from wallet $TXFLAG -y --output json -b block)
# Otherwise, you will have to type in the following command to upload the wasm binary to the testnet:
RES=$(wasmd tx wasm store artifacts/nft.wasm --from wallet --node https://rpc-palvus.pion-1.ntrn.tech:443 --chain-id pion-1 --gas-prices 0.025untrn --gas auto --gas-adjustment 1.3 -y --output json -b block)
```


Then, the following gives the code id of the deployed contract

```
QF_CODE_ID=$(echo $RES | jq -r '.logs[0].events[-1].attributes[0].value')
echo $QF_CODE_ID
```

QF_CODE_ID=3335


Now, the instantiate msg for the quadratic funding contract is:

Write your wallet address to with the same contract address as the nft contract admin. Note that the amount provided in instantiation is the budget of the quadratic funding round, and the nft_address is the nft contract address that is deployed.

```
INIT='{"admin": ..address..., "nft_address": ..address.. "leftover_addr": ..address.., "voting_period": { at_height: "257600" }}, "proposal_period": { at_height: "257600" }, "budget_denom": "ucosm", "algorithm": { capital_constrained_liberal_radicalism: {parameter: "param"}} '
```

 

```
wasmd tx wasm instantiate $QF_CODE_ID "$INIT" --amount 1000000untrn --from wallet --label "quadratic funding" $TXFLAG -y --no-admin
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
CREATE_PROPOSAL='{"title": "title1", "owner": ..address.., "description": "desc", "fund_addresss": ..address..}'
wasmd tx wasm execute $CONTRACT "$CREATE_PROPOSAL" --from wallet $TXFLAG -y
```


# Vote Proposals

Each non-transferable token holder can vote within their voting-power during the voting_period (given in instantiation where height is smaller than the voting period height) to a poroposal with proposal_id and the desired sent_vote. They can distribute their votes to more than one proposal within their total voting power.
It is assumed that there is only one non-transferable token per one address. So, if an address has more than one non-transferable token minted to their address, they can only use the first one to vote.

```
VoteProposal {
    proposal_id: u64,
    sent_vote: Uint128,
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
    sent_vote: Uint128,
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

# CosmJS Actions

To create typescript types and the client, we can use [ts-codegen](https://github.com/CosmWasm/ts-codegen) by [Cosmology](https://cosmology.zone/).

Under the direction src/cw-quadratic-funding, run

```
cosmwasm-ts-codegen generate \
    --plugin client
    --schema ./schema \
    --out ./ts \
    --name Qf
```

This will generate two files under ts directory.

```typescript title = "Qf.types.ts"
/**
* This file was automatically generated by @cosmwasm/ts-codegen@0.35.7.
* DO NOT MODIFY IT BY HAND. Instead, modify the source JSONSchema file,
* and run the @cosmwasm/ts-codegen generate command to regenerate this file.
*/

export type QuadraticFundingAlgorithm = {
  capital_constrained_liberal_radicalism: {
    parameter: string;
  };
};
export type Expiration = {
  at_height: number;
} | {
  at_time: Timestamp;
} | {
  never: {
    [k: string]: unknown;
  };
};
export type Timestamp = Uint64;
export type Uint64 = string;
export interface InstantiateMsg {
  admin: string;
  algorithm: QuadraticFundingAlgorithm;
  budget_denom: string;
  leftover_addr: string;
  nft_address: string;
  proposal_period: Expiration;
  voting_period: Expiration;
}
export type ExecuteMsg = {
  create_proposal: {
    description: string;
    fund_address: string;
    metadata?: Binary | null;
    owner: string;
    title: string;
  };
} | {
  vote_proposal: {
    proposal_id: number;
    sent_vote: Uint128;
  };
} | {
  change_vote: {
    proposal_id: number;
    sent_vote: Uint128;
  };
} | {
  trigger_distribution: {};
};
export type Binary = string;
export type Uint128 = string;
export type QueryMsg = {
  proposal_by_i_d: {
    id: number;
  };
} | {
  all_proposals: {};
};
export interface AllProposalsResponse {
  proposals: Proposal[];
}
export interface Proposal {
  collected_votes: Uint128;
  description: string;
  fund_address: string;
  id: number;
  metadata?: Binary | null;
  owner: string;
  title: string;
}
```


```typescript title = "Qf.client.ts"
/**
* This file was automatically generated by @cosmwasm/ts-codegen@0.35.7.
* DO NOT MODIFY IT BY HAND. Instead, modify the source JSONSchema file,
* and run the @cosmwasm/ts-codegen generate command to regenerate this file.
*/

import { CosmWasmClient, SigningCosmWasmClient, ExecuteResult } from "@cosmjs/cosmwasm-stargate";
import { Coin, StdFee } from "@cosmjs/amino";
import { QuadraticFundingAlgorithm, Expiration, Timestamp, Uint64, InstantiateMsg, ExecuteMsg, Binary, Uint128, QueryMsg, AllProposalsResponse, Proposal } from "./Qf.types";
export interface QfReadOnlyInterface {
  contractAddress: string;
  proposalByID: ({
    id
  }: {
    id: number;
  }) => Promise<Proposal>;
  allProposals: () => Promise<AllProposalsResponse>;
}
export class QfQueryClient implements QfReadOnlyInterface {
  client: CosmWasmClient;
  contractAddress: string;

  constructor(client: CosmWasmClient, contractAddress: string) {
    this.client = client;
    this.contractAddress = contractAddress;
    this.proposalByID = this.proposalByID.bind(this);
    this.allProposals = this.allProposals.bind(this);
  }

  proposalByID = async ({
    id
  }: {
    id: number;
  }): Promise<Proposal> => {
    return this.client.queryContractSmart(this.contractAddress, {
      proposal_by_i_d: {
        id
      }
    });
  };
  allProposals = async (): Promise<AllProposalsResponse> => {
    return this.client.queryContractSmart(this.contractAddress, {
      all_proposals: {}
    });
  };
}
export interface QfInterface extends QfReadOnlyInterface {
  contractAddress: string;
  sender: string;
  createProposal: ({
    description,
    fundAddress,
    metadata,
    owner,
    title
  }: {
    description: string;
    fundAddress: string;
    metadata?: Binary;
    owner: string;
    title: string;
  }, fee?: number | StdFee | "auto", memo?: string, _funds?: Coin[]) => Promise<ExecuteResult>;
  voteProposal: ({
    proposalId,
    sentVote
  }: {
    proposalId: number;
    sentVote: Uint128;
  }, fee?: number | StdFee | "auto", memo?: string, _funds?: Coin[]) => Promise<ExecuteResult>;
  changeVote: ({
    proposalId,
    sentVote
  }: {
    proposalId: number;
    sentVote: Uint128;
  }, fee?: number | StdFee | "auto", memo?: string, _funds?: Coin[]) => Promise<ExecuteResult>;
  triggerDistribution: (fee?: number | StdFee | "auto", memo?: string, _funds?: Coin[]) => Promise<ExecuteResult>;
}
export class QfClient extends QfQueryClient implements QfInterface {
  client: SigningCosmWasmClient;
  sender: string;
  contractAddress: string;

  constructor(client: SigningCosmWasmClient, sender: string, contractAddress: string) {
    super(client, contractAddress);
    this.client = client;
    this.sender = sender;
    this.contractAddress = contractAddress;
    this.createProposal = this.createProposal.bind(this);
    this.voteProposal = this.voteProposal.bind(this);
    this.changeVote = this.changeVote.bind(this);
    this.triggerDistribution = this.triggerDistribution.bind(this);
  }

  createProposal = async ({
    description,
    fundAddress,
    metadata,
    owner,
    title
  }: {
    description: string;
    fundAddress: string;
    metadata?: Binary;
    owner: string;
    title: string;
  }, fee: number | StdFee | "auto" = "auto", memo?: string, _funds?: Coin[]): Promise<ExecuteResult> => {
    return await this.client.execute(this.sender, this.contractAddress, {
      create_proposal: {
        description,
        fund_address: fundAddress,
        metadata,
        owner,
        title
      }
    }, fee, memo, _funds);
  };
  voteProposal = async ({
    proposalId,
    sentVote
  }: {
    proposalId: number;
    sentVote: Uint128;
  }, fee: number | StdFee | "auto" = "auto", memo?: string, _funds?: Coin[]): Promise<ExecuteResult> => {
    return await this.client.execute(this.sender, this.contractAddress, {
      vote_proposal: {
        proposal_id: proposalId,
        sent_vote: sentVote
      }
    }, fee, memo, _funds);
  };
  changeVote = async ({
    proposalId,
    sentVote
  }: {
    proposalId: number;
    sentVote: Uint128;
  }, fee: number | StdFee | "auto" = "auto", memo?: string, _funds?: Coin[]): Promise<ExecuteResult> => {
    return await this.client.execute(this.sender, this.contractAddress, {
      change_vote: {
        proposal_id: proposalId,
        sent_vote: sentVote
      }
    }, fee, memo, _funds);
  };
  triggerDistribution = async (fee: number | StdFee | "auto" = "auto", memo?: string, _funds?: Coin[]): Promise<ExecuteResult> => {
    return await this.client.execute(this.sender, this.contractAddress, {
      trigger_distribution: {}
    }, fee, memo, _funds);
  };
}
```

Then, start the CosmJS CLI using

```
npx @cosmjs/cli@^0.32.3 
```

Then,
```
import { QfClient } from "./Qf.client"; // Replace with the actual path to the file
import { SigningCosmWasmClient } from "@cosmjs/cosmwasm-stargate";
import { DirectSecp256k1HdWallet } from "@cosmjs/proto-signing";
import { GasPrice } from "@cosmjs/stargate";
import fs from "fs";


// Define private key 
// const privateKey = "your_private_key_here";

// Create a signer object using the private key
// const wallet = await DirectSecp256k1Wallet.fromKey(privateKey);

// Define a mnemonic
// const mnemonic = "your_mnemonic_here";

// Create a signer object using the mnemonic
const wallet = await DirectSecp256k1HdWallet.fromMnemonic(mnemonic, {
    prefix: "neutron",
});

// Initialize a CosmWasm client with the signer
const client = await SigningCosmWasmClient.connectWithSigner("https://rpc-palvus.pion-1.ntrn.tech", wallet, {
    gasPrice: GasPrice.fromString("0.025untrn"),
});

// Define the sender's address and the contract address
const [account] = await wallet.getAccounts();
const senderAddress = account.address;
console.log(senderAddress)

// deploy
const wasm = fs.readFileSync("/artifacts/cw_quadratic_funding.wasm") // write the path for the nft.wasm
const result = await client.upload(senderAddress, wasm, "auto")
console.log(result)

// instantiate
// const codeId = result.codeId;
const codeId = 3335;

const QuadraticFundingAlgorithm = {
    capital_constrained_liberal_radicalism: {
      parameter: "param"
}};

// the NFT contract we instantiated previously
const nftContractAddress = "neutron1e7yppujrshzzsqfflu09udrje0zpd6jnfwe304wdexl7dd28gqxqv8x776";

// get the current block height
const height = await client.getHeight();
console.log(height);

//Define the instantiate message
const instantiateMsg = {
    admin: senderAddress,
    algorithm: QuadraticFundingAlgorithm,
    budget_denom: "untrn", 
    leftover_addr: senderAddress,
    nft_address: nftContractAddress,
    proposal_period: {at_height: height + 1000},
    voting_period: {at_height: height + 1000},
  };

//Instantiate the contract with 1 NTRN
const instantiateResponse = await client.instantiate(senderAddress, codeId, instantiateMsg, "Quadratic Funding", "auto", {funds: [{amount: "1000000", denom: "untrn"}]})
console.log(instantiateResponse)

//const contractAddress = instantiateResponse.contractAddress;
const contractAddress = "neutron1kqsfsj6vszv0muxr0w62nwkk2g09m59hnhpu3p07mvsrptk62evque396x";

// Create an instance of QfClient
const qfClient = new QfClient(client, senderAddress, contractAddress);

// Example: Create a proposal
const proposalCreationResult = await qfClient.createProposal({
  description: "Proposal description",
  fundAddress: senderAddress,
  owner: senderAddress,
  title: "Proposal title",
});

console.log("Proposal creation result:", proposalCreationResult);

const voteProposalResult = await qfClient.voteProposal({
    proposalId: 1,
    sentVote: "1",
});

console.log("Vote result:", voteProposalResult);

const queryProposalbeforeResult = await client.queryContractSmart(contractAddress, {proposal_by_i_d: { id: 1}});

console.log("Query Result before Vote Change:", queryProposalbeforeResult);

const changeVoteResult = await qfClient.changeVote({
    proposalId: 1,
    sentVote: "5",
})

const queryProposalafterResult = await client.queryContractSmart(contractAddress, {proposal_by_i_d: { id: 1}});

console.log("Query Result after Vote Change:", queryProposalafterResult);

const queryAllProposalsResult = await client.queryContractSmart(contractAddress, {all_proposals: {}});
console.log("Query Result All Proposals:", queryAllProposalsResult)

const triggerDistributionResult = await qfClient.triggerDistribution();
console.log("Trigger Distribution Result:", triggerDistributionResult);

```

which gives

```
Trigger Distribution Result: {
  logs: [ { msg_index: 0, log: '', events: [Array] } ],
  height: 12302012,
  transactionHash: 'C7DA19EA60B942A700108319EE2234FA27730691D2BC7599E42CDCFECA44B401',
  events: [
    { type: 'coin_spent', attributes: [Array] },
    { type: 'coin_received', attributes: [Array] },
    { type: 'transfer', attributes: [Array] },
    { type: 'message', attributes: [Array] },
    { type: 'tx', attributes: [Array] },
    { type: 'tx', attributes: [Array] },
    { type: 'tx', attributes: [Array] },
    { type: 'message', attributes: [Array] },
    { type: 'execute', attributes: [Array] },
    { type: 'wasm', attributes: [Array] },
    { type: 'coin_spent', attributes: [Array] },
    { type: 'coin_received', attributes: [Array] },
    { type: 'transfer', attributes: [Array] },
    { type: 'coin_spent', attributes: [Array] },
    { type: 'coin_received', attributes: [Array] },
    { type: 'transfer', attributes: [Array] }
  ],
  gasWanted: 276928n,
  gasUsed: 224969n
}
undefined
```