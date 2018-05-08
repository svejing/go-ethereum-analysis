### Author()  
返回挖出该区块的矿工的接受奖励的地址。
```
// Author implements consensus.Engine, returning the header's coinbase as the
// proof-of-work verified author of the block.
func (ethash *Ethash) Author(header *types.Header) (common.Address, error) {
	return header.Coinbase, nil
}
```


### VerifyHeader()  
判断当前状态是否为ModeFullFake；  
判断该区块头是否已经存在；  
判断该区块的父区块是否存在；  
上述判断均无问题，调用verifyHeader()。
```
// VerifyHeader checks whether a header conforms to the consensus rules of the
// stock Ethereum ethash engine.
func (ethash *Ethash) VerifyHeader(chain consensus.ChainReader, header *types.Header, seal bool) error {
	// If we're running a full engine faking, accept any input as valid
	// ModeFullFake指输入均会被视作有效值
	if ethash.config.PowMode == ModeFullFake {
		return nil
	}
	// Short circuit if the header is known, or it's parent not
	// 该区块头可在链中找到则返回nil
	number := header.Number.Uint64()
	if chain.GetHeader(header.Hash(), number) != nil {
		return nil
	}
	// 父节点不存在，返回ErrUnknownAncestor错误类型
	parent := chain.GetHeader(header.ParentHash, number-1)
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	// Sanity checks passed, do a proper verification
	// 调用实际的区块头验证函数verifyHeader()
	return ethash.verifyHeader(chain, header, parent, false, seal)
}
```


### VerifyHeaders()  
判断模式是否为ModeFullFake，若是则接收所有输入；  
使用多个workers验证多个Headers。
```
// VerifyHeaders is similar to VerifyHeader, but verifies a batch of headers
// concurrently. The method returns a quit channel to abort the operations and
// a results channel to retrieve the async verifications.
func (ethash *Ethash) VerifyHeaders(chain consensus.ChainReader, headers []*types.Header, seals []bool) (chan<- struct{}, <-chan error) {
	// If we're running a full engine faking, accept any input as valid
	if ethash.config.PowMode == ModeFullFake || len(headers) == 0 {
		abort, results := make(chan struct{}), make(chan error, len(headers))
		for i := 0; i < len(headers); i++ {
			results <- nil
		}
		return abort, results
	}

	// Spawn as many workers as allowed threads
	workers := runtime.GOMAXPROCS(0)
	if len(headers) < workers {
		workers = len(headers)
	}

	// Create a task channel and spawn the verifiers
	var (
		inputs = make(chan int)
		done   = make(chan int, workers)
		errors = make([]error, len(headers))
		abort  = make(chan struct{})
	)
	for i := 0; i < workers; i++ {
		go func() {
			for index := range inputs {
				errors[index] = ethash.verifyHeaderWorker(chain, headers, seals, index)
				done <- index
			}
		}()
	}

	errorsOut := make(chan error, len(headers))
	go func() {
		defer close(inputs)
		var (
			in, out = 0, 0
			checked = make([]bool, len(headers))
			inputs  = inputs
		)
		for {
			select {
			case inputs <- in:
				if in++; in == len(headers) {
					// Reached end of headers. Stop sending to workers.
					inputs = nil
				}
			case index := <-done:
				for checked[index] = true; checked[out]; out++ {
					errorsOut <- errors[out]
					if out == len(headers)-1 {
						return
					}
				}
			case <-abort:
				return
			}
		}
	}()
	return abort, errorsOut
}
```


### verifyHeaderWorker()  
寻找该区块的父区块，若父区块不存在则返回错误类型ErrUnknownAncestor；  
若该区块已存在于链上，则不再需要验证其区块头；  
若链上未找到该区块，则调用verifyHeader()验证该区块头。

