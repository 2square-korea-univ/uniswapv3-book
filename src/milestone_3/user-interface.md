## 사용자 인터페이스

이번 개발 단계를 통해 변경된 사항들을 사용자 인터페이스에 반영할 준비가 되었습니다. 두 가지 새로운 기능을 추가할 것입니다:

1. 유동성 공급 대화 상자 창;
2. 스왑 시 슬리피지 허용 오차.

## 유동성 공급 대화 상자



![유동성 공급 대화 상자 창](images/add_liquidity_dialog.png)

이 변경 사항을 통해 코드에서 하드 코딩된 유동성 수량을 제거하고 임의의 범위에서 유동성을 공급할 수 있게 됩니다.

이 대화 상자는 몇 개의 입력 필드를 가진 간단한 컴포넌트입니다. 이전 구현에서 사용했던 `addLiquidity` 함수를 재사용할 수도 있습니다. 하지만 이제 가격을 JavaScript에서 틱 인덱스로 변환해야 합니다. 사용자는 가격을 입력하길 원하지만 컨트랙트는 틱을 요구하기 때문입니다. 작업을 더 쉽게 하기 위해 공식 Uniswap V3 SDK([https://github.com/Uniswap/v3-sdk/](https://github.com/Uniswap/v3-sdk/))를 사용할 것입니다.

가격을 $\sqrt{P}$로 변환하려면 [encodeSqrtRatioX96](https://github.com/Uniswap/v3-sdk/blob/08a7c050cba00377843497030f502c05982b1c43/src/utils/encodeSqrtRatioX96.ts) 함수를 사용할 수 있습니다. 이 함수는 두 개의 수량을 입력으로 받아 하나를 다른 하나로 나누어 가격을 계산합니다. 가격을 $\sqrt{P}$로 변환하기만 하면 되므로 `amount0`에 1을 전달할 수 있습니다:

```javascript
const priceToSqrtP = (price) => encodeSqrtRatioX96(price, 1);
```

가격을 틱 인덱스로 변환하려면 [TickMath.getTickAtSqrtRatio](https://github.com/Uniswap/v3-sdk/blob/08a7c050cba00377843497030f502c05982b1c43/src/utils/tickMath.ts#L82) 함수를 사용할 수 있습니다. 이것은 JavaScript로 구현된 Solidity TickMath 라이브러리입니다:

```javascript
const priceToTick = (price) => TickMath.getTickAtSqrtRatio(priceToSqrtP(price));
```

이제 사용자가 입력한 가격을 틱으로 변환할 수 있습니다:

```javascript
const lowerTick = priceToTick(lowerPrice);
const upperTick = priceToTick(upperPrice);
```

여기서 추가해야 할 또 다른 사항은 슬리피지 보호입니다. 단순성을 위해 하드 코딩된 값으로 설정하고 0.5%로 지정했습니다. 다음은 슬리피지 허용 오차를 사용하여 최소 수량을 계산하는 방법입니다:

```javascript
const slippage = 0.5;
const amount0Desired = ethers.utils.parseEther(amount0);
const amount1Desired = ethers.utils.parseEther(amount1);
const amount0Min = amount0Desired.mul((100 - slippage) * 100).div(10000);
const amount1Min = amount1Desired.mul((100 - slippage) * 100).div(10000);
```

## 스왑 시 슬리피지 허용 오차

우리는 애플리케이션의 유일한 사용자이므로 개발 중에 슬리피지 문제가 발생하지 않겠지만, 스왑 중에 슬리피지 허용 오차를 제어할 수 있는 입력 필드를 추가해 보겠습니다.



![웹 앱의 메인 화면](images/slippage_tolerance.png)

스왑 시 슬리피지 보호는 제한 가격(스왑 중에 넘거나 내려가지 않기를 원하는 가격)을 통해 구현됩니다. 즉, 스왑 트랜잭션을 보내기 전에 이 가격을 알아야 합니다. 하지만 프론트엔드에서 계산할 필요는 없습니다. Quoter 컨트랙트가 이를 대신 처리해주기 때문입니다:

```solidity
function quote(QuoteParams memory params)
    public
    returns (
        uint256 amountOut,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    ) { ... }
```

그리고 스왑 수량을 계산하기 위해 Quoter를 호출하고 있습니다.

따라서 제한 가격을 계산하려면 `sqrtPriceX96After`를 가져와서 슬리피지 허용 오차를 빼야 합니다. 이것이 스왑 중에 내려가지 않기를 원하는 가격이 됩니다.

```solidity
const limitPrice = priceAfter.mul((100 - parseFloat(slippage)) * 100).div(10000);
```

이것으로 완료입니다!