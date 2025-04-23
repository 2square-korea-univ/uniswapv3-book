## 첫 번째 스왑

이제 유동성이 확보되었으니, 첫 번째 스왑을 수행해 보겠습니다!

### 스왑 양 계산하기

가장 첫 번째 단계는 당연히 스왑 양을 계산하는 방법을 알아내는 것입니다. 그리고 다시 한번, 우리가 ETH로 거래할 USDC의 양을 임의로 정하고 하드 코딩해 보겠습니다. 42 USDC로 합시다! 우리는 42 USDC로 ETH를 구매할 것입니다.

판매할 토큰의 양을 결정한 후에는, 그 대가로 얼마나 많은 토큰을 받게 될지 계산해야 합니다. Uniswap V2에서는 현재 풀의 준비금을 사용했겠지만, Uniswap V3에서는 $L$과 $\sqrt{P}$를 사용하며, 가격 범위 내에서 스왑할 때 $\sqrt{P}$만 변경되고 $L$은 변하지 않는다는 사실을 알고 있습니다 (Uniswap V3는 스왑이 하나의 가격 범위 내에서만 수행될 때 V2와 정확히 동일하게 작동합니다). 또한 다음을 알고 있습니다:

$$L = \frac{\Delta y}{\Delta \sqrt{P}}$$

그리고... 우리는 $\Delta y$를 알고 있습니다! 이것이 우리가 거래할 42 USDC입니다! 따라서, $L$이 주어졌을 때 42 USDC 판매가 현재 $\sqrt{P}$에 어떤 영향을 미칠지 알 수 있습니다:

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

Uniswap V3에서는 **우리가 거래를 통해 도달하고자 하는 가격**을 선택합니다 (스왑은 현재 가격을 변경, 즉 현재 가격을 곡선을 따라 이동시킨다는 것을 상기하십시오). 목표 가격을 알면, 컨트랙트는 우리로부터 가져와야 할 입력 토큰의 양과 그에 상응하는 출력 토큰의 양을 계산합니다.

위의 공식에 숫자를 대입해 보겠습니다:

$$\Delta \sqrt{P} = \frac{42 \enspace USDC}{1517882343751509868544} = 2192253463713690532467206957$$

이 값을 현재 $\sqrt{P}$에 더하면 목표 가격을 얻을 수 있습니다:

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

$$\sqrt{P_{target}} = 5604469350942327889444743441197}$$

> Python으로 목표 가격을 계산하려면:
> ```python
> amount_in = 42 * eth
> price_diff = (amount_in * q96) // liq
> price_next = sqrtp_cur + price_diff
> print("New price:", (price_next / q96) ** 2)
> print("New sqrtP:", price_next)
> print("New tick:", price_to_tick((price_next / q96) ** 2))
> # New price: 5003.913912782393
> # New sqrtP: 5604469350942327889444743441197
> # New tick: 85184
> ```

목표 가격을 찾은 후에는 이전 장에서 사용한 양 계산 함수를 사용하여 토큰 양을 계산할 수 있습니다:

$$ x = \frac{L(\sqrt{p_b}-\sqrt{p_a})}{\sqrt{p_b}\sqrt{p_a}}$$
$$ y = L(\sqrt{p_b}-\sqrt{p_a}) $$

> Python에서:
> ```python
> amount_in = calc_amount1(liq, price_next, sqrtp_cur)
> amount_out = calc_amount0(liq, price_next, sqrtp_cur)
>
> print("USDC in:", amount_in / eth)
> print("ETH out:", amount_out / eth)
> # USDC in: 42.0
> # ETH out: 0.008396714242162444
> ```

양을 검증하기 위해, 다른 공식을 떠올려 봅시다:

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

이 공식을 사용하여, 가격 변화 $\Delta\frac{1}{\sqrt{P}}$와 유동성 $L$을 알면 우리가 구매하는 ETH의 양 $\Delta x$를 찾을 수 있습니다. 주의하세요: $\Delta \frac{1}{\sqrt{P}}$는 $\frac{1}{\Delta \sqrt{P}}$가 아닙니다! 전자는 ETH 가격의 변화이고, 다음 표현식을 사용하여 찾을 수 있습니다:

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}$$

다행히도, 우리는 이미 모든 값을 알고 있으므로, 바로 대입할 수 있습니다 (화면에 다 들어가지 않을 수도 있습니다!):

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{5604469350942327889444743441197} - \frac{1}{5602277097478614198912276234240}$$

$$= -6.982190286589445\text{e-}35 * 2^{96} $$
$$= -0.00000553186106731426$$

이제 $\Delta x$를 구해봅시다:

$$\Delta x = -0.00000553186106731426 * 1517882343751509868544 = -8396714242162698 $$

이는 0.008396714242162698 ETH이며, 위에서 구한 값과 매우 유사합니다! 이 양은 풀에서 제거되므로 음수라는 점에 유의하십시오.

### 스왑 구현하기

스왑은 `swap` 함수에서 구현됩니다:
```solidity
function swap(address recipient)
    public
    returns (int256 amount0, int256 amount1)
{
    ...
```
현재로서는 토큰 수신자인 수령인만 인수로 받습니다.

