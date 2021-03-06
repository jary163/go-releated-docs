# POA(Clique)
POA共识算法详解

# 简介
POA是一个基于许可的共识算法，只有经过认证的地址(signer)才能进行挖矿生成块（没有块奖励）；认证的地址可以通过协议进行
动态增减

# 共识引擎接口
1. Author: 返回生成当前块的地址，默认会使用当前的第一个账号
2. VerifyHeader:验证块头部
3. VerifyHeaders:验证多个块头部
4. VerifyUncles:验证uncle块
5. VerifySeal:验证seal是否符合共识算法
6. Prepare:对block的头部进行初始化
6. Finalize:生成最后的块，同时对state进行修改，有块奖励的话，是在此阶段给的。
7. Seal:生成一个块
8. CalcDifficulty:计算块的难度，在poa中有特殊的意义
9. APIs:共识算法支持的api接口

## 矿工挖矿流程
1. 调用miner.start()开始挖矿
2. 为txpool设置gasprice
3. 获取etherbase，如果没有，则返回错误，如果为Clique共识算法，还需要在keystore中找寻etherbase地址，并将其设置为Clique的signer
4. 开启线程进行挖矿
5. 如果当前节点未同步完，则需等待同步完成后开始挖矿
6. worker.start()进入挖矿主循环，等待channel消息
    1. 收到work消息，调用共识引擎Seal生成新的块
    2. 收到quit消息，退出循环
7. 提交新的work，commitNewWork
    1. 设置新的块的头部参数，计算gasLimit
    2. 调用共识算法的Prepare()方法，对头部参数进行初始化
    3. 获取txpool中的pending交易，并对其按照price和nonce进行排序
    4. 将排序后的交易提交
        1. 依次对交易进行处理（参数校验、签名验证、nonce判断等）
        2. 对state进行修改，如果交易发送错误，会回滚操作，生成receipt
    5. 计算可能的uncle块
    6. 调用共识算法Finalize生成新的块
    7. 提交该工作，通过channel将该工作提交给6中主循环。
    
    
## gasLimit计算
根据父块的使用的gasUsed计算出当前块的gasLimit，这是一个矿工的策略，并不是共识的策略。

