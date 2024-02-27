
# Deploy the NFT Contract

Firstly, export the variables for easier interaction so that we do not have to type them every time.

For the Malaga-420 testnet (change these for appropriate testnet)
```
export RPC="https://rpc.malaga-420.cosmwasm.com:443"
export CHAIN_ID="malaga-420"
export FEE_DENOM="umlg"
```

Then,
```
# bash
export NODE="--node $RPC"
export TXFLAG="${NODE} --chain-id ${CHAIN_ID} --gas-prices 0.25${FEE_DENOM} --gas auto --gas-adjustment 1.3"

# zsh
export NODE=(--node $RPC)
export TXFLAG=($NODE --chain-id $CHAIN_ID --gas-prices 0.25$FEE_DENOM --gas auto --gas-adjustment 1.3)
```

To store the contract: 

```
RES=$(wasmd tx wasm store artifacts/nft.wasm --from wallet $TXFLAG -y --output json -b block)
# Otherwise, you will have to type in the following command to upload the wasm binary to the testnet:
RES=$(wasmd tx wasm store artifacts/nft.wasm --from wallet --node https://rpc.malaga-420.cosmwasm.com:443 --chain-id malaga-420 --gas-prices 0.25umlg --gas auto --gas-adjustment 1.3 -y --output json -b block)
```


Then, the following gives the code id of the deployed contract:

```
CODE_ID=$(echo $RES | jq -r '.logs[0].events[-1].attributes[0].value')
echo $CODE_ID
```

Now, the instantiate msg for the NFT contract is:

Write your wallet address to both owner and minter as only the owner of the admin (owner) can use execute functions and if the admin and owner is different, minting will throw an error due to extending cw721_base.

```
CW721_INIT='{"owner": ..address...,"minter": ..address... , "name": "My NFT Tokens", "symbol": "MyNFT"}'
```

Now, instantiate the contract with CW721_INIT instantiate msg:

```
wasmd tx wasm instantiate $CODE_ID "$CW721_INIT" --from wallet --label "non-transferable token" $TXFLAG -y --no-admin
```

We can check the contract details

```
NFT_CONTRACT=$(wasmd query wasm list-contract-by-code $CODE_ID $NODE --output json | jq -r '.contracts[-1]')
echo $NFT_CONTRACT
```

# Mint A NFT
Mint a NFT to an address (don't confuse with the admin address, this address is the address that will hold this unique nft token):

```
MINT='{"mint": { "token_id": "1", "owner": ..address.., "extension": {"name": "Voting_NFT_1", "voting_power":"5"}}}'
wasmd tx wasm execute $NFT_CONTRACT "$MINT" --from wallet $TXFLAG -y
```

# Query Functions
Query all NFT info by its unique token id:

```
NFT_INFO='{ "nft_info": { "token_id": "1" } }'
wasmd query wasm contract-state smart $NFT_CONTRACT "$NFT_INFO" $NODE --output json
```

Query Extension by token id:

```
NFT_EXTENSION='{"check_voting_power": {"token_id": "1" } }'
wasmd query wasm contract-state smart $NFT_CONTRACT "$NFT_EXTENSION" $NODE --output json
```

To see the owner address of a token by its unique token id:

```
OWNER_OF='{ "owner_of": { "token_id": "1" } }'
wasmd query wasm contract-state smart $NFT_CONTRACT "$OWNER_OF" $NODE --output json
```

Query all tokens by its owner address:
```
TOKENS='{ "tokens": { "owner": ..address..}}'
wasmd query wasm contract-state smart $NFT_CONTRACT "$TOKENS" $NODE --output json
```