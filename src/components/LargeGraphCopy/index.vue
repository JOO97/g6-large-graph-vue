<template>
  <div :style="{ height, width }" ref="wrapper">
    <CanvasMenu
      :edgeLabelVisible="edgeLabelVisible"
      :enableSearch="enableSearch"
      :enableSelectPathEnd="enableSelectPathEnd"
      @update-props="setCanvasMenuProps"
      @on-search-node="handleSearchNode"
      @on-fit-view="handleFitView"
      @on-find-path="handleFindPath"
    />
    <div :id="id" />
    <!-- test -->
    <!-- <div
      class="g6-component-tooltip"
      style="position: absolute;bottom: 20px;left:20px"
    >
      <h4>
        <span
          style="display:inline-block;margin-right:4px;border-radius:10px;width:10px;height:10px;background-color:#5581D6;"
        ></span>
        Custom Content
      </h4>
      <ul>
        <li>
          <span>id</span>
          <span
            >type Custom Content Custom Content Custom Content Custom Content
            Custom Content Custom Content Custom Content Custom Content</span
          >
        </li>
        <li><span>id</span> <span>type</span></li>
      </ul>
    </div> -->
  </div>
</template>

<script>
//测试数据
import graphData from "./data.json";

//引入g6 定义节点、边
import G6 from "./g6";
//相关方法
import {
  processNodesEdges,
  getForceLayoutConfig,
  manageExpandCollapseArray,
  getMixedGraph,
  cacheNodePositions,
  getNeighborMixedGraph,
  labelFormatter,
  labelMaxLength,
  getExtractNodeMixedGraph
} from "./utils";

//样式插入组件
import insertCss from "insert-css";

//CanvasMenu组件
import CanvasMenu from "../CanvasMenu/index.vue";

//算法
const { labelPropagation, louvain, findShortestPath } = G6.Algorithm;

const { uniqueId } = G6.Util;

// insert-css 引入自定义样式
insertCss(`
  .g6-component-contextmenu {
    position: absolute;
    z-index: 2;
    list-style-type: none;
    background-color: #363b40;
    border-radius: 6px;
    font-size: 14px;
    color: hsla(0,0%,100%,.85);
    width: fit-content;
    transition: opacity .2s;
    text-align: center;
    padding: 0px 20px 0px 20px;
		box-shadow: 0 5px 18px 0 rgba(0, 0, 0, 0.6);
		border: 0px;
  }
  .g6-component-contextmenu ul {
		padding-left: 0px;
		margin: 0;
  }
  .g6-component-contextmenu li {
    cursor: pointer;
    list-style-type: none;
    list-style: none;
    margin-left: 0;
    line-height: 38px;
  }
  .g6-component-contextmenu li:hover {
    color: #aaaaaa;
	}
`);

const DEFAULTNODESIZE = 20;

let currentUnproccessedData = { nodes: [], edges: [] };
let nodeMap = {};
let aggregatedNodeMap = {};
let hiddenItemIds = []; // 隐藏的元素 id 数组
let largeGraphMode = true;
let cachePositions = {};
let manipulatePosition = undefined;

let layout = {
  type: "",
  instance: null,
  destroyed: true
};
let expandArray = [];
let collapseArray = [];
let shiftKeydown = false;
let CANVAS_WIDTH = 800,
  CANVAS_HEIGHT = 800;

const darkBackColor = "rgb(43,47,51)";
const disableColor = "#777";
const theme = "dark";
const subjectColors = [
  "#5F95FF", // blue
  "#61DDAA",
  "#65789B",
  "#F6BD16",
  "#7262FD",
  "#78D3F8",
  "#9661BC",
  "#F6903D",
  "#008685",
  "#F08BB4"
];

const colorSets = G6.Util.getColorSetsBySubjectColors(
  subjectColors,
  darkBackColor,
  theme,
  disableColor
);

