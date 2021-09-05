# 同步bsc私链测试

### 一、搭建bsc私链

1.先下载bsc创始块合约库编译生成用于私链的配置文件genesis.json

 https://github.com/binance-chain/bsc-genesis-contract/archive/refs/tags/v1.0.3.zip

主要配置init_holder.js和validators.js分别添加默认账户和出块验证者列表。

init_holder.js

```javascript
const init_holders = [
  {
    // private key is 0x9b28f36fbd67381120752d6172ecdcf10e06ab2d9a1367aac00cdcd6ac7855d3, only use in dev
    address: "0x9fB29AAc15b9A4B7F17c3385939b007540f4d791",
    balance: web3.utils.toBN("10000000000000000000000").toString("hex")
  }
];
```

validators.js

```javascript

// Configure
const validators = [
  {
    consensusAddr: "0x0b25F687Df22EB7dc55e6d547749b229811C3dAe",
    feeAddr: "0x0b25F687Df22EB7dc55e6d547749b229811C3dAe",
    bscFeeAddr: "0x0b25F687Df22EB7dc55e6d547749b229811C3dAe",
    votingPower: 0x0000000000000064
  },
  {
    consensusAddr: "0x7e83150c703F75547D0Bf5c08E8209Da0Ae4a40C",
    feeAddr: "0x7e83150c703F75547D0Bf5c08E8209Da0Ae4a40C",
    bscFeeAddr: "0x7e83150c703F75547D0Bf5c08E8209Da0Ae4a40C",
    votingPower: 0x0000000000000064
  },
  {
    consensusAddr: "0x4193803810050169F19f2eb45793399AE443275c",
    feeAddr: "0x4193803810050169F19f2eb45793399AE443275c",
    bscFeeAddr: "0x4193803810050169F19f2eb45793399AE443275c",
    votingPower: 0x0000000000000064
  }
];

```

2.下载[bsc1.1.1](https://github.com/binance-chain/bsc/releases/tag/v1.1.1)源码编译 https://github.com/binance-chain/bsc/archive/refs/tags/v1.1.1.zip

编译geth模块源码生成2个以上的私链节点，分别用以上生成的genesis.json构建创世快，配置大概如下

```
# 启动节点0, 关闭自动发现（防止不希望的节点进入）
geth --datadir node0 init genesis.json
geth --datadir node0 --port 30000 --nodiscover --unlock '0' --password ./node0/password console
# 不加console的话，可以通过geth attach ipc:node0/geth.ipc来访问

# 启动节点1，另起一个终端，通过不同端口模拟
geth --datadir node1 init genesis.json
geth --datadir node1 --port 30001 --nodiscover --unlock '0' --password ./node1/password  console

# 启动节点2，genesis.json里已经指定networkID,启动时无需指定了
geth --datadir node2 init genesis.json
geth --datadir node2 --port 30002 --nodiscover --unlock '0' --password ./node2/password  console
```

详细配置可参考https://github.com/evsward/blog/blob/master/posts/%E4%BB%A5%E5%A4%AA%E5%9D%8A-%E7%A7%81%E6%9C%89%E9%93%BE%E6%90%AD%E5%BB%BA%E5%88%9D%E6%AD%A5%E5%AE%9E%E8%B7%B5.md

### 二、编译启动erigon-bsc同步bsc私链数据

1.下载erigon-bsc

git clone https://github.com/somewheel/erigon-bsc.git

2.编译erigon

go build ./cmd/erigon

3.启动

目前创始块的networkId及其他配置信息硬编码在代码里，暂时通过以下启动参数运行可同步bsc数据区块。

硬编码代码分别对应在

/params/config.go 328行

```go
ParliaChainConfig = &ChainConfig{
		ChainID:             big.NewInt(97),
		HomesteadBlock:      big.NewInt(0),
		DAOForkBlock:        nil,
		DAOForkSupport:      false,
		EIP150Block:         big.NewInt(0),
		EIP150Hash:          common.Hash{},
		EIP155Block:         big.NewInt(0),
		EIP158Block:         big.NewInt(0),
		ByzantiumBlock:      big.NewInt(0),
		ConstantinopleBlock: big.NewInt(0),
		PetersburgBlock:     big.NewInt(0),
		IstanbulBlock:       big.NewInt(0),
		MuirGlacierBlock:    big.NewInt(0),
		BerlinBlock:         big.NewInt(0),
		LondonBlock:         nil,
		Ethash:              nil,
		Clique:              nil,
		Parlia:              &ParliaConfig{Period: 3, Epoch: 200},
	}
```

/core/genesis.go 645行。 <u>**注意需要把ExtraData(同genesis.json的extraData字段)替换为私链的验证者们，此字段由第一步构建而成**</u>

```go
func DefaultParliaGenesisBlock() *Genesis {
	return &Genesis{
		Config:     params.ParliaChainConfig,
		Timestamp:  0x5e9da7ce,
		ExtraData:  hexutil.MustDecode("0x00000000000000000000000000000000000000000000000000000000000000000b25f687df22eb7dc55e6d547749b229811c3dae4193803810050169f19f2eb45793399ae443275c7e83150c703f75547d0bf5c08e8209da0ae4a40c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"),
		GasLimit:   0x2625a00,
		Difficulty: big.NewInt(1),
		Coinbase:   common.HexToAddress("0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE"),
		Alloc:      readPrealloc("allocs/parlia.json"),
	}
}
```

/core/allocs/parlia.json文件存储的为创世块的系统合约代码和默认账户，可根据需求调整。



./erigon --datadir ./dev --chain parlia --nodiscover --staticpeers enode://d43a9bb1481d00cdbe3f103777c2968675419eee9d5cb4e9df434c0d620b28ea09459a2b4b18b660d4ad01f825c20d71e2f0b4f56107e4b356d5bdbb7c175213@127.0.0.1:30001?discport=0

--staticpeers 为上面构建的bsc任意私链节点地址，可通过admin.nodeinfo进行查询。

根目录下的genesis.json文件可作对比参考。