```
func (ethash *Ethash) verifyHeaderWorker(chain consensus.ChainReader, headers []*types.Header, seals []bool, index int) error {
	var parent *types.Header
	if index == 0 {
		parent = chain.GetHeader(headers[0].ParentHash, headers[0].Number.Uint64()-1)
	} else if headers[index-1].Hash() == headers[index].ParentHash {
		parent = headers[index-1]
	}
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	if chain.GetHeader(headers[index].Hash(), headers[index].Number.Uint64()) != nil {
		return nil // known block
	}
	return ethash.verifyHeader(chain, headers[index], parent, false, seals[index])
}
```


### VerifyUncles()  
验证该区块的叔块：  
判断模式是否为测试模式（ModeFullFake）；  
判断包含的叔区块是否超过2个；
验证叔区块是否合乎以太坊标准（7代以内，区块头有效，未曾以叔区块的身份出现过）。
```
// VerifyUncles verifies that the given  block's uncles conform to the consensus
// rules of the stock Ethereum ethash engine.
func (ethash *Ethash) VerifyUncles(chain consensus.ChainReader, block *types.Block) error {
	// If we're running a full engine faking, accept any input as valid
	// 模式为ModeFullFake（测试用）时，任意输入均有效
	if ethash.config.PowMode == ModeFullFake {
		return nil
	}
	// Verify that there are at most 2 uncles included in this block
	// 区块中最多包括2个叔区块（maxUnlces = 2）
	if len(block.Uncles()) > maxUncles {
		return errTooManyUncles
	}
	// Gather the set of past uncles and ancestors
	uncles, ancestors := set.New(), make(map[common.Hash]*types.Header)

	// number存放父区块的区块号
	// parent存放父区块的哈希值
	number, parent := block.NumberU64()-1, block.ParentHash()
	// 将该区块及其上数7代祖先存入ancestors中
	for i := 0; i < 7; i++ {
		ancestor := chain.GetBlock(parent, number)
		if ancestor == nil {
			break
		}
		ancestors[ancestor.Hash()] = ancestor.Header()
		for _, uncle := range ancestor.Uncles() {
			uncles.Add(uncle.Hash())
		}
		parent, number = ancestor.ParentHash(), number-1
	}
	ancestors[block.Hash()] = block.Header()
	uncles.Add(block.Hash())

	// Verify each of the uncles that it's recent, but not an ancestor
	for _, uncle := range block.Uncles() {
		// Make sure every uncle is rewarded only once
		hash := uncle.Hash()
		// 若该叔块已被奖励过（即已是之前某块的叔块），则返回错误类型errDuplicateUncle
		if uncles.Has(hash) {
			return errDuplicateUncle
		}
		// 未被奖励过则存入uncles中
		uncles.Add(hash)

		// Make sure the uncle has a valid ancestry
		// 该叔块已是该区块的祖先，则返回错误类型errUncleIsAncestor
		if ancestors[hash] != nil {
			return errUncleIsAncestor
		}
		// 叔块的父区块不在该区块的祖先中（即其实并不是叔块），或叔块的父区块即该区块的父区块（即与该区块为兄弟节点），则返回错误类型errDanglingUncle
		if ancestors[uncle.ParentHash] == nil || uncle.ParentHash == block.ParentHash() {
			return errDanglingUncle
		}
		// 验证该叔块的区块头是否有效
		if err := ethash.verifyHeader(chain, uncle, ancestors[uncle.ParentHash], true, true); err != nil {
			return err
		}
	}
	return nil
}
```



