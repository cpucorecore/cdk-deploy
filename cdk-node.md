**文件中所用的命令要求本文件所在目录处于$HOME下**

# steps
- prepare software
- prepare env
- prepare config
- prepare account keystore
- start docker compose
- approve node

## 1. prepare software
| Software | Version | Installation link |
| --- | --- | --- |
| `cast` | 0.2.0 | https://book.getfoundry.sh/getting-started/installation |
| `jq` | 1.0 | https://jqlang.github.io/jq/download/ |
| `tomlq` | 3.0.0 | https://kislyuk.github.io/yq/#installation |
| `make` | 3.80.0 | https://www.gnu.org/software/make/ |
| `docker` | 24.0.0 | https://docs.docker.com/engine/install/ |

## 2. prepare env
```bash
cd ~/cdk-deploy/contract/cdk-validium-contracts-0.0.2/deployment

echo "GEN_BLOCK_NUMBER=$(jq -r '.deploymentBlockNumber' deploy_output.json)" >> ~/cdk-deploy/.env
echo "CDK_VALIDIUM_ADDRESS=$(jq -r '.cdkValidiumAddress' deploy_output.json)" >> ~/cdk-deploy/.env
echo "POLYGON_ZKEVM_BRIDGE_ADDRESS=$(jq -r '.polygonZkEVMBridgeAddress' deploy_output.json)" >> ~/cdk-deploy/.env
echo "POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS=$(jq -r '.polygonZkEVMGlobalExitRootAddress' deploy_output.json)" >> ~/cdk-deploy/.env
echo "CDK_DATA_COMMITTEE_CONTRACT_ADDRESS=$(jq -r '.cdkDataCommitteeContract' deploy_output.json)" >> ~/cdk-deploy/.env
echo "MATIC_TOKEN_ADDRESS=$(jq -r '.maticTokenAddress' deploy_output.json)" >> ~/cdk-deploy/.env
```

## 3. prepare config
- source env
- genesis
- node
- dac
- bridge

### source env
```bash
source ~/cdk-deploy/.env
```

### genesis
```bash
cd ~/cdk-deploy/cdk-node

jq --arg L1_CHAIN_ID $L1_CHAIN_ID --argjson data "$(jq '{maticTokenAddress, cdkValidiumAddress, cdkDataCommitteeContract, polygonZkEVMGlobalExitRootAddress, deploymentBlockNumber}' ~/cdk-deploy/contract/cdk-validium-contracts-0.0.2/deployment/deploy_output.json)" \
'.L1Config.chainId = ($L1_CHAIN_ID|tonumber) | 
.L1Config.maticTokenAddress = $data.maticTokenAddress | 
.L1Config.polygonZkEVMAddress = $data.cdkValidiumAddress | 
.L1Config.cdkDataCommitteeContract = $data.cdkDataCommitteeContract | 
.L1Config.polygonZkEVMGlobalExitRootAddress = $data.polygonZkEVMGlobalExitRootAddress | 
.genesisBlockNumber = $data.deploymentBlockNumber' ~/cdk-deploy/contract/cdk-validium-contracts-0.0.2/deployment/genesis.json > ./config/genesis.json
```

### node
```bash
tomlq -i -t --arg L1_URL "$L1_URL" '.Etherman.URL = $L1_URL' ./config/node-config.toml
tomlq -i -t --arg TEST_ADDRESS "$TEST_ADDRESS" '.SequenceSender.L2Coinbase = $TEST_ADDRESS' ./config/node-config.toml
tomlq -i -t --arg TEST_ADDRESS "$TEST_ADDRESS" '.Aggregator.SenderAddress = $TEST_ADDRESS' ./config/node-config.toml
```

### dac
```bash
tomlq -i -t --arg L1_URL "$L1_URL" '.L1.RpcURL = $L1_URL' ./config/dac-config.toml
tomlq -i -t --arg L1_WS_URL "$L1_WS_URL" '.L1.WsURL = $L1_WS_URL' ./config/dac-config.toml
tomlq -i -t --arg CDK_VALIDIUM_ADDRESS "$CDK_VALIDIUM_ADDRESS" '.L1.CDKValidiumAddress = $CDK_VALIDIUM_ADDRESS' ./config/dac-config.toml
tomlq -i -t --arg CDK_DATA_COMMITTEE_CONTRACT_ADDRESS "$CDK_DATA_COMMITTEE_CONTRACT_ADDRESS" '.L1.DataCommitteeAddress = $CDK_DATA_COMMITTEE_CONTRACT_ADDRESS' ./config/dac-config.toml
```

`请注意参数--rpc-url,根据实际的L1设置`
```bash
cast send --legacy --from $TEST_ADDRESS --private-key $TEST_PRIVATE_KEY --rpc-url http://localhost:8545 $CDK_DATA_COMMITTEE_CONTRACT_ADDRESS 'function setupCommittee(uint256 _requiredAmountOfSignatures, string[] urls, bytes addrsBytes) returns()' 1 '["http://zkevm-data-availability:8444"]' $TEST_ADDRESS
```

### bridge
```bash
tomlq -i -t --arg L1_URL "$L1_URL" '.Etherman.L1URL = $L1_URL' ./config/bridge-config.toml
tomlq -i -t --arg GEN_BLOCK_NUMBER "$GEN_BLOCK_NUMBER" '.NetworkConfig.GenBlockNumber = $GEN_BLOCK_NUMBER' ./config/bridge-config.toml
tomlq -i -t --arg CDK_VALIDIUM_ADDRESS "$CDK_VALIDIUM_ADDRESS" '.NetworkConfig.PolygonZkEVMAddress = $CDK_VALIDIUM_ADDRESS' ./config/bridge-config.toml
tomlq -i -t --arg POLYGON_ZKEVM_BRIDGE_ADDRESS "$POLYGON_ZKEVM_BRIDGE_ADDRESS" '.NetworkConfig.PolygonBridgeAddress = $POLYGON_ZKEVM_BRIDGE_ADDRESS' ./config/bridge-config.toml
tomlq -i -t --arg POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS "$POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS" '.NetworkConfig.PolygonZkEVMGlobalExitRootAddress = $POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS' ./config/bridge-config.toml
tomlq -i -t --arg MATIC_TOKEN_ADDRESS "$MATIC_TOKEN_ADDRESS" '.NetworkConfig.MaticTokenAddress = $MATIC_TOKEN_ADDRESS' ./config/bridge-config.toml
tomlq -i -t --arg POLYGON_ZKEVM_BRIDGE_ADDRESS "$POLYGON_ZKEVM_BRIDGE_ADDRESS" '.NetworkConfig.L2PolygonBridgeAddresses = [$POLYGON_ZKEVM_BRIDGE_ADDRESS]' ./config/bridge-config.toml
```

### 4. prepare account keystore
```bash
docker run -v ./config:/app/account cpucorecore/cdk-env /app/zkevm-node encryptKey --pk=$TEST_PRIVATE_KEY --pw="testonly" --output=/app/account/keystore

find ./config/keystore -type f -name 'UTC--*' | head -n 1 | xargs -I xxx mv xxx ./config/account.key

# keystore生成完毕后该容器可删除
```

## 5. start docker compose
```bash
make run
```

## 6. approve node
```bash
docker run -v ./config/genesis.json:/app/genesis.json -v ./config/node-config.toml:/app/node-config.toml -v ./config/account.key:/app/account.key cpucorecore/cdk-env /app/zkevm-node approve --network custom --custom-network-file /app/genesis.json --cfg /app/node-config.toml --amount 1000000000000000000000000000 --password "testonly" --yes --key-store-path /app/account.key
```