1. 计算出应该设置的gasLimit（如果父块使用的gas大于父块gasLimit*2/3，则增加gasLimit，否则减少gas或者不变）
    1. gasLimit=parent.gasLimit + (parent.gasUsed * 1.5 - parent.gasLimit） / 1024 + 1
2. limit不能小于最小值5000
3. 如果设置了TargetGasLimit，并且limit小于TargetGasLimit时，gasLimit = parent.gasLimit + parent.gasLimit/1024 - 1（不能大于目标值）

# 参数介绍
1. EPOCH_LENGTH: 经过多少个块后，设置检查点；检查点之前的数据无法修改
2. BLOCK_PERIOD: 平均出块时间
3. EXTRA_VANITY： 在extra-data字段的头部数据中，miner设置的自定义数据 
4. EXTRA_SEAL： 在extra-data字段的尾部数据中，signer签名数据
5. NONCE_AUTH： 添加一个新的signer时，将nonce设置为"0xffffffffffffffff"
6. NONCE_DROP: 删除一个signer时，将nonce设置为"0x0000000000000000"
7. UNCLE_HASH: uncle hash，值为Keccak256(RLP([]))，因为在poa中，uncle没有任何意义
8. DIFF_NOTURN：使用block中的difficulty字段，1代表outturn：代表当前块按照顺序应该由另外一个人来进行签名。
9. DIFF_INTURN：使用block中的difficulty字段，2代表inturn：表示按照顺序，该块应该由当前的signer进行签名。
10. SIGNER_COUNT：在一个特定的时间点，链中合法的signer个数
11. SIGNER_INDEX：合法signer的索引
12. SIGNER_LIMIT：同一signer生成两个块之间最少的块间隔数量。如共5个signer，该数字为3，即每个地址在签名后，到下一次签名之间，必须由
其他signer签署3个块

# header结构
1. difficulty：块难度（链会自动切换到难度最长的一条链）
    * 1代表out-turn，当前块本应由另外一个signer签名
    * 2代表in-turn，当前块应有该signer进行签名
2. extraData：额外数据，含signer的签名数据
    * 在创世块及checkpoint时，前32及后65个字节为任意数据，中间为singer地址
    * 普通块时，前32个字节为miner自定义内容，后65个字节为singer的签名，中间的数据未做处理
3. gasLimit：当前块的gas限制，同pow
4. gasUsed：使用的gas数量，同pow
5. hash：该块hash，同pow
6. logsBloom：同pow
7. miner：不进行投票时，为0，投票时，被投票表决（删除或者加入）的地址。（可设为任意值）
8. mixHash：保留字段，正常情况下为0;
9. nonce："0x0000000000000000"代表删除一个singer，"0xffffffffffffffff"代表添加一个signer
10. number：块高度，同pow
11. parentHash：父块hash，同pow
12. receiptsRoot：同pow
13. sha3Uncles：为固定字符串"0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347"
    1. 该字符串为web3.sha3("0xc0",{encoding:'hex'})的结果，其中0xc0为RLP([])的结果（即空数组的rlp编码）
14. size：块大小，同pow
15. stateRoot：同pow
16. timestamp：出块时间，>=parentBlock.timestamp+BLOCK_PERIOD
17. totalDifficulty：同pow
18. transactions：同pow
19. transactionsRoot：同pow
20. uncles：应该始终为空数组[]

# POA出块
POA的块是由signer轮流进行签名的，但是当系统中有一半以上的signer不进行挖矿后，整个链就不能正常工作，必须要保证挖矿的signer人数>=总signer/2+1

1. Prepare阶段，初始化块头部的各项参数
    1. 当不处于检查点时（检查点blockNumber%Epoch==0），如果当前有投票，则对投票进行处理
        1. 对当前所有的投票进行合法性检验
        2. 若当前自己节点有投票，则将coinbase设置为被投票的地址，并且将投票结果写入nonce
    2. 计算Difficulty
        1. in-turn则为2；根据当前signer的数量，当offset = blockNumber%len(signers);signers[offset]==当前signer时，为in-turn
        否则为no-turn
        2. no-turn(out-of-turn)则为1
    3. 设置extra数据，格式如上述所述（POW中，该字段可由miner进行设置）
    4. 设置块时间，块时间=父块时间+出块间隔，如果小于当前时间，则设为当前时间
```go
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error {
	// If the block isn't a checkpoint, cast a random vote (good enough for now)
	header.Coinbase = common.Address{}
	header.Nonce = types.BlockNonce{}

	number := header.Number.Uint64()
	// Assemble the voting snapshot to check which votes make sense
	snap, err := c.snapshot(chain, number-1, header.ParentHash, nil)
	if err != nil {
		return err
	}
	if number%c.config.Epoch != 0 {
		c.lock.RLock()

		// Gather all the proposals that make sense voting on
		addresses := make([]common.Address, 0, len(c.proposals))
		for address, authorize := range c.proposals {
		    // 添加已经有的地址和移除不是signer的地址是非法的
			if snap.validVote(address, authorize) {
				addresses = append(addresses, address)
			}
		}
		// If there's pending proposals, cast a vote on them
		// 从当前的proposals中随机选取投票，将其投票写入块头部
		if len(addresses) > 0 {
			header.Coinbase = addresses[rand.Intn(len(addresses))]
			if c.proposals[header.Coinbase] {
				copy(header.Nonce[:], nonceAuthVote)
			} else {
				copy(header.Nonce[:], nonceDropVote)
			}
		}
		c.lock.RUnlock()
	}
	// Set the correct difficulty
	header.Difficulty = CalcDifficulty(snap, c.signer)

	// Ensure the extra data has all it's components
	if len(header.Extra) < extraVanity {
		header.Extra = append(header.Extra, bytes.Repeat([]byte{0x00}, extraVanity-len(header.Extra))...)
	}
	header.Extra = header.Extra[:extraVanity]

	if number%c.config.Epoch == 0 {
		for _, signer := range snap.signers() {
			header.Extra = append(header.Extra, signer[:]...)
		}
	}
	header.Extra = append(header.Extra, make([]byte, extraSeal)...)

	// Mix digest is reserved for now, set to empty
	header.MixDigest = common.Hash{}

	// Ensure the timestamp has the correct delay
	parent := chain.GetHeader(header.ParentHash, number-1)
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	header.Time = new(big.Int).Add(parent.Time, new(big.Int).SetUint64(c.config.Period))
	if header.Time.Int64() < time.Now().Unix() {
		header.Time = big.NewInt(time.Now().Unix())
	}
	return nil
}
```

2. 调用Finalize
    1. 设置uncle为[]；并且没有块奖励
```go
func (c *Clique) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error) {
	// No block rewards in PoA, so the state remains as is and uncles are dropped
	header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))
	header.UncleHash = types.CalcUncleHash(nil)

	// Assemble and return the final block for sealing
	return types.NewBlock(header, txs, nil, receipts), nil
}
```

3. 调用Seal
    1. 对于period为0的链，不允许生成空块
    2. 验证当前signer是否为合法的signer
    3. 如果当前signer在最近已经对块进行了签名（最近的limit=signer/2+1个块中有该signer签名的块，相关信息保存在Recents中），則終止此次出块操作；
    4. 如果当前的signer为out-of-turn，代表应该由另外一个signer进行签名，则等待一定时间(limit*500毫秒)后在生成本块。（该signer未在最近的limit个块中签名）
    5. 如果当前的signer为in-turn，则等待blockNumber.timestamp-now后对块进行签名（blockNumber.timestamp在prepare中计算得到，大概率大于当前时间），生成块。

```go
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
	header := block.Header()

	// Sealing the genesis block is not supported
	number := header.Number.Uint64()
	if number == 0 {
		return nil, errUnknownBlock
	}
	// For 0-period chains, refuse to seal empty blocks (no reward but would spin sealing)
	if c.config.Period == 0 && len(block.Transactions()) == 0 {
		return nil, errWaitTransactions
	}
	// Don't hold the signer fields for the entire sealing procedure
	c.lock.RLock()
	signer, signFn := c.signer, c.signFn
	c.lock.RUnlock()

	// Bail out if we're unauthorized to sign a block
	snap, err := c.snapshot(chain, number-1, header.ParentHash, nil)
	if err != nil {
		return nil, err
	}
	if _, authorized := snap.Signers[signer]; !authorized {
		return nil, errUnauthorized
	}
	// If we're amongst the recent signers, wait for the next block
	for seen, recent := range snap.Recents {
		if recent == signer {
			// Signer is among recents, only wait if the current block doesn't shift it out
			if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
				log.Info("Signed recently, must wait for others")
				<-stop
				return nil, nil
			}
		}
	}
	// Sweet, the protocol permits us to sign the block, wait for our time
	delay := time.Unix(header.Time.Int64(), 0).Sub(time.Now()) // nolint: gosimple
	if header.Difficulty.Cmp(diffNoTurn) == 0 {
		// It's not our turn explicitly to sign, delay it a bit
		wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime
		// 注意该步骤使用了rand函数，即其随机等待0-delay之间的时间后，生成该块，在signer比较多的时候，能够在一定
		// 程度上保证出块速率，但是也可能造成链的reorg情况比较多（自动切换到链难度最高的链）
		delay += time.Duration(rand.Int63n(int64(wiggle)))

		log.Trace("Out-of-turn signing requested", "wiggle", common.PrettyDuration(wiggle))
	}
	log.Trace("Waiting for slot to sign and propagate", "delay", common.PrettyDuration(delay))

	select {
	case <-stop:
		return nil, nil
	case <-time.After(delay):
	}
	// Sign all the things!
	sighash, err := signFn(accounts.Account{Address: signer}, sigHash(header).Bytes())
	if err != nil {
		return nil, err
	}
	copy(header.Extra[len(header.Extra)-extraSeal:], sighash)

	return block.WithSeal(header), nil
}
```
注意，在Prepare和Seal阶段，均调用了snapshot方法，该方法会apply方法，根据当前头部的信息，计算vote的情况，
并进行相应的更新，如果某个节点加入/剔除出列表投票通过后，则更新signers列表

# POA投票
* signer调用Propose(address, auth)，发起对某个地址的投票，auth为false代表将该地址移除，auth为true代表将某个
地址加入
* 不要尝试添加已有的地址；添加已有的地址，或者是删除不是singer的地址均属于不合法的投票，在实际进行投票计算时，不会将
该部分的投票计算入内。
* poa并没有对signer的投票地址进行限制，即你可以给自己投票，但是结果需要统计所有人的投票才能得出（包括自己将自己移除）
* 投票后的propose并不是直接被删除，如果没有使用discard或停止/重启节点，该部分数据会一直在内存中
* 当signer只有一个时，signer可以自己将自己投票出去，使signer长度为0，此时系统会panic
* 投票的情况会记录到块头部，并依此更新内存中的snapshot，所以当前的signer信息是保存在内存中的（在块为0或者内部检查点时(1024块间隔)，会将数据保存至disk）

## POA clique api

* clique.propose(address, auth)
    * clique.propose(0x281055Afc982d96fAB65b3a49cAc8b878184Cb16, true): 提议添加0x281055Afc982d96fAB65b3a49cAc8b878184Cb16为signer
    * clique.propose(0x281055Afc982d96fAB65b3a49cAc8b878184Cb16, false): 提议将0x281055Afc982d96fAB65b3a49cAc8b878184Cb16从signer中移除
    *如果没有使用clique.discard(address)，那么这个propose会一致存在内存中，miner每次生成块的时候，都会将该propose添加到块中
    

