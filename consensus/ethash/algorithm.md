### datasetSize()  
通过当前区块号得到datasetSize (ethash挖矿所需数据集的大小)；
```
// datasetSize returns the size of the ethash mining dataset that belongs to a certain
// block number.
func datasetSize(block uint64) uint64 {  
	// epochLength = 30000
	epoch := int(block / epochLength)
	// maxEpoch = 2048
	// epoch < 2048 时，查表得到size
	if epoch < maxEpoch {
		return datasetSizes[epoch]
	}
	// 否则，调用calDatasetSize()计算
	return calcDatasetSize(epoch)
}
```  




### calDatasetSize()  
当epoch > 2048时，调用该函数计算size；
```
// calcDatasetSize calculates the dataset size for epoch. The dataset size grows linearly,
// however, we always take the highest prime below the linearly growing threshold in order
// to reduce the risk of accidental regularities leading to cyclic behavior.
func calcDatasetSize(epoch int) uint64 {
	// datasetInitBytes = 2^30
	// datasetGrowthBytes = 2^23
	// mixBytes = 128
	size := datasetInitBytes + datasetGrowthBytes*uint64(epoch) - mixBytes
	// 调整size保证其为素数
	for !new(big.Int).SetUint64(size / mixBytes).ProbablyPrime(1) { // Always accurate for n < 2^64
		size -= 2 * mixBytes
	}
	return size
}
```


### makeHasher()  
使用已有的哈希函数生成一个新的哈希函数
```
// makeHasher creates a repetitive hasher, allowing the same hash data structures
// to be reused between hash runs instead of requiring new ones to be created.
// The returned function is not thread safe!
func  makeHasher(h hash.Hash) hasher {
	return func(dest []byte, data []byte) {
		h.Write(data)
		h.Sum(dest[:0])
		h.Reset()
	}
}
```


### fnv()  
ethash中的fnv()与常用的fnv-1的区别在于：  
fnv-1中，异或运算时使用的data是8位；  
此处的fnv中，异或运算时使用的data(即b)是32位。
```
// fnv is an algorithm inspired by the FNV hash, which in some cases is used as
// a non-associative substitute for XOR. Note that we multiply the prime with
// the full 32-bit input, in contrast with the FNV-1 spec which multiplies the
// prime with one byte (octet) in turn.
func fnv(a, b uint32) uint32 {
	return a*0x01000193 ^ b
}
```


### fnvHash()  
uint32型数组的fnv算法的具体实现。
```
// fnvHash mixes in data into mix using the ethash fnv method.
func fnvHash(mix []uint32, data []uint32) {
	for i := 0; i < len(mix); i++ {
		mix[i] = mix[i]*0x01000193 ^ data[i]
	}
}
```


### generateDatasetItem()
多次哈希生成随机byte型数组mix；  
```
// generateDatasetItem combines data from 256 pseudorandomly selected cache nodes,
// and hashes that to compute a single dataset node.
func generateDatasetItem(cache []uint32, index uint32, keccak512 hasher) []byte {
	// Calculate the number of theoretical rows (we use one buffer nonetheless)
	// hashWords = 16
	rows := uint32(len(cache) / hashWords)

	// Initialize the mix
	// hashBytes = 64
	mix := make([]byte, hashBytes)

	// 填充mix，伪随机序列（将整数序列化至byte型数组中）
	binary.LittleEndian.PutUint32(mix, cache[(index%rows)*hashWords]^index)
	for i := 1; i < hashWords; i++ {
		binary.LittleEndian.PutUint32(mix[i*4:], cache[(index%rows)*hashWords+uint32(i)])
	}
	// 使用keccak512对mix做原地哈希
	// 其中，keccak512 := makeHasher(sha3.NewKeccak512())
	keccak512(mix, mix)

	// Convert the mix to uint32s to avoid constant bit shifting
	// 将整数从byte数组中反序列化出来，并存入uint32型数组中
	intMix := make([]uint32, hashWords)
	for i := 0; i < len(intMix); i++ {
		intMix[i] = binary.LittleEndian.Uint32(mix[i*4:])
	}
	// fnv it with a lot of random cache nodes based on index
	// datasetParents = 256
	// 使用fnv算法哈希intMix（与fnv-1所不同的是，此处的fnv将素数与整个32位的输入相乘）
	for i := uint32(0); i < datasetParents; i++ {
		parent := fnv(index^i, intMix[i%16]) % rows
		fnvHash(intMix, cache[parent*hashWords:])
	}
	// Flatten the uint32 mix into a binary one and return
	// 将intMix序列化至byte型数组mix中
	for i, val := range intMix {
		binary.LittleEndian.PutUint32(mix[i*4:], val)
	}
	//再次对mix原地哈希
	keccak512(mix, mix)
	return mix
}
```


