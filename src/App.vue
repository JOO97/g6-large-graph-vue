<template>
  <div id="app">
    <div>
      <button>create model</button>
      <button>edit model</button>
      <button>del model</button>
    </div>
    <LargeGraph
      v-if="!loading"
      @on-click-menu="handleGraphMenuClick"
      :data="graphData"
    />
  </div>
</template>

<script>
import Vue from "vue";

import LargeGraph from "./components/LargeGraph/index.vue";
import LargeGraphOriginal from "./components/LargeGraphCopy/index.vue";
import CanvasMenu from "./components/CanvasMenu/index.vue";

export default {
  name: "app",
  components: { LargeGraph, CanvasMenu, LargeGraphOriginal },
  data() {
    return {
      showOriginal: false,
      loading: true,
      modeList: [
        {
          id: "1269305762472103938",
          createUser: "admin",
          createTime: 1591461051000,
          modifyUser: "admin",
          modifyTime: 1591461051000,
          name: "patent",
          description: "专利",
          projectId: "1267801840259440641",
          createType: 0,
          isDisplay: null,
          deleted: false,
          count: 100,
          color: "#1890FF"
        },
        {
          id: "1269301532961116162",
          createUser: "admin",
          createTime: 1591460043000,
          modifyUser: "admin",
          modifyTime: 1591460043000,
          name: "news",
          description: "新闻",
          projectId: "1267801840259440641",
          createType: 0,
          isDisplay: null,
          deleted: false,
          count: 100,
          color: "#EC1716"
        },
        {
          id: "1267802258834202625",
          createUser: "admin",
          createTime: 1591102588000,
          modifyUser: "admin",
          modifyTime: 1591453008000,
          name: "wos",
          description: "论文",
          projectId: "1267801840259440641",
          createType: 0,
          isDisplay: null,
          deleted: false,
          count: 100,
          color: "#40BC83"
        }
      ],
      graphData: {
        nodes: [],
        edges: [
          {
            source: "1267802258834202625",
            target: "1269305762472103938",
            customLabel: "hasNarrow"
          }
        ]
      }
    };
  },
  created() {
    this.getModelList();
  },
  methods: {
    //获取g6需要的节点数据
    getGraphNodeData(list, nodeType = 1, pKey = "") {
      return list.map(item => {
        const { id, name, expand } = item;
        return {
          id,
          name,
          nodeType,
          expand: nodeType === 1 ? expand : undefined,
          pId: nodeType !== 1 ? item[pKey] : undefined,
          customInfo: item
        };
      });
    },
    getModelList() {
      console.log(this.modeList);
      const data = this.getGraphNodeData(
        this.modeList.map(item => {
          return { ...item, expand: false };
        }),
        1
      );
      this.graphData.nodes = data;
      this.loading = false;
    },
    handleGraphMenuClick({ menuType, data }) {
      console.log("data", data);
      const { id } = data;
      switch (menuType) {
        case "createModel":
          this.graphData.nodes.push(
            ...this.getGraphNodeData(
              [
                {
                  id: "1267802258834202620",
                  createUser: "admin",
                  createTime: 1591102588000,
                  modifyUser: "admin",
                  modifyTime: 1591453008000,
                  name: "new model",
                  description: "论文",
                  projectId: "1267801840259440641",
                  createType: 0,
                  isDisplay: null,
                  deleted: false,
                  count: 100,
                  color: "#40BC83"
                }
              ],
              1
            )
          );
          break;
        case "editModel":
          Vue.set(this.modeList, 0, {
            id: "1269305762472103938",
            createUser: "admin",
            createTime: 1591461051000,
            modifyUser: "admin",
            modifyTime: 1591461051000,
            name: "edit model",
            description: "专利",
            projectId: "1267801840259440641",
            createType: 0,
            isDisplay: null,
            deleted: false,
            count: 100,
            color: "#1890FF"
          });
          this.getModelList();
          break;
        case "showCNode":
          const cList = this.getGraphNodeData(
            [
              {
                id: "1269305763956887554",
                createUser: "admin",
                createTime: 1591461051000,
                modifyUser: "admin",
                modifyTime: 1591461051000,
                projectId: "1267801840259440641",
                dataModelId: "1269305762472103938",
                name: "id",
                label: "主键",
                type: 2,
                isDisplay: false,
                isForeign: false,
                foreignModelId: null,
                foreignModelPropertyId: null,
                isAttachment: false,
                isSearch: false,
                jsonDictionaryEnumeration: null,
                primaryDisplayProperty: false,
                displayListIndex: null,
                displayDetailIndex: null,
                defaultFilterProperty: false,
                foreignKey: null,
                dictionaryEnumerations: null
              }
            ],
            2,
            "dataModelId"
          );
          console.log(cList);
          let i;
          this.graphData.nodes.map((item, index) => {
            if (item.id === id) i = index;
          });
          console.log(i);
          Vue.set(this.graphData.nodes[i], "expand", true);
          this.graphData.nodes.push(...cList);
          break;
        default:
          break;
      }
    }
  }
};
</script>

<style lang="scss">
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background: #fff;
  overflow: hidden;
  height: 100vh;
  width: 100vw;
  position: relative;
}
</style>