먼저, 목표 가격과 틱을 찾고, 토큰 양을 계산해야 합니다. 다시 한번, 간단하게 유지하기 위해 이전에 계산한 값을 하드 코딩하겠습니다:
```solidity
...
int24 nextTick = 85184;
uint160 nextPrice = 5604469350942327889444743441197;

amount0 = -0.008396714242162444 ether;
amount1 = 42 ether;
...
```

다음으로, 거래는 현재 가격에 영향을 미치므로 현재 틱과 `sqrtP`를 업데이트해야 합니다:
```solidity
...
(slot0.tick, slot0.sqrtPriceX96) = (nextTick, nextPrice);
...
```

다음으로, 컨트랙트는 수령인에게 토큰을 보내고, 호출자가 입력 금액을 컨트랙트로 전송하도록 합니다:
```solidity
...
IERC20(token0).transfer(recipient, uint256(-amount0));

uint256 balance1Before = balance1();
IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
    amount0,
    amount1
);
if (balance1Before + uint256(amount1) < balance1())
    revert InsufficientInputAmount();
...
```

다시 한번, 콜백을 사용하여 제어를 호출자에게 전달하고 토큰을 전송하도록 합니다. 그 후, 풀의 잔액이 올바른지, 입력 금액이 포함되었는지 확인합니다.

마지막으로, 컨트랙트는 스왑을 검색 가능하게 만들기 위해 `Swap` 이벤트를 발생시킵니다. 이벤트에는 스왑에 대한 모든 정보가 포함됩니다:
```solidity
...
emit Swap(
    msg.sender,
    recipient,
    amount0,
    amount1,
    slot0.sqrtPriceX96,
    liquidity,
    slot0.tick
);
```

이것으로 끝입니다! 함수는 지정된 수령인 주소로 일정량의 토큰을 보내고, 그 대가로 다른 토큰을 특정 수량만큼 받기를 기대합니다. 이 책 전체를 통해, 이 함수는 훨씬 더 복잡해질 것입니다.

### 스왑 테스트하기

이제 스왑 함수를 테스트할 수 있습니다. 동일한 테스트 파일에서 `testSwapBuyEth` 함수를 만들고 테스트 케이스를 설정합니다. 이 테스트 케이스는 `testMintSuccess`와 동일한 매개변수를 사용합니다:
```solidity
function testSwapBuyEth() public {
    TestCaseParams memory params = TestCaseParams({
        wethBalance: 1 ether,
        usdcBalance: 5000 ether,
        currentTick: 85176,
        lowerTick: 84222,
        upperTick: 86129,
        liquidity: 1517882343751509868544,
        currentSqrtP: 5602277097478614198912276234240,
        shouldTransferInCallback: true,
        mintLiqudity: true
    });
    (uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);

    ...
```

그러나 다음 단계는 다를 것입니다.

> 유동성이 풀에 올바르게 추가되었는지 테스트하는 것은 다른 테스트 케이스에서 이미 테스트했으므로 여기서는 생략합니다.

테스트 스왑을 수행하려면 42 USDC가 필요합니다:
```solidity
token1.mint(address(this), 42 ether);
```

스왑을 수행하기 전에, 풀 컨트랙트가 요청할 때 토큰을 컨트랙트로 전송할 수 있는지 확인해야 합니다:
```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```
스왑 중 양은 양수 (풀로 보내는 양) 및 음수 (풀에서 가져오는 양)가 될 수 있으므로, 콜백에서 우리는 양수 양, 즉 우리가 거래하는 양만 보내고 싶습니다.

이제 `swap`을 호출할 수 있습니다:
```solidity
(int256 amount0Delta, int256 amount1Delta) = pool.swap(address(this));
```

함수는 스왑에 사용된 토큰 양을 반환하며, 즉시 확인할 수 있습니다:
```solidity
assertEq(amount0Delta, -0.008396714242162444 ether, "invalid ETH out");
assertEq(amount1Delta, 42 ether, "invalid USDC in");
```

그런 다음, 토큰이 호출자로부터 전송되었는지 확인해야 합니다:
```solidity
assertEq(
    token0.balanceOf(address(this)),
    uint256(userBalance0Before - amount0Delta),
    "invalid user ETH balance"
);
assertEq(
    token1.balanceOf(address(this)),
    0,
    "invalid user USDC balance"
);
```

그리고 풀 컨트랙트로 전송되었는지 확인합니다:
```solidity
assertEq(
    token0.balanceOf(address(pool)),
    uint256(int256(poolBalance0) + amount0Delta),
    "invalid pool ETH balance"
);
assertEq(
    token1.balanceOf(address(pool)),
    uint256(int256(poolBalance1) + amount1Delta),
    "invalid pool USDC balance"
);
```

마지막으로, 풀 상태가 올바르게 업데이트되었는지 확인합니다:
```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5604469350942327889444743441197,
    "invalid current sqrtP"
);
assertEq(tick, 85184, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

스왑은 현재 유동성을 변경하지 않습니다. 나중에 유동성이 변경되는 시점을 다루는 장에서 살펴보겠습니다.

### 숙제

`InsufficientInputAmount` 에러로 실패하는 테스트를 작성하세요. 숨겨진 버그가 있다는 것을 명심하세요 🙂