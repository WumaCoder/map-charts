<template>
  <div id="app">
    <MapCharts
      v-model="select"
      :chartsTitle="chartsTitle"
      :markers="markers"
      :xAxisData="xAxisData"
      :onLoadSerie="onLoadSerie"
      ref="mc"
    ></MapCharts>
  </div>
</template>

<script>
import MapCharts from "./components/MapCharts.vue";

export default {
  name: "App",
  components: {
    MapCharts,
  },
  data() {
    return {
      chartsTitle: "数据图表",
      xAxisData: ["2010", "2011", "2012", "2013"],
      select: [1],
      markers: [
        {
          text: "测试1",
          color: "blue",
          id: 1,
          serie: {
            type: "bar",
            data: [44, 33, 55, 33],
            itemStyle: {
              color: "blue",
            },
          },
          longitude: 121.435159,
          latitude: 31.256971,
        },
      ],
    };
  },

  watch: {
    select(v) {
      console.log(v);
    },
  },

  methods: {
    onLoadSerie() {
      return {
        type: "bar",
        data: [
          Math.floor(Math.random() * 100),
          Math.floor(Math.random() * 100),
          Math.floor(Math.random() * 100),
          Math.floor(Math.random() * 100),
        ],
      };
    },
  },

  created() {
    setTimeout(() => {
      this.markers.push({
        text: "测试2",
        color: "yellow",
        id: 2,
        serie: {
          type: "bar",
          data: [5, 20, 36, 10],
          itemStyle: {
            color: "yellow",
          },
        },
        longitude: 121.435159,
        latitude: 31.256,
      });
    }, 4000);
    setInterval(() => {
      this.$refs.mc.updateCharts();
    }, 1000);
  },
};
</script>
