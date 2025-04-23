# 사용자 인터페이스

드디어 이번 마일스톤의 마지막 단계인 사용자 인터페이스 구축에 도달했습니다!



![UI 앱의 인터페이스](images/ui.png)

프론트엔드 앱을 구축하는 것은 이 책의 주요 목표가 아니므로, 앱을 처음부터 구축하는 방법은 보여드리지 않겠습니다. 대신, MetaMask를 사용하여 스마트 컨트랙트와 상호 작용하는 방법을 보여드리겠습니다.

> 앱을 직접 사용해보고 로컬에서 실행하고 싶다면, 코드 저장소의 [ui](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_1/ui) 폴더에서 펀딩할 수 있습니다. 이것은 간단한 React 앱이며, 로컬에서 실행하려면 `App.js`에서 컨트랙트 주소를 설정하고 `yarn start`를 실행하세요.


## 도구 개요

### MetaMask란 무엇인가요?

MetaMask는 브라우저 확장 프로그램으로 구현된 이더리움 지갑입니다. 개인 키를 생성 및 저장하고, 토큰 잔액을 표시하며, 다양한 네트워크에 연결하고, 이더와 토큰을 보내고 받을 수 있습니다. 즉, 지갑이 해야 하는 모든 기능을 수행합니다.

그 외에도 MetaMask는 서명자 및 제공자 역할을 합니다. 제공자로서 이더리움 노드에 연결하고 JSON-RPC API를 사용할 수 있는 인터페이스를 제공합니다. 서명자로서 안전한 트랜잭션 서명을 위한 인터페이스를 제공하므로, 지갑의 개인 키를 사용하여 모든 트랜잭션에 서명하는 데 사용할 수 있습니다.



![MetaMask 작동 방식](images/metamask.png)

### 편의 라이브러리

그러나 MetaMask는 많은 기능을 제공하지 않습니다. 계정 관리와 원시 트랜잭션 전송만 가능합니다. 컨트랙트와의 상호 작용을 쉽게 만들어 줄 또 다른 라이브러리가 필요합니다. 또한 EVM 특정 데이터 (ABI 인코딩/디코딩, 큰 숫자 처리 등)를 처리할 때 삶을 더 편하게 만들어 줄 유틸리티 세트도 원합니다.

