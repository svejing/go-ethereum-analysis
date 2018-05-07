### Author()  
返回挖出该区块的矿工的接受奖励的地址
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



### verifyHeader()  
判断区块头中附加数据的长度是否超过参数中规定的附加数据的最大长度；  
验证区块头的时间戳;  
验证该区块的挖矿难度；  
验证gas限制；  
验证已使用的gas是否小于gas limit；  
验证该区块的gas limit是否满足参数设定；  
验证该区块的区块号是否是其父区块的区块号+1；   
验证封装，调用VerifySeal()；  
验证判断该区块是否和DAO的硬分叉有关；  
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
	// 若该区块有叔节点，验证其时间戳是否大于2^256-1（MaxBig256定义在common\math\big.go中）；  
	// 若无叔节点，判断该区块的时间戳是否超过了所允许的某未来时间节点（allowedFutureBolcktime被规定为15s）；
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