### verifyHeader()  
判断区块头中附加数据的长度是否超过参数中规定的附加数据的最大长度；  
验证区块头的时间戳;  
验证该区块的挖矿难度；  
验证gas限制；  
验证已使用的gas是否小于gas limit；  
验证该区块的gas limit是否满足参数设定；  
验证该区块的区块号是否是其父区块的区块号+1；   
验证封装，调用VerifySeal()；  
判断该区块是否和DAO的硬分叉有关；  
验证EIP 150分叉时的区块hash是否正确。  
```
// verifyHeader checks whether a header conforms to the consensus rules of the
// stock Ethereum ethash engine.
// See YP section 4.3.4. "Block Header Validity"
func (ethash *Ethash) verifyHeader(chain consensus.ChainReader, header, parent *types.Header, uncle bool, seal bool) error {
	// Ensure that the header's extra-data section is of a reasonable size
	if uint64(len(header.Extra)) > params.MaximumExtraDataSize {
		return fmt.Errorf("extra-data too long: %d > %d", len(header.Extra), params.MaximumExtraDataSize)
	}
	// Verify the header's timestamp
	// 若该区块是叔节点，验证其时间戳是否大于2^256-1（MaxBig256定义在common\math\big.go中）；  
	// 若不是叔节点，判断该区块的时间戳是否超过了所允许的某未来时间节点（allowedFutureBolcktime被规定为15s）；
	if uncle {
		if header.Time.Cmp(math.MaxBig256) > 0 {
			return errLargeBlockTime
		}
	} else {
		if header.Time.Cmp(big.NewInt(time.Now().Add(allowedFutureBlockTime).Unix())) > 0 {
			return consensus.ErrFutureBlock
		}
	}  
	// 判断该区块时间戳是否小于其父节点的时间戳，若是，则返回errZeroBlockTime错误类型。
	if header.Time.Cmp(parent.Time) <= 0 {
		return errZeroBlockTime
	}  
	// 通过该区块时间戳和其父节点的挖矿难度计算该区块应有的挖矿难度，验证该区块实际所使用的挖矿难度是否满足上述难度。
	// Verify the block's difficulty based in it's timestamp and parent's difficulty
	expected := ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)  
	// 若不满足，则返回“无效难度”错误
	if expected.Cmp(header.Difficulty) != 0 {
		return fmt.Errorf("invalid difficulty: have %v, want %v", header.Difficulty, expected)
	}
	// Verify that the gas limit is <= 2^63-1
	cap := uint64(0x7fffffffffffffff)
	if header.GasLimit > cap {
		return fmt.Errorf("invalid gasLimit: have %v, max %v", header.GasLimit, cap)
	}
	// Verify that the gasUsed is <= gasLimit
	if header.GasUsed > header.GasLimit {
		return fmt.Errorf("invalid gasUsed: have %d, gasLimit %d", header.GasUsed, header.GasLimit)
	}  
	// 计算diff=|parent.GasLimit-header.GasLimit|
	// Verify that the gas limit remains within allowed bounds
	diff := int64(parent.GasLimit) - int64(header.GasLimit)
	if diff < 0 {
		diff *= -1
	}  
	//计算limit=parent.GasLimit/1024（GasLimitBoundDivisor设定在protocol_params中）
	limit := parent.GasLimit / params.GasLimitBoundDivisor  
	//判断差值是否大于等于所限制的，该区块的GasLimit是否超过设定值5000（MinGasLimit设定在protocol_params中）
	if uint64(diff) >= limit || header.GasLimit < params.MinGasLimit {
		return fmt.Errorf("invalid gas limit: have %d, want %d += %d", header.GasLimit, parent.GasLimit, limit)
	}
	// Verify that the block number is parent's +1
	if diff := new(big.Int).Sub(header.Number, parent.Number); diff.Cmp(big.NewInt(1)) != 0 {
		return consensus.ErrInvalidNumber
	}
	// Verify the engine specific seal securing the block
	if seal {
		if err := ethash.VerifySeal(chain, header); err != nil {
			return err
		}
	}
	// If all checks passed, validate any special fields for hard forks
	// 判断该区块是否和DAO的硬分叉有关
	if err := misc.VerifyDAOHeaderExtraData(chain.Config(), header); err != nil {
		return err
	}
	if err := misc.VerifyForkHashes(chain.Config(), header, uncle); err != nil {
		return err
	}
	return nil
}
```


### CalcDifficulty()
调用CalcDifficulty()计算该区块的挖矿难度。
```
// CalcDifficulty is the difficulty adjustment algorithm. It returns
// the difficulty that a new block should have when created at time
// given the parent block's time and difficulty.
func (ethash *Ethash) CalcDifficulty(chain consensus.ChainReader, time uint64, parent *types.Header) *big.Int {
	return CalcDifficulty(chain.Config(), time, parent)
}
```


