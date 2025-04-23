## 사용자 인터페이스

이번 마일스톤에서는 풀에서 유동성을 제거하고 누적된 수수료를 수집하는 기능을 추가했습니다. 따라서 사용자가 유동성을 제거할 수 있도록 사용자 인터페이스에 이러한 변경 사항을 반영해야 합니다.

## 포지션 가져오기

사용자가 제거할 유동성 양을 선택할 수 있도록 하려면 먼저 풀에서 사용자의 포지션을 가져와야 합니다. 이를 더 쉽게 만들기 위해 Manager 컨트랙트에 헬퍼 함수를 추가하여 특정 풀에서 사용자 포지션을 반환할 수 있습니다.

```solidity
function getPosition(GetPositionParams calldata params)
    public
    view
    returns (
        uint128 liquidity,
        uint256 feeGrowthInside0LastX128,
        uint256 feeGrowthInside1LastX128,
        uint128 tokensOwed0,
        uint128 tokensOwed1
    )
{
    IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

    (
        liquidity,
        feeGrowthInside0LastX128,
        feeGrowthInside1LastX128,
        tokensOwed0,
        tokensOwed1
    ) = pool.positions(
        keccak256(
            abi.encodePacked(
                params.owner,
                params.lowerTick,
                params.upperTick
            )
        )
    );
}
```

이렇게 하면 프론트엔드에서 풀 주소와 포지션 키를 계산하는 번거로움에서 벗어날 수 있습니다.

그런 다음 사용자가 포지션 범위를 입력했을 때 포지션을 가져올 수 있습니다.

```js
const getAvailableLiquidity = debounce((amount, isLower) => {
  const lowerTick = priceToTick(isLower ? amount : lowerPrice);
  const upperTick = priceToTick(isLower ? upperPrice : amount);

  const params = {
    tokenA: token0.address,
    tokenB: token1.address,
    fee: fee,
    owner: account,
    lowerTick: nearestUsableTick(lowerTick, feeToSpacing[fee]),
    upperTick: nearestUsableTick(upperTick, feeToSpacing[fee]),
  }

  manager.getPosition(params)
    .then(position => setAvailableAmount(position.liquidity.toString()))
    .catch(err => console.error(err));
}, 500);
```

## 풀 주소 가져오기

`burn` 및 `collect`를 풀에서 호출해야 하므로 여전히 프론트엔드에서 풀의 주소를 계산해야 합니다. 풀 주소는 salt와 컨트랙트 코드의 해시가 필요한 `CREATE2` opcode를 사용하여 계산된다는 것을 상기하십시오. 다행히 Ether.js에는 JavaScript에서 `CREATE2`를 계산할 수 있는 `getCreate2Address` 함수가 있습니다.

```js
const sortTokens = (tokenA, tokenB) => {
  return tokenA.toLowerCase() < tokenB.toLowerCase ? [tokenA, tokenB] : [tokenB, tokenA];
}

const computePoolAddress = (factory, tokenA, tokenB, fee) => {
  [tokenA, tokenB] = sortTokens(tokenA, tokenB);

  return ethers.utils.getCreate2Address(
    factory,
    ethers.utils.keccak256(
      ethers.utils.solidityPack(
        ['address', 'address', 'uint24'],
        [tokenA, tokenB, fee]
      )),
    poolCodeHash
  );
}
```

그러나 풀의 codehash는 하드 코딩되어야 합니다. 해시를 계산하기 위해 프론트엔드에 코드를 저장하고 싶지 않기 때문입니다. 따라서 Forge를 사용하여 해시를 얻을 것입니다.

```shell
$ forge inspect UniswapV3Pool bytecode| xargs cast keccak
0x...
```

그런 다음 출력 값을 JS 상수에 사용합니다.

```js
const poolCodeHash = "0x9dc805423bd1664a6a73b31955de538c338bac1f5c61beb8f4635be5032076a2";
```

## 유동성 제거

유동성 양과 풀 주소를 얻은 후 `burn`을 호출할 준비가 되었습니다.

```js
const removeLiquidity = (e) => {
  e.preventDefault();

  if (!token0 || !token1) {
    return;
  }

  setLoading(true);

  const lowerTick = nearestUsableTick(priceToTick(lowerPrice), feeToSpacing[fee]);
  const upperTick = nearestUsableTick(priceToTick(upperPrice), feeToSpacing[fee]);

  pool.burn(lowerTick, upperTick, amount)
    .then(tx => tx.wait())
    .then(receipt => {
      if (!receipt.events[0] || receipt.events[0].event !== "Burn") {
        throw Error("Missing Burn event after burning!");
      }

      const amount0Burned = receipt.events[0].args.amount0;
      const amount1Burned = receipt.events[0].args.amount1;

      return pool.collect(account, lowerTick, upperTick, amount0Burned, amount1Burned)
    })
    .then(tx => tx.wait())
    .then(() => toggle())
    .catch(err => console.error(err));
}
```

번(burn)이 성공적으로 완료되면 번(burn) 중에 해제된 토큰 양을 수집하기 위해 즉시 `collect`를 호출합니다.