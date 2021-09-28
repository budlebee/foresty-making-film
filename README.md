# Multi-window 노트 테이킹 서비스 만들기

https://foresty.net

![nutshell](https://user-images.githubusercontent.com/33917337/112747527-0513b500-8ff1-11eb-8288-a2ec4d1b07f3.gif)

## 기능명세

1. Tree Graph 형식의 구조. 마인드맵처럼 캔버스 위에 문서(노드)를 생성할 수 있어야 합니다.
   ![node](https://user-images.githubusercontent.com/33917337/135022238-c194fa21-d4b2-4d58-81b9-3307142116a3.gif)
2. 클릭앤 드래그로 노드끼리 연결할 수 있어야 됩니다.
   ![drag](https://user-images.githubusercontent.com/33917337/135022267-5236083b-2b18-4794-9631-4da862360ea1.gif)
3. 노드의 위치를 변경할 수 있어야 합니다.
   ![position](https://user-images.githubusercontent.com/33917337/135022294-c012bb18-1de3-49fd-96d9-96659b6f571a.gif)
4. 여러개의 문서창을 동시에 띄워놓고 문서창의 위치를 변경할 수 있어야 됩니다.
   ![window](https://user-images.githubusercontent.com/33917337/135022312-84031ed6-8faa-4c3f-9455-17ab6a730a24.gif)

## 구현과정

- 기본적인 UI 와 상태관리는 리액트를 사용했습니다.
- 문서 데이터에 따라 트리 그래프를 렌더링하는 것은 d3js 를 사용했습니다.

### 1. 마인드맵처럼 캔버스 위에 문서(노드)를 생성할 수 있어야 합니다.

![node](https://user-images.githubusercontent.com/33917337/135022238-c194fa21-d4b2-4d58-81b9-3307142116a3.gif)

- 데이터 시각화 라이브러리인 d3js 를 사용해서 svg dom 에 이벤트리스너("dblclick")을 붙이고, 사용자가 더블클릭하면 노드가 생성되게 만들었습니다. 노드가 생성되면 redux 를 통해 상태를 갱신하고, 갱신된 상태에 따라 svg 노드를 렌더링 합니다.

```javascript
// 더블클릭시 노드 생성
svg.on("dblclick", async (d) => {
  const offsetElement = document.getElementById("treeContainer");
  const clientRect = offsetElement.getBoundingClientRect();

  const ratioFactor = width / clientRect.width; // 브라우저 창크기가 줄어들면 노드 위치도 재조정되게끔 하기 위한 비율 팩터.
  const createdNode = {
    id: `node${uid(20)}`,
    name: "New Node",
    x: d3.event.offsetX * ratioFactor,
    y: d3.event.offsetY * ratioFactor,
    radius: nodeRadius,
    body: "New Document",
    hashtags: [],
    fillColor: "#51cf66",
    parentNodeID: [],
    childNodeID: [],
  };
  nodeList = [...nodeList, createdNode];
  await reduxStore.dispatch(createNode(treeID, nodeList));
  changeTreeInfo();
  initNode();
});
```

```javascript
// d3js 를 이용해 svg circle 을 렌더링. nodeList 라는 데이터에 따라 dom attribute 를 설정할 수 있습니다.
function initNode() {
  nodeGroup
    .selectAll("circle")
    .data(nodeList)
    .join("circle")
    .style("fill", (d) => d.fillColor)
    .attr("cx", (d) => {
      return d.x;
    })
    .attr("cy", (d) => {
      return d.y;
    });
}
```

> #### Trouble & Answer
>
> 렌더링된 svg 노드들의 위치와 크기가 반응형으로 움직여야 합니다. getBoundingClientRect() 함수를 이용해서 dom 의 크기를 가져온뒤, 브라우저의 windowSize 와의 비율 width / clientRect.width 를 렌더링때 반영하는 것으로 해결했습니다.

### 2. 클릭앤 드래그로 노드끼리 연결할 수 있어야 됩니다. && 3. 노드의 위치를 변경할 수 있어야 합니다.

![drag](https://user-images.githubusercontent.com/33917337/135022267-5236083b-2b18-4794-9631-4da862360ea1.gif)
![position](https://user-images.githubusercontent.com/33917337/135022294-c012bb18-1de3-49fd-96d9-96659b6f571a.gif)

d3js 의 .drag() 함수를 사용했습니다. 사용자가 드래그를 시작할때, 드래그중일때, 드래그가 끝났을때의 동작을 지정할 수 있습니다.

```javascript
createdNodeGroup.call(
  d3
    .drag()
    .on("start", (node) => {
      d3.select(this).raise().classed("active", true);
    })
    .on("drag", (node) => {
      const newLinkList = linkList.map((link) => {
        if (link.startNodeID === node.id) {
          return { ...link, startX: d3.event.x, startY: d3.event.y };
        } else if (link.endNodeID === node.id) {
          return { ...link, endX: d3.event.x, endY: d3.event.y };
        } else {
          return link;
        }
      });
      linkList = newLinkList;
      initLink();
      d3.select(this).attr("cx", d3.event.x).attr("cy", d3.event.y);
      node.x = d3.event.x;
      node.y = d3.event.y;
      initNode();
      initLabel();
    })
    .on("end", async (node) => {
      d3.select(this).classed("active", false);
      node.x = d3.event.x;
      node.y = d3.event.y;

      await reduxStore.dispatch(createLink(treeID, linkList));
      await reduxStore.dispatch(createNode(treeID, nodeList)); // 노드의 위치가 바뀌었으면 리덕스를 통해 state를 갱신합니다.
      changeTreeInfo();
    })
);
```

#### 4. 여러개의 문서창을 동시에 띄워놓고 문서창의 위치를 변경할 수 있어야 됩니다.

![window](https://user-images.githubusercontent.com/33917337/135022312-84031ed6-8faa-4c3f-9455-17ab6a730a24.gif)

문서창을 여러개 띄워놓는 기능이 있을때 리액트로 상태관리를 어떻게 해야할지가 관건이었습니다.

- 당장 생각나는 방법은 문서창 정보를 담고 있는 배열을 하나 만들고, 문서창이 새로 생성될때마다 배열을 갱신하는 것이었습니다.
- 이런 방식을 채택하면, 사용자가 여러개의 문서창을 띄워놓은 상태에서 하나의 문서창만 수정해도 다른 문서창들이 영향을 받는다는 문제가 있었습니다.
- 이런 문제점은 노드를 클릭했을때 ReactDOM.render 함수를 호출해서 새로운 react dom 을 생성하는 것으로 해결했습니다.
- 사용자가 노드를 클릭해서 문서창을 열때마다, 새로운 react dom 을 생성하면 각 문서창의 state 들이 독립적으로 작동할 수 있습니다.

```javascript
ReactDOM.render(
  <SelectedNodeModal defaultZ={max + 1} node={node} />,
  document.getElementById(d.id)
);
```

문서창의 위치를 바꿀 수 있는 기능도 문제가 있었습니다.

- 운영체제의 창(window)처럼 문서창 상단을 클릭한채 움직이면 문서창의 위치가 마우스를 따라가게 만들고 싶은데, js ondrag 이벤트는 원하는 내대로 작동하지 않았습니다.
- 문서창의 상단바에만 ondrag 이벤트에 반응하게끔 해뒀는데, 사용자가 마우스를 빠르게 움직이면 문서창이 상단바를 따라가지 못하는 문제가 생겼습니다.
- 2, 3번 기능인 노드끼리의 드래그 연결을 위해 drag 이벤트를 캔버스에 할당한 것이 문제라고 추측하고, ondrag 이벤트 대신 mousemove 이벤트를 사용하는 것으로 해결했습니다.

```javascript
const modalID = `domInModal${node.id}`;
document.addEventListener("mousemove", function (e) {
  // onDrag 는 사용자가 마우스를 클릭중인지 아닌지를 체크하는 bool 변수입니다. mousedown 시 true, mouseup 시 false 가 됩니다.
  if (onDrag) {
    const modal = document.getElementById(modalID);
    const scrolledLeftLength = window.pageXOffset;
    const scrolledTopLength = window.pageYOffset;
    modal.style.left = e.clientX - relativeX + scrolledLeftLength + "px";
    modal.style.top = e.clientY - relativeY + scrolledTopLength + "px";
  }
});
document.addEventListener("mouseup", () => {
  onDrag = false;
  relativeX = 0;
  relativeY = 0;
});
```

- 문서창이 여러개 떠있으면 z index 에 따라서 배치되는데, 생성될때마다 기존에 생성된 문서창들의 zindex 값들을 읽고 그중 가장 큰값+1 의 zindex 가지게끔 생성했습니다.

```javascript
reduxStore.dispatch(selectNode(node));
const d = document.createElement("div");
d.id = `${node.id}`;
d.className = "nodeModalDOM";
document.getElementById("root").appendChild(d);
// 현재 만들어진 문서창을 가져오고
const modalList = document.getElementsByClassName("nodeModal");
// 문서창의 zindex 중 가장 큰 값을 가져오고
const zIndexList = Array.from(modalList).map((ele) => {
  return ele.style.zIndex;
});
const max = Math.max(...zIndexList);
// 가장 큰 zindex 값+1 만큼의 zindex를 가진 문서창을 생성합니다.
ReactDOM.render(
  <SelectedNodeModal defaultZ={max + 1} node={node} />,
  document.getElementById(d.id)
);
```

- 사용자가 여러개의 문서창을 띄워놓았을때, 아래에 가려진 문서창을 클릭하면 해당 문서창을 최상단으로 끌어올리기 위해 zindex 값을 재조정할 필요가 있었습니다.

```javascript
// 사용자가 문서창을 클릭할때마다, 해당 문서창의 z index 값을 갱신하는 함수
function changeZIndex(e) {
  if (e.target) {
    const modalList = document.getElementsByClassName("nodeModal");
    const zIndexList = Array.from(modalList).map((ele) => {
      return ele.style.zIndex;
    });
    const max = Math.max(...zIndexList);
    const modalDOM = document.getElementById(modalID);
    if (modalDOM) {
      modalDOM.style.zIndex = max + 2;
    }
  }
}
```

## 배운 것 : 테스트 코드를 쓰는 습관을 만들자

- dom 요소들의 좌표를 제어하는 것(offsetX, windowSize, clientX 등 여러 상대좌표, 절대좌표 요소들을 다뤘습니다)
- zindex 와 관련해 웹브라우저의 stacking order 에 대해 배웠습니다.
- 함수는 한번에 한가지 기능만 수행해야 했음에도, 리팩토링의 어려움때문에 같은 시점에 수행될 기능들을 하나의 함수에 전부 집어넣었습니다. 하나의 파일내의 코드가 매우 길어져서 가독성이 떨어지고, 다른 컴포넌트에서 같은 기능의 코드를 반복해서 작성해야됐습니다.
- API request, response 스펙을 작성하듯, 작성할 함수의 input, output 스펙 또는 호출이유, 호출결과 등을 먼저 작성하고 코드를 작성하면 더 좋은 결과가 나왔을것 같습니다.
- 테스트 코드를 한번도 작성해보지 않았는데, 적어도 테스트 코드를 작성하려고 시도하도 했다면, 함수를 더 잘게 쪼개고, 스펙 변경에도 유연하게 대처할 수 있는 코드를 작성할 수 있었을 것.