### CalcDifficulty()  
被上一函数调用，区别在于输入参数不一样。  
主要功能是根据config参数决定调用何种难度计算方式，使用该区块的时间戳和其父区块的区块号来计算该区块的挖矿难度：  
判断该区块处于Byzantium阶段或Homestead阶段或Frontier阶段（根据params\config.go，目前均处于Byzantium阶段），并调用相应的难度计算函数。  
```
// CalcDifficulty is the difficulty adjustment algorithm. It returns
// the difficulty that a new block should have when created at time
// given the parent block's time and difficulty.
func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
	next := new(big.Int).Add(parent.Number, big1)
	switch {
	case config.IsByzantium(next):
		return calcDifficultyByzantium(time, parent)
	case config.IsHomestead(next):
		return calcDifficultyHomestead(time, parent)
	default:
		return calcDifficultyFrontier(time, parent)
	}
}
```

### calcDifficultyByzantium()    

父区块有叔块：

diff = ( parent_diff + ( parent_diff / 2048 * max( 2 - (timestamp - parent.timestamp) / 9) , -99 ) + 2 ^ ( periodCount - 2 )

父区块无叔块：

diff = ( parent_diff + ( parent_diff / 2048 * max( 1 - (timestamp - parent.timestamp) / 9) , -99 ) + 2 ^ ( periodCount - 2 )

```
// calcDifficultyByzantium is the difficulty adjustment algorithm. It returns
// the difficulty that a new block should have when created at time given the
// parent block's time and difficulty. The calculation uses the Byzantium rules.
func calcDifficultyByzantium(time uint64, parent *types.Header) *big.Int {
	// https://github.com/ethereum/EIPs/issues/100.
	// algorithm:
	// diff = (parent_diff +
	//         (parent_diff / 2048 * max((2 if len(parent.uncles) else 1) - ((timestamp - parent.timestamp) // 9), -99))
	//        ) + 2^(periodCount - 2)

	// bigTime存储该区块的时间戳
	// bigParentTime存储其父区块的时间戳
	bigTime := new(big.Int).SetUint64(time)
	bigParentTime := new(big.Int).Set(parent.Time)

	// holds intermediate values to make the algo easier to read & audit
	x := new(big.Int)
	y := new(big.Int)

	// (2 if len(parent_uncles) else 1) - (block_timestamp - parent_timestamp) // 9
	// x = (bigTime - bigParentTime) / 9 （即x为该区块与父区块的时间戳的差值再除以9）
	// 如果父区块有叔块，则 x = 2 - x ，否则 x = 1 - x
	x.Sub(bigTime, bigParentTime)
	x.Div(x, big9)
	if parent.UncleHash == types.EmptyUncleHash {
		x.Sub(big1, x)
	} else {
		x.Sub(big2, x)
	}
	// max((2 if len(parent_uncles) else 1) - (block_timestamp - parent_timestamp) // 9, -99)
	// 比较x与-99，取它们中较大的值赋给x
	if x.Cmp(bigMinus99) < 0 {
		x.Set(bigMinus99)
	}
	// parent_diff + (parent_diff / 2048 * max((2 if len(parent.uncles) else 1) - ((timestamp - parent.timestamp) // 9), -99))
	// y = parent_diff / 2048 (DifficultyBoundDivisor定义在params\protocol_params.go中)
	y.Div(parent.Difficulty, params.DifficultyBoundDivisor)
	// x = y * x
	x.Mul(y, x)
	// x = parent_diff + x
	x.Add(parent.Difficulty, x)

	// minimum difficulty can ever be (before exponential factor)
	// 比较x与131072，取其中较大的赋值给x（MinimumDifficulty定义在params\protocol_params.go中）
	if x.Cmp(params.MinimumDifficulty) < 0 {
		x.Set(params.MinimumDifficulty)
	}
	// calculate a fake block number for the ice-age delay:
	//   https://github.com/ethereum/EIPs/pull/669
	//   fake_block_number = min(0, block.number - 3_000_000
	// ice-age : 冰河期
	// "bomb" ：难度炸弹
	// 冰河期指出块速度越来越慢（挖矿难度极高），为了由PoW转为PoS所设定的难度机制
	// 原有的注释错误，应改为fakeBlockNumber = max(0, block.number - 3_000_000)
	fakeBlockNumber := new(big.Int)
	if parent.Number.Cmp(big2999999) >= 0 {
		fakeBlockNumber = fakeBlockNumber.Sub(parent.Number, big2999999) // Note, parent is 1 less than the actual block number
	}
	// for the exponential factor
	// periodCount = fakeBlockNumber / 100000，比如现在的periodCount = 25(2018.5.8)
	periodCount := fakeBlockNumber
	periodCount.Div(periodCount, expDiffPeriod)

	// the exponential factor, commonly referred to as "the bomb"
	// diff = diff + 2^(periodCount - 2)
	// 此处就是"bomb"的实现，随着时间迁移，难度将指数级增长
	if periodCount.Cmp(big1) > 0 {
		y.Sub(periodCount, big2)
		y.Exp(big2, y, nil)
		x.Add(x, y)
	}
	return x
}
```



### VerifySeal()  
判断该区块的模式是否为ModeFake或ModeFullFake（这2种模式通常是测试时使用）；  
判断当前PoW类型是否为shared（通常测试用），若是，则转而验证该shared型；  
验证该区块的挖矿难度是否为有效值；  
调用hashimoto()计算digest和result，验证该区块头的hash（此处的hash指除了MixDigest和Nonce外的区块头）和nonce，并验证PoW是否有效。
```
// VerifySeal implements consensus.Engine, checking whether the given block satisfies
// the PoW difficulty requirements.
func (ethash *Ethash) VerifySeal(chain consensus.ChainReader, header *types.Header) error {
	// If we're running a fake PoW, accept any seal as valid
	if ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
		time.Sleep(ethash.fakeDelay)
		if ethash.fakeFail == header.Number.Uint64() {
			return errInvalidPoW
		}
		return nil
	}
	// If we're running a shared PoW, delegate verification to it
	if ethash.shared != nil {
		return ethash.shared.VerifySeal(chain, header)
	}  
	// Sign()定义在mobile\big.go中  
	// Sign returns:  
	//	-1 if x <  0  
	//	 0 if x == 0  
	//	+1 if x >  0
	// Ensure that we have a valid difficulty for the block
	if header.Difficulty.Sign() <= 0 {
		return errInvalidDifficulty
	}
	// Recompute the digest and PoW value and verify against the header  
	// number是该区块的区块编号
	number := header.Number.Uint64()

	// 寻找或生成该区块号对应的存储，具体见ethash.go
	cache := ethash.cache(number)  
	// 通过区块号得到datasetSize
	size := datasetSize(number)
	// 模式是ModeTest时，固定size
	if ethash.config.PowMode == ModeTest {
		size = 32 * 1024
	}
	// 调用hashimotoLight()计算digest[]和result[]
	digest, result := hashimotoLight(size, cache.cache, header.HashNoNonce().Bytes(), header.Nonce.Uint64())
	// Caches are unmapped in a finalizer. Ensure that the cache stays live
	// until after the call to hashimotoLight so it's not unmapped while being used.
	runtime.KeepAlive(cache)

	// 验证区块头中的MixDigest[]是否与我们计算所得的digest[]一致（即检验hash（此处的hash指除了MixDigest和Nonce外的区块头）和nonce）
	if !bytes.Equal(header.MixDigest[:], digest) {
		return errInvalidMixDigest
	}
	// maxUint256 = 2^256
	// 当 result > maxUint256 / Difficulty 时，返回错误类型errInvalidPow
	target := new(big.Int).Div(maxUint256, header.Difficulty)
	if new(big.Int).SetBytes(result).Cmp(target) > 0 {
		return errInvalidPoW
	}
	return nil
}
```