//tooltip
const tooltip = new G6.Tooltip({
  offsetX: 10,
  offsetY: 10,
  shouldBegin: e => {
    const model = e.item.getModel();
    const { type, isReal } = e.item.getModel();
    if ((type === "custom-quadratic" || type === "custom-line") && !isReal)
      return false;
    return true;
  },
  // 自定义 tooltip 内容
  getContent: e => {
    const model = e.item.getModel();
    const { type, label, count, colorSet, oriLabel, id } = e.item.getModel();
    let itemBgColor = "",
      itemLabel = "",
      infoArr = [];
    switch (type) {
      case "aggregated-node":
        itemBgColor = colorSet.activeStroke;
        itemLabel = "聚合节点";
        infoArr.push({
          key: "count",
          value: count + " 个节点"
        });
        break;
      case "real-node":
        itemBgColor = colorSet.activeStroke;
        itemLabel = label;
        if (model.customInfo) {
          const keys = Object.keys(model.customInfo);
          keys.map(key => {
            infoArr.push({
              key,
              value: model.customInfo[key]
            });
          });
        } else {
          infoArr.push({
            key: "id",
            value: id
          });
        }
        break;
      case "custom-line":
      case "custom-quadratic":
      case "loop":
        itemBgColor = "#6DD400";
        if (model.isReal) {
          itemLabel = label || oriLabel || "";
          const { source, target, customInfo } = model;
          if (customInfo) {
            const keys = Object.keys(customInfo);
            keys.map(key => {
              infoArr.push({
                key,
                value: model.customInfo[key]
              });
            });
          }
          infoArr.push(
            {
              key: "source",
              value: source
            },
            {
              key: "target",
              value: target
            }
          );
        } else {
          if (type === "loop") {
            itemLabel = "聚合边";
            infoArr.push({
              key: "count",
              value: model.count + " 条边"
            });
          }
        }
        break;
      default:
        break;
    }
    let innerHTML = `<h4>
      <span style="display:inline-block;margin-right:4px;border-radius:5px;width:10px;height:10px;background-color:${itemBgColor};"></span>
      ${itemLabel}
    </h4>`;
    infoArr.map((item, index) => {
      innerHTML += `
        ${index === 0 ? "<ul>" : ""}
          <li>
            <span>${item.key}</span>
            <span>${item.value}</span>
          </li>
        ${index === infoArr.length - 1 ? "<ul />" : ""}
      `;
    });
    return innerHTML;
  }
});

