**文件中所用的命令要求本文件所在目录应该处于$HOME下**

# steps
- prepare software
- create account
- start L1
- fund account on L1
- prepare contract deploy env
- prepare common env
- modiyfy contract deploy parameters
- deploy contract

## prepare software
| Download link | Version | Check version | 
| --- | --- | --- |
| [Node](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) | ^20 | `node --version` |
| npm | ^10 | `npm --version` |
| [Foundry](https://book.getfoundry.sh/getting-started/installation) | ^0.2 | `forge --version` |
| [jq](https://jqlang.github.io/jq/download/) | ^1.6 | `jq -V` |

## create account
```bash
cast wallet new-mnemonic --words 12
```

output:
```text
Successfully generated a new mnemonic.
Phrase:
forward victory coin shove worry early there scorpion differ vibrant strong pizza

Accounts:
- Account 0:
Address:     0x28fc11543495a8C65D733A72fBf2b2c1E17F3823
Private key: 0x65abee65710addfb29b2d7119af1d167eb046267ef54798adb1ad1959f1ca967
```

## start L1
[download](https://geth.ethereum.org/downloads)

## start geth dev
```bash
geth --http --http.api "admin,eth,debug,miner,net,txpool,personal,web3" --http.addr "0.0.0.0" --http.corsdomain "*" --http.vhosts "*" --ws --ws.origins "*" --ws.addr "0.0.0.0" --dev --dev.period 1 --datadir ./geth_data --rpc.allow-unprotected-txs
```

## fund account on L1
```bash
geth attach geth_data/geth.ipc
```

fund account
```bash
> eth.sendTransaction({from: eth.accounts[0], to: "0x28fc11543495a8C65D733A72fBf2b2c1E17F3823", value: web3.toWei(10, "ether")})
```

## prepare contract deploy env
```bash
cd ~/cdk-deploy/contract/cdk-validium-contracts-0.0.2
cp .env.example .env
```

modify .env
```bash
MNEMONIC="forward victory coin shove worry early there scorpion differ vibrant strong pizza"
INFURA_PROJECT_ID=""
ETHERSCAN_API_KEY=""
```
- MNEMONIC: create account步骤生成的助记词
- INFURA_PROJECT_ID: 不修改
- ETHERSCAN_API_KEY: 不修改

## prepare common env
```bash
cd ~/cdk-deploy
cp .env.example .env
```

`modify .env:`
- TEST_ADDRESS: create account步骤创建的账号地址
- TEST_PRIVATE_KEY: create account步骤创建的账号私钥

## modiyfy contract deploy parameters
```bash
source ~/cdk-deploy/.env

cd ~/cdk-deploy/contract/cdk-validium-contracts-0.0.2/deployment

jq --arg TEST_ADDRESS "$TEST_ADDRESS" --arg L2_CHAIN_ID "$L2_CHAIN_ID" '.trustedSequencerURL = "http://127.0.0.1:8123" | .trustedSequencer = $TEST_ADDRESS | .trustedAggregator = $TEST_ADDRESS | .admin = $TEST_ADDRESS | .cdkValidiumOwner = $TEST_ADDRESS | .initialCDKValidiumDeployerOwner = $TEST_ADDRESS | .timelockAddress = $TEST_ADDRESS | .forkID = 6 | .chainID = $L2_CHAIN_ID' ./deploy_parameters.json.example > ./deploy_parameters.json
```

也可以用其他方法修改参数文件deploy_parameters.json:
```json
{
    "realVerifier": false,
    "trustedSequencerURL": "http://127.0.0.1:8123,
    "networkName": "cdk-validium",
    "version":"0.0.1",
    "trustedSequencer":"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "chainID": 1001,
    "trustedAggregator":"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "trustedAggregatorTimeout": 604799,
    "pendingStateTimeout": 604799,
    "forkID": 6,
    "admin":"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "cdkValidiumOwner": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "timelockAddress": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "minDelayTimelock": 3600,
    "salt": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "initialCDKValidiumDeployerOwner" :"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "maticTokenAddress":"0x617b3a3528F9cDd6630fd3301B9c8911F7Bf063D",
    "cdkValidiumDeployerAddress":"",
    "deployerPvtKey": "",
    "maxFeePerGas":"",
    "maxPriorityFeePerGas":"",
    "multiplierGas": "",
    "setupEmptyCommittee": true,
    "committeeTimelock": false
}
```
需要关注修改的参数:
- chainID
- jq命令中的参数

## deploy contract
```bash
cd ~/cdk-deploy/contract/cdk-validium-contracts-0.0.2
npm install

cd ~/cdk-deploy/contract/cdk-validium-contracts-0.0.2/deployment
npm run deploy:deployer:CDKValidium:sepolia
npm run deploy:testnet:CDKValidium:sepolia
```

`如果需要部署到非本地geth的L1请修改~/cdk-deploy/contract/cdk-validium-contracts-0.0.2/hardhat.config.js的networks.sepolia.url`