이러한 라이브러리는 여러 가지가 있습니다. 가장 인기 있는 두 가지는 [web3.js](https://github.com/ChainSafe/web3.js)와 [ethers.js](https://github.com/ethers-io/ethers.js/)입니다. 둘 중 하나를 선택하는 것은 개인 취향의 문제입니다. 저에게는 Ethers.js가 더 깔끔한 컨트랙트 상호 작용 인터페이스를 가진 것 같아서 이것을 선택하겠습니다.

## 워크플로우

이제 MetaMask + Ethers.js를 사용하여 상호 작용 시나리오를 어떻게 구현할 수 있는지 살펴봅시다.

### 로컬 노드에 연결

트랜잭션을 보내고 블록체인 데이터를 가져오려면 MetaMask가 이더리움 노드에 연결해야 합니다. 컨트랙트와 상호 작용하려면 로컬 Anvil 노드에 연결해야 합니다. 이렇게 하려면 MetaMask를 열고 네트워크 목록을 클릭한 다음 "네트워크 추가"를 클릭하고 RPC URL `http://localhost:8545`로 네트워크를 추가하세요. 자동으로 체인 ID (Anvil의 경우 31337)를 감지합니다.

로컬 노드에 연결한 후 개인 키를 가져와야 합니다. MetaMask에서 주소 목록을 클릭하고 "계정 가져오기"를 클릭한 다음 컨트랙트를 배포하기 전에 선택한 주소의 개인 키를 붙여넣으세요. 그런 다음 자산 목록으로 이동하여 두 토큰의 주소를 가져옵니다. 이제 MetaMask에서 토큰 잔액이 표시됩니다.

> MetaMask는 여전히 버그가 좀 있습니다. 제가 겪었던 한 가지 문제는 `localhost`에 연결되었을 때 블록체인 상태를 캐시한다는 것입니다. 이 때문에 노드를 다시 시작할 때 이전 토큰 잔액과 상태가 표시될 수 있습니다. 이를 해결하려면 고급 설정으로 이동하여 "계정 재설정"을 클릭하세요. 노드를 다시 시작할 때마다 이 작업을 수행해야 합니다.

### MetaMask에 연결

모든 웹사이트가 MetaMask에서 주소에 접근할 수 있는 것은 아닙니다. 웹사이트는 먼저 MetaMask에 연결해야 합니다. 새 웹사이트가 MetaMask에 연결하려고 하면 권한을 묻는 창이 표시됩니다.

프론트엔드 앱에서 MetaMask에 연결하는 방법은 다음과 같습니다.
```js
// ui/src/contexts/MetaMask.js
const connect = () => {
  if (typeof (window.ethereum) === 'undefined') {
    return setStatus('not_installed');
  }

  Promise.all([
    window.ethereum.request({ method: 'eth_requestAccounts' }),
    window.ethereum.request({ method: 'eth_chainId' }),
  ]).then(function ([accounts, chainId]) {
    setAccount(accounts[0]);
    setChain(chainId);
    setStatus('connected');
  })
    .catch(function (error) {
      console.error(error)
    });
}
```

`window.ethereum`은 MetaMask에서 제공하는 객체이며, MetaMask와 통신하는 인터페이스입니다. 정의되지 않은 경우 MetaMask가 설치되지 않은 것입니다. 정의된 경우 MetaMask에 두 가지 요청 (`eth_requestAccounts` 및 `eth_chainId`)을 보낼 수 있습니다. 실제로 `eth_requestAccounts`는 웹사이트를 MetaMask에 연결합니다. MetaMask에서 주소를 쿼리하고 MetaMask는 사용자에게 권한을 요청합니다. 사용자는 접근 권한을 부여할 주소를 선택할 수 있습니다.

`eth_chainId`는 MetaMask가 연결된 노드의 체인 ID를 요청합니다. 주소와 체인 ID를 얻은 후에는 인터페이스에 표시하는 것이 좋습니다.



![MetaMask가 연결되었습니다.](images/ui_metamask_connected.png)

### 유동성 공급

풀에 유동성을 공급하려면 사용자가 예치하려는 금액을 입력하도록 요청하는 양식을 만들어야 합니다. "제출"을 클릭하면 앱은 관리자 컨트랙트에서 `mint`를 호출하고 사용자가 선택한 금액을 제공하는 트랜잭션을 구축합니다. 이것을 어떻게 하는지 살펴봅시다.

Ether.js는 컨트랙트와 상호 작용하기 위한 `Contract` 인터페이스를 제공합니다. 함수 매개변수를 인코딩하고, 유효한 트랜잭션을 만들고, MetaMask에 전달하는 작업을 수행하므로 삶이 훨씬 쉬워집니다. 저희에게 컨트랙트 호출은 JS 객체에서 비동기 메서드를 호출하는 것처럼 보입니다.

`Contracts` 인스턴스를 만드는 방법을 살펴봅시다.

```js
token0 = new ethers.Contract(
  props.config.token0Address,
  props.config.ABIs.ERC20,
  new ethers.providers.Web3Provider(window.ethereum).getSigner()
);
```

`Contract` 인스턴스는 주소와 이 주소에 배포된 컨트랙트의 ABI입니다. ABI는 컨트랙트와 상호 작용하는 데 필요합니다. 세 번째 매개변수는 MetaMask에서 제공하는 서명자 인터페이스입니다. JS 컨트랙트 인스턴스에서 MetaMask를 통해 트랜잭션에 서명하는 데 사용됩니다.

이제 풀에 유동성을 추가하는 함수를 추가해 보겠습니다.
```js
const addLiquidity = (account, { token0, token1, manager }, { managerAddress, poolAddress }) => {
  const amount0 = ethers.utils.parseEther("0.998976618347425280");
  const amount1 = ethers.utils.parseEther("5000"); // 5000 USDC
  const lowerTick = 84222;
  const upperTick = 86129;
  const liquidity = ethers.BigNumber.from("1517882343751509868544");
  const extra = ethers.utils.defaultAbiCoder.encode(
    ["address", "address", "address"],
    [token0.address, token1.address, account]
  );
  ...
```

가장 먼저 해야 할 일은 매개변수를 준비하는 것입니다. 이전과 동일한 값을 사용합니다.

다음으로 관리자 컨트랙트가 토큰을 가져갈 수 있도록 허용합니다. 먼저 현재 허용량을 확인합니다.
```js
Promise.all(
  [
    token0.allowance(account, managerAddress),
    token1.allowance(account, managerAddress)
  ]
)
```

그런 다음 둘 중 하나가 해당 양의 토큰을 전송하기에 충분한지 확인합니다. 그렇지 않은 경우 `approve` 트랜잭션을 보내 사용자에게 관리자 컨트랙트에 특정 금액을 지출하도록 승인하도록 요청합니다. 사용자가 전체 금액을 승인했는지 확인한 후 `manager.mint`를 호출하여 유동성을 추가합니다.
```js
.then(([allowance0, allowance1]) => {
  return Promise.resolve()
    .then(() => {
      if (allowance0.lt(amount0)) {
        return token0.approve(managerAddress, amount0).then(tx => tx.wait())
      }
    })
    .then(() => {
      if (allowance1.lt(amount1)) {
        return token1.approve(managerAddress, amount1).then(tx => tx.wait())
      }
    })
    .then(() => {
      return manager.mint(poolAddress, lowerTick, upperTick, liquidity, extra)
        .then(tx => tx.wait())
    })
    .then(() => {
      alert('유동성이 추가되었습니다!');
    });
})
```

> `lt`는 [BigNumber](https://docs.ethers.io/v5/api/utils/bignumber/)의 메서드입니다. Ethers.js는 JavaScript가 [정밀도가 충분하지 않기 때문에](https://docs.ethers.io/v5/api/utils/bignumber/#BigNumber--notes-safenumbers) `uint256` 유형을 나타내기 위해 BigNumber를 사용합니다. 이것이 편리한 라이브러리를 원하는 이유 중 하나입니다.

이것은 허용량 부분을 제외하고는 테스트 컨트랙트와 매우 유사합니다.

위 코드의 `token0`, `token1`, `manager`는 `Contract`의 인스턴스입니다. `approve`와 `mint`는 컨트랙트를 인스턴스화할 때 제공한 ABI에서 동적으로 생성된 컨트랙트 함수입니다. 이러한 메서드를 호출할 때 Ethers.js는 다음을 수행합니다.
1. 함수 매개변수를 인코딩합니다.
2. 트랜잭션을 구축합니다.
3. 트랜잭션을 MetaMask에 전달하고 서명을 요청합니다. 사용자는 MetaMask 창을 보고 "확인"을 누릅니다.
4. 트랜잭션을 MetaMask가 연결된 노드로 보냅니다.
5. 전송된 트랜잭션에 대한 모든 정보가 포함된 트랜잭션 객체를 반환합니다.

트랜잭션 객체에는 트랜잭션이 마이닝될 때까지 기다리는 데 사용하는 `wait` 함수도 포함되어 있습니다. 이를 통해 다른 트랜잭션을 보내기 전에 트랜잭션이 성공적으로 실행될 때까지 기다릴 수 있습니다.

> 이더리움은 트랜잭션의 엄격한 순서를 요구합니다. nonce를 기억하십니까? 이것은 이 계정에서 보낸 트랜잭션의 계정 전체 인덱스입니다. 모든 새 트랜잭션은 이 인덱스를 증가시키고, 이더리움은 이전 트랜잭션 (nonce가 더 작은 트랜잭션)이 마이닝될 때까지 트랜잭션을 마이닝하지 않습니다.

### 토큰 스왑

토큰을 스왑하려면 동일한 패턴을 사용합니다. 사용자로부터 매개변수를 가져오고, 허용량을 확인하고, 관리자에서 `swap`을 호출합니다.

```js
const swap = (amountIn, account, { tokenIn, manager, token0, token1 }, { managerAddress, poolAddress }) => {
  const amountInWei = ethers.utils.parseEther(amountIn);
  const extra = ethers.utils.defaultAbiCoder.encode(
    ["address", "address", "address"],
    [token0.address, token1.address, account]
  );

  tokenIn.allowance(account, managerAddress)
    .then((allowance) => {
      if (allowance.lt(amountInWei)) {
        return tokenIn.approve(managerAddress, amountInWei).then(tx => tx.wait())
      }
    })
    .then(() => {
      return manager.swap(poolAddress, extra).then(tx => tx.wait())
    })
    .then(() => {
      alert('스왑에 성공했습니다!');
    }).catch((err) => {
      console.error(err);
      alert('실패했습니다!');
    });
}
```

여기서 유일한 새로운 것은 숫자를 wei (이더리움의 최소 단위)로 변환하는 데 사용하는 `ethers.utils.parseEther()` 함수입니다.

### 변경 사항 구독

탈중앙화 애플리케이션의 경우 현재 블록체인 상태를 반영하는 것이 중요합니다. 예를 들어, 탈중앙화 거래소의 경우 현재 풀 준비금을 기반으로 스왑 가격을 적절하게 계산하는 것이 중요합니다. 오래된 데이터는 슬리피지를 유발하고 스왑 트랜잭션이 실패하게 만들 수 있습니다.

풀 컨트랙트를 개발하는 동안 블록체인 데이터 인덱스 역할을 하는 이벤트에 대해 배웠습니다. 스마트 컨트랙트 상태가 수정될 때마다 이벤트는 빠른 검색을 위해 인덱싱되므로 이벤트를 발생시키는 것이 좋습니다. 지금 하려는 것은 프론트엔드 앱을 최신 상태로 유지하기 위해 컨트랙트 이벤트를 구독하는 것입니다. 이벤트 피드를 구축해 봅시다!

이전에 권장한 대로 ABI 파일을 확인했다면 이벤트 이름과 필드에 대한 이벤트 설명도 보셨을 것입니다. [Ether.js는 이를 파싱하고](https://docs.ethers.io/v5/api/contract/contract/#Contract--events) 새 이벤트를 구독하는 인터페이스를 제공합니다. 이것이 어떻게 작동하는지 살펴봅시다.

이벤트를 구독하려면 `on(EVENT_NAME, handler)` 함수를 사용합니다. 콜백은 이벤트의 모든 필드와 이벤트 자체를 매개변수로 받습니다.
```js
const subscribeToEvents = (pool, callback) => {
  pool.on("Mint", (sender, owner, tickLower, tickUpper, amount, amount0, amount1, event) => callback(event));
  pool.on("Swap", (sender, recipient, amount0, amount1, sqrtPriceX96, liquidity, tick, event) => callback(event));
}
```

이전 이벤트를 필터링하고 가져오려면 `queryFilter`를 사용할 수 있습니다.
```js
Promise.all([
  pool.queryFilter("Mint", "earliest", "latest"),
  pool.queryFilter("Swap", "earliest", "latest"),
]).then(([mints, swaps]) => {
  ...
});
```

일부 이벤트 필드가 `indexed`로 표시된 것을 눈치채셨을 것입니다. 이러한 필드는 이더리움 노드에서 인덱싱되므로 이러한 필드의 특정 값으로 이벤트를 검색할 수 있습니다. 예를 들어, `Swap` 이벤트에는 `sender` 및 `recipient` 필드가 인덱싱되어 있으므로 스왑 보낸 사람 및 받는 사람별로 검색할 수 있습니다. 다시 말하지만, Ethers.js는 이를 더 쉽게 만듭니다.
```js
const swapFilter = pool.filters.Swap(sender, recipient);
const swaps = await pool.queryFilter(swapFilter, fromBlock, toBlock);
```

---

이것으로 끝입니다! 마일스톤 1을 완료했습니다!

<p style="font-size:3rem; text-align: center">
🎉🍾🍾🍾🎉
</p>