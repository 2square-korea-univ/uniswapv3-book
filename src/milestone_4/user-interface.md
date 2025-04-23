# 사용자 인터페이스

스왑 경로를 도입한 후, 저희는 웹 앱의 내부를 상당히 단순화할 수 있습니다. 우선, 모든 스왑은 이제 경로를 사용하는데, 경로는 여러 풀을 포함할 필요가 없기 때문입니다. 둘째, 스왑의 방향을 변경하는 것이 더 쉬워졌습니다. 단순히 경로를 반전시키면 됩니다. 그리고 `CREATE2`를 통한 통합된 풀 주소 생성과 고유한 솔트 덕분에, 더 이상 풀 주소를 저장하고 토큰 순서에 대해 신경 쓸 필요가 없습니다.

하지만, 하나의 중요한 알고리즘을 추가하지 않고는 웹 앱에 멀티풀 스왑을 통합할 수 없습니다. 스스로에게 질문해 보세요: "풀이 없는 두 토큰 사이에 경로를 어떻게 찾을 수 있을까요?"

## AutoRouter

Uniswap은 *AutoRouter*라고 불리는 알고리즘을 구현하는데, 이는 두 토큰 사이의 최단 경로를 찾는 알고리즘입니다. 더욱이, 최고의 평균 환율을 찾기 위해 하나의 지불을 여러 개의 작은 지불로 분할하기도 합니다. 이익은 [분할되지 않은 거래에 비해 36.84%까지 커질 수 있습니다](https://uniswap.org/blog/auto-router-v2). 이는 매우 훌륭하게 들리지만, 저희는 그렇게 고도화된 알고리즘을 구축하지는 않을 것입니다. 대신, 저희는 더 간단한 것을 구축할 것입니다.

## 간단한 라우터 설계

가정해 봅시다, 저희에게 다음과 같이 많은 풀들이 있다고 합시다:



![흩어진 풀들](images/pools_scattered.png)

이런 혼란 속에서 두 토큰 사이의 최단 경로를 어떻게 찾을 수 있을까요?

이러한 종류의 작업에 가장 적합한 해결책은 *그래프*에 기반합니다. 그래프는 노드(어떤 것을 나타내는 객체)와 에지(노드를 연결하는 링크)로 구성된 데이터 구조입니다. 저희는 풀들의 혼란을 그래프로 바꿀 수 있습니다. 여기서 각 노드는 (풀을 가진) 토큰이고 각 에지는 이 토큰이 속한 풀입니다. 따라서 그래프로 표현된 풀은 에지로 연결된 두 노드입니다. 위의 풀들은 다음 그래프가 됩니다:



![풀 그래프](images/pools_graph.png)

그래프가 저희에게 주는 가장 큰 이점은 노드에서 다른 노드로 이동하여 경로를 찾을 수 있는 능력입니다. 특히, 저희는 [A* 탐색 알고리즘](https://en.wikipedia.org/wiki/A*_search_algorithm)을 사용할 것입니다. 알고리즘이 어떻게 작동하는지 자유롭게 알아보세요. 하지만 저희 앱에서는 삶을 더 편하게 만들기 위해 라이브러리를 사용할 것입니다. 저희가 사용할 라이브러리 세트는 그래프를 구축하기 위한 [ngraph.ngraph](https://github.com/anvaka/ngraph.graph)와 경로를 찾기 위한 [ngraph.path](https://github.com/anvaka/ngraph.path)입니다 (A* 탐색 알고리즘과 다른 몇 가지 알고리즘을 구현하는 것은 후자입니다).

UI 앱에서, 경로 탐색기를 만들어 봅시다. 이것은 클래스가 될 것이며, 인스턴스화될 때, 쌍 목록을 그래프로 변환하여 나중에 그래프를 사용하여 두 토큰 사이의 최단 경로를 찾을 것입니다.
```javascript
import createGraph from 'ngraph.graph';
import path from 'ngraph.path';

class PathFinder {
  constructor(pairs) {
    this.graph = createGraph();

    pairs.forEach((pair) => {
      this.graph.addNode(pair.token0.address);
      this.graph.addNode(pair.token1.address);
      this.graph.addLink(pair.token0.address, pair.token1.address, pair.tickSpacing);
      this.graph.addLink(pair.token1.address, pair.token0.address, pair.tickSpacing);
    });

    this.finder = path.aStar(this.graph);
  }

  ...
```

생성자에서, 저희는 빈 그래프를 만들고 링크된 노드로 채우고 있습니다. 각 노드는 토큰 주소이고 링크는 연결된 데이터를 가지고 있습니다. 이는 틱 간격입니다. 저희는 A*가 찾은 경로에서 이 정보를 추출할 수 있을 것입니다. 그래프를 초기화한 후, 저희는 A* 알고리즘 구현을 인스턴스화합니다.

다음으로, 토큰 사이의 경로를 찾고 그것을 토큰 주소와 틱 간격의 배열로 변환하는 함수를 구현해야 합니다:

```javascript
findPath(fromToken, toToken) {
  return this.finder.find(fromToken, toToken).reduce((acc, node, i, orig) => {
    if (acc.length > 0) {
      acc.push(this.graph.getLink(orig[i - 1].id, node.id).data);
    }

    acc.push(node.id);

    return acc;
  }, []).reverse();
}
```

`this.finder.find(fromToken, toToken)`은 노드 목록을 반환하며, 불행히도 그들 사이의 에지에 대한 정보는 포함하지 않습니다 (저희는 틱 간격을 에지에 저장합니다). 따라서, 저희는 에지를 찾기 위해 `this.graph.getLink(previousNode, currentNode)`를 호출하고 있습니다.

이제, 사용자가 입력 또는 출력 토큰을 변경할 때마다, 저희는 새 경로를 구축하기 위해 `pathFinder.findPath(token0, token1)`을 호출할 수 있습니다.