export default {
  props: {
    data: {
      type: Object,
      default: () => {
        return {
          nodes: [
            {
              id: "模型A1",
              customInfo: {
                id: "xx",
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              id: "模型A2",
              customInfo: {
                id: "xx",
                desc: "This website stores."
              }
            },
            {
              id: "模型A3",
              customInfo: {
                id: "xx",
                desc: "This."
              }
            },
            {
              id: "模型B1",
              customInfo: {
                id: "xx",
                desc: "This website stores cookies on your computer."
              }
            },
            {
              id: "模型B2",
              customInfo: {
                id: "xx",
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              id: "模型B3",
              customInfo: {
                id: "xx",
                desc: "remember you."
              }
            },
            {
              id: "模型B4",
              customInfo: {
                id: "xx",
                desc: "stores"
              }
            },
            {
              id: "模型B5",
              customInfo: {
                id: "xx",
                desc: "."
              }
            },
            {
              id: "模型B6",
              customInfo: {
                id: "xx",
                desc: "information."
              }
            },
            {
              id: "模型B7",
              customInfo: {
                id: "xx",
                desc: "Policy."
              }
            }
          ],
          edges: [
            {
              source: "模型A2",
              target: "模型B7",
              value: 1,
              customLabel: "l1",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型A1",
              target: "模型A1",
              value: 1,
              customLabel: "l2",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },

            {
              source: "模型A1",
              target: "模型A3",
              value: 1,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型B1",
              target: "模型A1",
              value: 2,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型A1",
              target: "模型B1",
              value: 20,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型A3",
              target: "模型B1",
              value: 20,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型B2",
              target: "模型B1",
              value: 20,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型B3",
              target: "模型B1",
              value: 20,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型B4",
              target: "模型B1",
              value: 20,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型B5",
              target: "模型B1",
              value: 20,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            },
            {
              source: "模型B6",
              target: "模型B1",
              value: 20,
              customLabel: "l3",
              customInfo: {
                desc:
                  "These cookies are used to collect information about how you interact with our website and allow us to remember you."
              }
            }
          ]
        };
      }
    },
    id: {
      type: String,
      default: "container"
    },
    height: {
      type: String,
      required: false,
      default: "100vh"
    },
    width: {
      type: String,
      required: false,
      default: "100%"
    }
  },
  components: {
    CanvasMenu
  },
  data() {
    return {
      graph: null,
      clusteredData: "",
      edgeLabelVisible: false,
      enableSearch: false,
      enableSelectPathEnd: false
    };
  },
  watch: {
    edgeLabelVisible(nVal) {
      const { graph } = this;
      if (!graph || graph.get("destroyed")) return;
      graph.getEdges().forEach(edge => {
        const oriLabel = edge.getModel().oriLabel;
        edge.update({
          label: nVal ? labelFormatter(oriLabel, labelMaxLength) : ""
        });
      });
    },
    enableSearch(nVal) {}
  },
  mounted() {
    this.init();
  },
  methods: {
    init() {
      const { id, data } = this;
      const container = document.getElementById(id);
      container.style.backgroundColor = "#2b2f33";
      const wrapper = this.$refs.wrapper.getBoundingClientRect();
      CANVAS_WIDTH = wrapper.width;
      CANVAS_HEIGHT = wrapper.height;
      nodeMap = {};
      const clusteredData = louvain(data, false);
      const aggregatedData = { nodes: [], edges: [] };
      clusteredData.clusters.forEach((cluster, i) => {
        cluster.nodes.forEach(node => {
          node.level = 0;
          node.label = node.id;
          node.type = "";
          node.colorSet = colorSets[i];
          data.nodes.map(item => {
            const { id, customInfo } = item;
            if (id === node.id && customInfo) {
              node.customInfo = customInfo;
            }
          });
          nodeMap[node.id] = node;
        });
        const cnode = {
          id: cluster.id,
          type: "aggregated-node",
          count: cluster.nodes.length,
          //   category: cluster.category,
          level: 1,
          label: cluster.id,
          colorSet: colorSets[i],
          idx: i
        };
        aggregatedNodeMap[cluster.id] = cnode;
        aggregatedData.nodes.push(cnode);
      });
      clusteredData.clusterEdges.forEach(clusterEdge => {
        const cedge = {
          ...clusterEdge,
          size: Math.log(clusterEdge.count),
          label: "",
          id: `edge-${uniqueId("cedge")}`
        };
        if (cedge.source === cedge.target) {
          cedge.type = "loop";
          cedge.loopCfg = {
            dist: 20
          };
        } else cedge.type = "line";
        aggregatedData.edges.push(cedge);
      });
      this.clusteredData = clusteredData;
      const { clusters } = clusteredData;
      const cNode = [];
      clusters.map(item => cNode.push(...item.nodes));
      const { edgeLabelVisible } = this;
      data.edges.forEach(edge => {
        edge.label = "";
        edge.oriLabel = edge.customLabel
          ? edge.customLabel
          : `${edge.source}-${edge.target}`;
        edge.id = `edge-${uniqueId("edge")}`;
        edge.style = { endArrow: true };
      });
      currentUnproccessedData = aggregatedData;
      const { edges: processedEdges } = processNodesEdges(
        currentUnproccessedData.nodes,
        currentUnproccessedData.edges,
        CANVAS_WIDTH,
        CANVAS_HEIGHT,
        largeGraphMode,
        edgeLabelVisible,
        true,
        cachePositions
      );
      //create graph
      this.graph = new G6.Graph({
        container: id,
        width: CANVAS_WIDTH,
        height: CANVAS_HEIGHT,
        linkCenter: true,
        minZoom: 0.1,
        groupByTypes: true,
        modes: {
          default: [
            {
              type: "drag-canvas",
              enableOptimize: true
            },
            {
              type: "zoom-canvas",
              enableOptimize: true,
              optimizeZoom: 0.01
            },
            "drag-node",
            "shortcuts-call"
            // 'create-edge'
          ],
          lassoSelect: [
            {
              type: "zoom-canvas",
              enableOptimize: true,
              optimizeZoom: 0.01
            },
            {
              type: "lasso-select",
              selectedState: "focus",
              trigger: "drag"
            }
          ],
          fisheyeMode: []
        },
        defaultNode: {
          type: "aggregated-node",
          size: DEFAULTNODESIZE
        },
        plugins: [this.setContextMenu(clusteredData), tooltip]
      });
      this.graph.get("canvas").set("localRefresh", false);

      const layoutConfig = getForceLayoutConfig(
        this.graph,
        largeGraphMode,
        null,
        nodeMap,
        aggregatedNodeMap
      );
      layoutConfig.center = [CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2];
      layout.instance = new G6.Layout["gForce"](layoutConfig);
      layout.instance.init({
        nodes: currentUnproccessedData.nodes,
        edges: processedEdges
      });
      layout.instance.execute();
      this.bindListener(this.graph);
      this.graph.data({
        nodes: aggregatedData.nodes,
        edges: processedEdges
      });
      this.graph.render();
    },
    //context menu
    setContextMenu(clusteredData) {
      const { data } = this;
      return new G6.Menu({
        shouldBegin(evt) {
          if (evt.target && evt.target.isCanvas && evt.target.isCanvas())
            return true;
          if (evt.item) return true;
          return false;
        },
        getContent(evt) {
          const { item } = evt;
          if (evt.target && evt.target.isCanvas && evt.target.isCanvas()) {
            return `<ul>
          <li id='show'>显示所有隐藏元素</li>
          <li id='collapseAll'>聚合所有聚类</li>
        </ul>`;
          } else if (!item) return;
          const itemType = item.getType();
          const model = item.getModel();
          if (itemType && model) {
            if (itemType === "node") {
              if (model.level !== 0) {
                return `<ul>
              <li id='expand'>展开该聚合点</li>
              <li id='hide'>隐藏该节点</li>
            </ul>`;
              } else {
                return `<ul>
            <li id='neighbor-1'>新建属性</li>
            <li id='neighbor-1'>新建关系</li>
            <li id='neighbor-1'>编辑模型</li>
             <!-- <li id='collapse'>聚合所属聚类</li>
            <li id='neighbor-1'>扩展一度关系</li>
              <li id='neighbor-2'>扩展二度关系</li>
              <li id='neighbor-3'>扩展三度关系</li>
              <li id='hide'>隐藏该节点</li>-->
            </ul>`;
              }
            } else {
              return `<ul>
            <li id='hide'>隐藏该边</li>
          </ul>`;
            }
          }
        },
        handleMenuClick: (target, item) => {
          const model = item && item.getModel();
          const liIdStrs = target.id.split("-");
          let mixedGraphData;
          switch (liIdStrs[0]) {
            case "hide":
              this.graph.hideItem(item);
              hiddenItemIds.push(model.id);
              break;
            case "expand":
              mixedGraphData = this.getAfterExpandNodeData({
                model,
                clusteredData,
                data
              });
              break;
            //FIXME
            // case "expandAll":
            //   console.log(aggregatedNodeMap);
            //   const keys = Object.keys(aggregatedNodeMap) || [];
            //   const tempMixedGraphData = { edges: [], nodes: [] };
            //   if (!keys.length) return;
            //   keys.map(key => {
            //     console.log(aggregatedNodeMap[key]);
            //     const { edges, nodes } = this.getAfterExpandNodeData({
            //       model: aggregatedNodeMap[key],
            //       clusteredData,
            //       data
            //     });
            //     tempMixedGraphData.edges.push(...edges);
            //     tempMixedGraphData.nodes.push(...nodes);
            //   });
            //   mixedGraphData = tempMixedGraphData;
            //   break;
            case "collapse":
              const aggregatedNode = aggregatedNodeMap[model.clusterId];
              manipulatePosition = {
                x: aggregatedNode.x,
                y: aggregatedNode.y
              };
              collapseArray.push(aggregatedNode);
              for (let i = 0; i < expandArray.length; i++) {
                if (expandArray[i].id === model.clusterId) {
                  expandArray.splice(i, 1);
                  break;
                }
              }
              mixedGraphData = getMixedGraph(
                clusteredData,
                data,
                nodeMap,
                aggregatedNodeMap,
                expandArray,
                collapseArray
              );
              break;
            case "collapseAll":
              expandArray = [];
              collapseArray = [];
              mixedGraphData = getMixedGraph(
                clusteredData,
                data,
                nodeMap,
                aggregatedNodeMap,
                expandArray,
                collapseArray
              );
              break;
            case "neighbor":
              const expandNeighborSteps = parseInt(liIdStrs[1]);
              mixedGraphData = getNeighborMixedGraph(
                model,
                expandNeighborSteps,
                data,
                clusteredData,
                currentUnproccessedData,
                nodeMap,
                aggregatedNodeMap,
                1
              );
              break;
            case "show":
              this.showItems(this.graph);
              break;
            default:
              break;
          }
          if (mixedGraphData) {
            cachePositions = cacheNodePositions(this.graph.getNodes());
            currentUnproccessedData = mixedGraphData;
            this.handleRefreshGraph(
              this.graph,
              currentUnproccessedData,
              CANVAS_WIDTH,
              CANVAS_HEIGHT,
              largeGraphMode,
              this.edgeLabelVisible,
              false
            );
          }
        },
        // offsetX and offsetY include the padding of the parent container
        // 需要加上父级容器的 padding-left 16 与自身偏移量 10
        offsetX: 16 + 10,
        // 需要加上父级容器的 padding-top 24 、画布兄弟元素高度、与自身偏移量 10
        offsetY: 0,
        // the types of items that allow the menu show up
        // 在哪些类型的元素上响应
        itemTypes: ["node", "edge", "canvas"]
      });
    },
    //展开节点后的数据
    getAfterExpandNodeData({ model, clusteredData, data }) {
      const newArray = manageExpandCollapseArray(
        this.graph.getNodes().length,
        model,
        collapseArray,
        expandArray,
        this.graph
      );
      expandArray = newArray.expandArray;
      collapseArray = newArray.collapseArray;
      console.log("aggregatedNodeMap", aggregatedNodeMap);
      let mixedGraphData = getMixedGraph(
        clusteredData,
        data,
        nodeMap,
        aggregatedNodeMap,
        expandArray,
        collapseArray
      );
      console.log("newArray", newArray);
      console.log("mixedGraphData", mixedGraphData);
      return mixedGraphData;
    },
    // 事件绑定
    bindListener(graph) {
      graph.on("keydown", evt => {
        const code = evt.key;
        if (!code) {
          return;
        }
        if (code.toLowerCase() === "shift") {
          shiftKeydown = true;
        } else {
          shiftKeydown = false;
        }
      });
      graph.on("keyup", evt => {
        const code = evt.key;
        if (!code) {
          return;
        }
        if (code.toLowerCase() === "shift") {
          shiftKeydown = false;
        }
      });
      graph.on("node:mouseenter", evt => {
        const { item } = evt;
        const model = item.getModel();
        const currentLabel = model.label;
        model.oriFontSize = model.labelCfg.style.fontSize;
        item.update({
          label: model.oriLabel
        });
        model.oriLabel = currentLabel;
        graph.setItemState(item, "hover", true);
        item.toFront();
      });

      graph.on("node:mouseleave", evt => {
        const { item } = evt;
        const model = item.getModel();
        const currentLabel = model.label;
        item.update({
          label: model.oriLabel
        });
        model.oriLabel = currentLabel;
        graph.setItemState(item, "hover", false);
      });

      graph.on("edge:mouseenter", evt => {
        const { item } = evt;
        const model = item.getModel();
        const currentLabel = model.label;
        item.update({
          label: model.oriLabel
        });
        model.oriLabel = currentLabel;
        item.toFront();
        item.getSource().toFront();
        item.getTarget().toFront();
      });

      graph.on("edge:mouseleave", evt => {
        const { item } = evt;
        const model = item.getModel();
        const currentLabel = model.label;
        item.update({
          label: model.oriLabel
        });
        model.oriLabel = currentLabel;
      });
      // click node to show the detail drawer
      graph.on("node:click", evt => {
        this.stopLayout();
        if (!shiftKeydown) this.clearFocusItemState(graph);
        else this.clearFocusEdgeState(graph);
        const { item } = evt;

        // highlight the clicked node, it is down by click-select
        graph.setItemState(item, "focus", true);

        if (!shiftKeydown) {
          // 将相关边也高亮
          const relatedEdges = item.getEdges();
          relatedEdges.forEach(edge => {
            graph.setItemState(edge, "focus", true);
          });
        }
      });

      // click edge to show the detail of integrated edge drawer
      graph.on("edge:click", evt => {
        this.stopLayout();
        if (!shiftKeydown) this.clearFocusItemState(graph);
        const { item } = evt;
        // highlight the clicked edge
        graph.setItemState(item, "focus", true);
      });

      // click canvas to cancel all the focus state
      graph.on("canvas:click", evt => {
        this.clearFocusItemState(graph);
        console.log(
          graph.getGroup(),
          graph.getGroup().getBBox(),
          graph.getGroup().getCanvasBBox()
        );
      });
      //创建边
      // graph.on('aftercreateedge', e => {
      //   const edges = graph.save().edges
      //   G6.Util.processParallelEdges(edges)
      //   graph.getEdges().forEach((edge, i) => {
      //     graph.updateItem(edge, {
      //       curveOffset: edges[i].curveOffset,
      //       curvePosition: edges[i].curvePosition
      //     })
      //   })
      // })
    },

    // 清除focus状态及相应样式
    clearFocusItemState(graph) {
      if (!graph) return;
      this.clearFocusNodeState(graph);
      this.clearFocusEdgeState(graph);
    },
    // 清除图上所有节点的 focus 状态及相应样式
    clearFocusNodeState(graph) {
      const focusNodes = graph.findAllByState("node", "focus");
      focusNodes.forEach(fnode => {
        graph.setItemState(fnode, "focus", false); // false
      });
    },
    // 清除图上所有边的 focus 状态及相应样式
    clearFocusEdgeState(graph) {
      const focusEdges = graph.findAllByState("edge", "focus");
      focusEdges.forEach(fedge => {
        graph.setItemState(fedge, "focus", false);
      });
    },
    //隐藏指定元素
    hideItems(graph) {
      hiddenItemIds.forEach(id => {
        graph.hideItem(id);
      });
    },
    //展示指定元素
    showItems(graph) {
      graph.getNodes().forEach(node => {
        if (!node.isVisible()) graph.showItem(node);
      });
      graph.getEdges().forEach(edge => {
        if (!edge.isVisible()) edge.showItem(edge);
      });
      hiddenItemIds = [];
    },
    stopLayout() {
      layout.instance.stop();
    },
    //refresh graph
    handleRefreshGraph(
      graph,
      graphData,
      width = CANVAS_WIDTH,
      height = CANVAS_HEIGHT,
      largeGraphMode = false,
      edgeLabelVisible = false,
      isNewGraph = false
    ) {
      if (!graphData || !graph) return;
      this.clearFocusItemState(graph);
      // reset the filtering
      graph.getNodes().forEach(node => {
        if (!node.isVisible()) node.show();
      });
      graph.getEdges().forEach(edge => {
        if (!edge.isVisible()) edge.show();
      });

      let nodes = [],
        edges = [];

      nodes = graphData.nodes;
      const processRes = processNodesEdges(
        nodes,
        graphData.edges || [],
        width,
        height,
        largeGraphMode,
        edgeLabelVisible,
        isNewGraph,
        cachePositions
      );

      edges = processRes.edges;
      graph.changeData({ nodes, edges });

      this.hideItems(graph);
      graph.getNodes().forEach(node => {
        node.toFront();
      });

      // layout.instance.stop();
      // force 需要使用不同 id 的对象才能进行全新的布局，否则会使用原来的引用。因此复制一份节点和边作为 force 的布局数据
      layout.instance.init({
        nodes: graphData.nodes,
        edges
      });

      layout.instance.minMovement = 0.0001;
      // layout.instance.getCenter = d => {
      // 	const cachePosition = cachePositions[d.id];
      // 	if (!cachePosition && (d.x || d.y)) return [d.x, d.y, 10];
      // 	else if (cachePosition) return [cachePosition.x, cachePosition.y, 10];
      // 	return [width / 2, height / 2, 10];
      // }
      layout.instance.getMass = d => {
        const cachePosition = cachePositions[d.id];
        if (cachePosition) return 5;
        return 1;
      };
      layout.instance.execute();

      return { nodes, edges };
    },

    //canvas-menu func
    setCanvasMenuProps(key) {
      this[key] = !this[key];
    },
    //检索节点
    handleSearchNode({ keyword: id, cb }) {
      const { graph, data, edgeLabelVisible, clusteredData } = this;
      if (!graph || graph.get("destroyed")) return false;
      let item = graph.findById(id);
      const originNodeData = nodeMap[id];
      //节点存在但还未展示
      if (!item && originNodeData) {
        const { clusterId } = originNodeData;
        const aggregatedNode = aggregatedNodeMap[clusterId];
        cachePositions = cacheNodePositions(graph.getNodes());
        currentUnproccessedData = this.getAfterExpandNodeData({
          model: aggregatedNode,
          clusteredData,
          data
        });
        //  getExtractNodeMixedGraph(
        //   originNodeData,
        //   data,
        //   nodeMap,
        //   aggregatedNodeMap,
        //   currentUnproccessedData
        // );
        this.handleRefreshGraph(
          graph,
          currentUnproccessedData,
          CANVAS_WIDTH,
          CANVAS_HEIGHT,
          largeGraphMode,
          edgeLabelVisible,
          false
        );
        item = graph.findById(id);
      }
      if (!item) return cb(false);
      if (item && item.getType() !== "node") return false;
      this.clearFocusItemState(graph);
      graph.focusItem(item, true);
      graph.setItemState(item, "focus", true);
      const relatedEdges = item.getEdges();
      relatedEdges.forEach(edge => {
        graph.setItemState(edge, "focus", true);
      });
      return cb(true);
    },
    //适配屏幕
    handleFitView() {
      const { graph } = this;
      if (!graph || graph.destroyed) return;
      graph.fitView(40);
    },
    //检索最短路径
    handleFindPath() {
      const { graph, data } = this;
      if (!graph || graph.get("destroyed")) return false;
      const selectedNodes = graph.findAllByState("node", "focus");
      if (selectedNodes.length !== 2) {
        alert("请选择有且两个节点！");
        return;
      }
      this.clearFocusItemState(graph);
      const { path } = findShortestPath(
        data,
        selectedNodes[0].getID(),
        selectedNodes[1].getID(),
        false, //方向性
        false //边的权重
      );
      graph.getEdges().forEach(edge => {
        const edgeModel = edge.getModel();
        const source = edgeModel.source;
        const target = edgeModel.target;
        const sourceInPathIdx = path.indexOf(source);
        const targetInPathIdx = path.indexOf(target);
        if (sourceInPathIdx === -1 || targetInPathIdx === -1) return;
        if (Math.abs(sourceInPathIdx - targetInPathIdx) === 1) {
          edge.toFront();
          graph.setItemState(edge, "focus", true);
        }
      });
      path.forEach(id => {
        const pathNode = graph.findById(id);
        pathNode.toFront();
        graph.setItemState(pathNode, "focus", true);
      });
    }
  }
};
</script>

<style lang="scss">
/* tooltip wrapper */
.g6-component-tooltip {
  background-color: #424242;
  padding: 5px 8px;
  color: #e4e4e4;
  font-size: 14px;
  border: none;
  box-shadow: rgba(0, 0, 0, 0.3) 0px 0px 6px;
  max-width: 400px;
  width: fit-content;
  ul {
    padding: 0;
    li {
      list-style: none;
      display: flex;
      border-bottom: 1px solid #545454;
      padding: 5px;
      span {
        &:nth-child(1) {
          width: 100px;
          flex-shrink: 0;
        }
        &:nth-child(2) {
        }
      }
    }
  }
}
</style>