### hashimoto()  
具体的哈希运算，将返回两个大小32的byte型数组。
```
// hashimoto aggregates data from the full dataset in order to produce our final
// value for a particular header hash and nonce.
func hashimoto(hash []byte, nonce uint64, size uint64, lookup func(index uint32) []uint32) ([]byte, []byte) {
	// Calculate the number of theoretical rows (we use one buffer nonetheless)
	// mixBytes = 128
	rows := uint32(size / mixBytes)

	// Combine header+nonce into a 64 byte seed
	// seed = hash + nonce
	// 此处的hash指除了MixDigest和Nonce外的区块头
	seed := make([]byte, 40)
	copy(seed, hash)
	binary.LittleEndian.PutUint64(seed[32:], nonce)

	// 对seed做一次SHA-512哈希，seed[]大小64
	seed = crypto.Keccak512(seed)
	// 取seed前32位做seedHead
	seedHead := binary.LittleEndian.Uint32(seed)

	// Start the mix with replicated seed
	// seed[]转化为uint32型数组mix[]，mix[]大小32，因此mix[]前后两半是一样的
	mix := make([]uint32, mixBytes/4)
	for i := 0; i < len(mix); i++ {
		mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])
	}
	// Mix in random dataset nodes
	// 调用lookup()，随机获取数据存入temp[]中，使用temp[]向mix[]中混入数据（混入方式使用fnvHash()）
	temp := make([]uint32, len(mix))

	// loopAcceses = 64
	for i := 0; i < loopAccesses; i++ {
		parent := fnv(uint32(i)^seedHead, mix[i%len(mix)]) % rows
		for j := uint32(0); j < mixBytes/hashBytes; j++ {
			copy(temp[j*hashWords:], lookup(2*parent+j))
		}
		fnvHash(mix, temp)
	}
	// Compress mix
	// 折叠mix[]，大小压缩为原来的1/4（8），使用fnv()（新mix[]中的每个元素都经历了3次fnv()）
	for i := 0; i < len(mix); i += 4 {
		mix[i/4] = fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
	}
	mix = mix[:len(mix)/4]

	// HashLength = 32
	// 将uint32型数组mix[]直接转化为byte型数组digest[]
	digest := make([]byte, common.HashLength)
	for i, val := range mix {
		binary.LittleEndian.PutUint32(digest[i*4:], val)
	}
	// 返回digest[]
	// 合并seed[]和digest[]并对其SHA-256一次，将哈希后的值也作为返回值
	return digest, crypto.Keccak256(append(seed, digest...))
}
```
上述代码即区块头中hash（此处的hash指除了MixDigest和Nonce外的区块头）与nonce的具体哈希变换过程，可由下图概括，可参考[[以太坊源代码分析]III. 挖矿和共识算法的奥秘](https://blog.csdn.net/teaspring/article/details/78050274)  
![image](https://img-blog.csdn.net/20171009120120925?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGVhc3ByaW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
  
  
  
### hashimotoLight()  
生成data供hashimoto()使用；  
（以非线性表查找方式进行的哈希运算，这里使用非线性表cache{}，数据规模较小）
```
// hashimotoLight aggregates data from the full dataset (using only a small
// in-memory cache) in order to produce our final value for a particular header
// hash and nonce.  
// 此处的hash指除了MixDigest和Nonce外的区块头
func hashimotoLight(size uint64, cache []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	// 利用SHA3-512生成新的哈希函数
	keccak512 := makeHasher(sha3.NewKeccak512())

	lookup := func(index uint32) []uint32 {
		// 调用generateDatasetItem()生成随机序列rawData
		rawData := generateDatasetItem(cache, index, keccak512)

		// 将rawData中数据反序列化，存入uint32型数组data中
		data := make([]uint32, len(rawData)/4)
		for i := 0; i < len(data); i++ {
			data[i] = binary.LittleEndian.Uint32(rawData[i*4:])
		}
		return data
	}
	return hashimoto(hash, nonce, size, lookup)
}
```



### hashimotoFull()  
生成dataset供hashimoto()使用；  
（以非线性表查找方式进行的哈希运算，这里使用非线性表dataset{}，数据规模较大）
```
// hashimotoFull aggregates data from the full dataset (using the full in-memory
// dataset) in order to produce our final value for a particular header hash and
// nonce.
// 此处的hash指除了MixDigest和Nonce外的区块头
func hashimotoFull(dataset []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	lookup := func(index uint32) []uint32 {
		offset := index * hashWords
		return dataset[offset : offset+hashWords]
	}
	return hashimoto(hash, nonce, uint64(len(dataset))*4, lookup)
}
```