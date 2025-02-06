## 处理流程
1. 通过路由询价获取最佳交易路径
2. 判断是否需要 approve（使用主网合约地址："TCFNp179Lg46D16zKoumd4Poa2WFFdtqYj"）
3. 如不需要 approve 则开始构建交易
4. 执行兑换交易

## 合约参数
根据 [SunSwap 文档](https://docs-zh.sun.io/kai-fa-zhe/dui-huan/zhi-neng-lu-you)，`swapExactInput` 函数需要 5 个参数：

1. 路径数组（代币地址列表）
2. Pool 版本数组
3. 相邻 Pool 版本的长度数组
4. 手续费率数组
5. 兑换数据元组：
   - 输入金额
   - 最小接受输出金额
   - 接收地址
   - 截止时间
- 我们现在不太清楚的是 **pool的版本数组**，**相邻pool版本的长度数组**，这两个参数的计算方式，现在完整的计算逻辑是这样的，根据route中的 poolVersions字段通过compressPath这个方法来计算，然后来得出这两个数据，需要重点帮忙review 下这个计算的逻辑是不是有问题。

## Pool 版本压缩算法
Pool 版本数组和长度数组通过 `compressPath` 函数计算：
```typescript
function compressPath(path: string[]) {
  const versions: string[] = []
  const lengths: number[] = []
  let i = 0
  while (i < path.length) {
    const currentVersion = path[i]
    versions.push(currentVersion)
    let count = i === 0 ? 2 : 1
    while (i + 1 < path.length && path[i] === path[i + 1]) {
      count++
      i++
    }
    lengths.push(count)
    i++
  }
  return {
    versions,
    lengths
  }
}
```

### 示例
输入: `["v2", "v2", "v3", "v3", "v3"]`
输出: 
- versions: `["v2", "v3"]`
- lengths: `[3, 3]`
  PS：这里的versionLen第一个元素需要 +1。这里的length不仅仅是versions的元素重复计数，实际是version在path中的占用的address的数量，因此versionLen[0] 需要加一

## 交易构建过程

### 1. 地址转换
```typescript
const addressToHex = (address: string) => {
try {
if (!address) return ''
return tronWeb.address.toHex(address).replace(/^41/, '0x')
} catch (error) {
console.error('地址转换错误:', error)
return ''
}
}
```

### 2. 参数构造

> originRoute 为询价返回值

```typescript
const args = [
      originRoute.tokens.map(addressToHex),
      versions,
      lengths.map(String),
      originRoute.poolFees.map(String),
      [
        parseUnits(originRoute.amountIn, fromDecimals).toString(),
        parseUnits(
          String(Number(originRoute.amountOut) / 2),
          toDecimals
        ).toString(),
        addressToHex(toAddress),
        dayjs().add(600, 'second').unix().toString()
      ]
    ]
```
### 3. ABI 定义

```typescript
    const abi = {
      inputs: [
        { type: 'address[]', name: 'path' },
        { type: 'string[]', name: 'poolVersion' },
        { type: 'uint256[]', name: 'versionLen' },
        { type: 'uint24[]', name: 'fees' },
        {
          components: [
            { type: 'uint256', name: 'amountIn' },
            { type: 'uint256', name: 'amountOutMin' },
            { type: 'address', name: 'to' },
            { type: 'uint256', name: 'deadline' }
          ],
          type: 'tuple',
          name: 'data'
        }
      ],
      name: 'swapExactInput',
      type: 'function'
    }
```
    
### 4. 交易构建
> 通过 encodeParamsV2ByABI 解析得到data后再通过 triggerSmartContract获取的完整的 calldata
```typescript
  const parameter = tronWeb.utils.abi.encodeParamsV2ByABI(abi, args)
```
### 5. 交易执行
```typescript
  const functionSelector =
      'swapExactInput(address[],string[],uint256[],uint24[],(uint256,uint256,address,uint256))'
    const res = await tronWeb.transactionBuilder.triggerSmartContract(
      'TCFNp179Lg46D16zKoumd4Poa2WFFdtqYj',
      functionSelector,
      {
        feeLimit: 150000000,
        callValue: tronWeb.toSun(callValue),
        rawParameter: parameter
      },
      [],
      owner
    )

```

## 问题
1. 需要重点帮忙确认的就是 pool的版本数组，相邻pool版本的长度数组的计算逻辑是不是对的
2. 用的合约地址是否正确 `TCFNp179Lg46D16zKoumd4Poa2WFFdtqY`
