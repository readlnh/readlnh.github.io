---
title: Darwinia Eth-Backing
date: 2019-12-20 18:37
tags: 
    - Rust
    - Substrate
    - blcokchain
    - darwinia
---

这几天在写Darwinia的eth-backing模块的测试，遇到不少问题，写个记录。

## 概述
eth-backing模块的主要负责以太坊的跨链功能，darwinia跨链的思路大致如下：
我在以太坊上建立相应的智能合约，以太坊上的token和darwinia上的token应该是可以1:1兑换的。一笔钱当然不能花两次，所以以太坊上的钱转移到darwinia上时，以太坊上的token就应该被销毁。所以总体逻辑应该是:

- 1. 以太坊上销毁这笔token
- 2. 将销毁的证据proof发给darwinia
- 3. darwinia解析proof并在darwinia网络上生成token

这个过程在darwinia上被称为redeem。

### redeem
实际上，在darwinia上有两种原生资产：

- ring
- kton

所以理论上redeem应该有这两种对象。但是由于darwinia中锁定ring可以得到kton，所以ring有两种状态，一种是锁定中的，一种是普通的。如此一来redeem的对象又多了一个，对于锁定中的ring，我们可以认为它是被存在银行里，所以有一个存单deposit。故，redeem的对象如下：

- ring
- kton
- deposit

### storage
链上存储的数据大致可以分为三类：

- Redeem地址，即proof提交的地址，测试时需要预设
    - RingRedeemAddress get(ring_redeem_address) config(): EthAddress;
    - KtonRedeemAddress get(kton_redeem_address) config(): EthAddress;
    - DepositRedeemAddress get(deposit_redeem_address) config(): EthAddress;

- 链上初始抵押的ring和kton，每次redemm后就减掉
    - RingLocked get(fn ring_locked) config(): RingBalanceOf<T>;
    - KtonLocked get(fn kton_locked) config(): KtonBalanceOf<T>;

- ProofVerifed，用来存储已经redeem的proof，保证一个proof不会redeem两次
    - RingProofVerified get(ring_proof_verfied): map (H256, u64) => Option<EthReceiptProof>;
    - KtonProofVerified get(kton_proof_verfied): map (H256, u64) => Option<EthReceiptProof>;
    - DepositProofVerified 

## 实现
前两者的实现其实非常简单，就是检查proof，解析proof，token转账，保存proof信息等等。比较特殊的是deposit的redeem。因为deposit里面的ring是locked的ring，所以需要先将ring转入账户中然后再锁定起来，但是要注意的是这时候锁定是没有kton的，因为在之前就已经奖励过了。这里似乎用到了support回调？不知道具体为何可以这么做。

## 测试
测试部分由于不熟悉substrate的测试，所以浪费了很多时间。

### mock
首先是mock，主要就是引入模块，以及初始化，初始化构造如下

```rust
impl ExtBuilder {
	pub fn build(self) -> runtime_io::TestExternalities {
		let mut t = system::GenesisConfig::default().build_storage::<Test>().unwrap();

		let _ = GenesisConfig::<Test> {
			ring_redeem_address: hex!["dbc888d701167cbfb86486c516aafbefc3a4de6e"].into(),
			kton_redeem_address: hex!["dbc888d701167cbfb86486c516aafbefc3a4de6e"].into(),
			deposit_redeem_address: hex!["6ef538314829efa8386fc43386cb13b4e0a67d1e"].into(),
			ring_locked: 20000000000000,
			kton_locked: 5000000000000,
		}
		.assimilate_storage(&mut t)
		.unwrap();

		t.into()
	}
}
```


### tests
测试主要就是对比信息，这里没有太多的难点，主要就是需要调用一些辅助的api来帮助找出原始数据。另外就是关于调用其他模块的private函数的问题。

#### Some apis(tools) for darwinia_eth_backing development 

- ropsten
	https://ropsten.etherscan.io/tx/0x59c6758bd2b93b2f060e471df8d6f4d901c453d2c2c012ba28088acfb94f821

- api for proof
	https://alpha.evolution.land/api/darwinia/receipt?tx=0x59c6758bd2b93b2f060e471df8d6f4d901c453d2c2c012ba28088acfb94f8216

	- tx = Transaction Hash

	Just change Transaction Hash
    - index 
    - proof
    - header_hash

- api for all information
http://api-ropsten.etherscan.io/api?module=proxy&action=eth_getBlockByNumber&tag=0x6a910b&boolean=true&apikey=YourApiKeyToken

	Just replace the blocknumber(hex)

#### Calling Private Dispatchable Functions
具体可参见[Extending Substrate Runtime Modules](https://www.shawntabrizi.com/substrate/extending-substrate-runtime-modules/#calling-private-dispatchable-functions)

```rust
assert_ok!(
	staking::Call::<Test>::bond(
		controller.clone(),
		StakingBalances::Ring(1),
		RewardDestination::Controller,
		0).dispatch(Origin::signed(expect_account_id.clone())
	)
); 	
```

这里需要调用`bond`函数，因为只有参与staking的token才可以lock，而参与staking需要controller。



