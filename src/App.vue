<template>
  <v-app>
    <v-main>
      <MapCharts
        v-model="select"
        :latitude="41.202626"
        :longitude="111.778313"
        :chartsTitle="chartsTitle"
        :markers="markers"
        :xAxisData="xAxisData"
        ref="mc"
      ></MapCharts>
    </v-main>
  </v-app>
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
      select: [],
      markers: [
        {
          id: 1,
          text: "测试1",
          color: "blue",
          longitude: 111.778313,
          latitude: 41.202626,
          serie: {
            type: "line",
            data: [
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
            ],
          },
          line: [
            {
              path: [
                [111.778313, 41.202626],
                [116.414543, 43.67502],
              ],
            },
            {
              path: [
                [111.778313, 41.202626],
                [121.512199, 45.720751],
              ],
            },
          ],
        },
        {
          id: 2,
          text: "测试2",
          color: "yellow",
          longitude: 116.414543,
          latitude: 43.67502,
          serie: {
            type: "line",
            data: [
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
            ],
          },
        },
        {
          id: 3,
          text: "测试3",
          color: "red",
          longitude: 109.537102,
          latitude: 40.245149,
          serie: {
            type: "line",
            data: [
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
            ],
          },
        },
        {
          id: 4,
          text: "测试4",
          color: "green",
          longitude: 121.512199,
          latitude: 45.720751,
          serie: {
            type: "bar",
            data: [
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
              Math.floor(Math.random() * 100),
            ],
          },
          line: [
            {
              path: [
                [121.512199, 45.720751],
                [111.778313, 41.202626],
              ],
            },
            {
              path: [
                [111.778313, 41.202626],
                [109.537102, 40.245149],
              ],
            },
          ],
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
        type: "line",
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
    // setTimeout(() => {}, 4000);
    // setInterval(() => {
    //   this.$refs.mc.updateCharts();
    // }, 1000);
  },
};
</script>
