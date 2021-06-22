<template>
  <div class="map-charts">
    <div id="map"></div>
    <Window :ico="countIco">
      <div class="charts" ref="charts"></div>
    </Window>
  </div>
</template>
<script>
import { Marker, Scene, LineLayer } from "@antv/l7";
import { GaodeMap } from "@antv/l7-maps";
import { DrawControl } from "@antv/l7-draw";
// import { DrillDownLayer } from "@antv/l7-district";
// import { CountryLayer } from "@antv/l7-district";

import { pointInPolygon } from "geometric";
import * as echarts from "echarts";

import Window from "./Window.vue";

export default {
  props: {
    longitude: Number,
    latitude: Number,
    defaultGaodeConfig: {
      type: Object,
      default: () => ({
        pitch: 0,
        style: "light",
        center: [121.435159, 31.256971],
        zoom: 6,
        // minZoom: 10,
        token: "ba2513c38d63b973e454bc231ba1dbce",
      }),
    },

    markers: {
      type: Array,
      default: () => [],
      /**
       * id
       * text
       * color
       * serie
       * longitude
       * latitude
       */
    },
    value: Array, //选中ID列表

    chartsTitle: String,
    xAxisData: Array,
    onLoadSerie: {
      type: Function,
      default: (data) => data.serie,
    },
    onLoadLine: {
      type: Function,
      default: (data) => data.line,
    },
  },
  components: { Window },
  data() {
    return {
      // @ts-ignore
      countIco: require("../assets/count.svg"),
      series: [],
      lines: [],
      select: this.value || [],
    };
  },
  computed: {
    proxyValue: {
      get() {
        return this.value || this.select;
      },
      set(v) {
        v = [...new Set(v)];
        this.select = v;
        this.$emit("input", v);
      },
    },
    selectedMarkers() {
      return this.proxyValue.map((id) =>
        this.markers.find((marker) => marker.id === id)
      );
    },
    chartsOption() {
      return {
        title: {
          text: this.chartsTitle,
          left: "center",
        },
        tooltip: {},
        xAxis: {
          data: this.xAxisData,
        },
        yAxis: {},
        series: this.series,
      };
    },
    innerMarkers() {
      return this.markers.map((marker) => {
        if (!marker.innerMarker) {
          const innerMarker = toInnerMarker(marker, this.onClickMarker);
          Object.defineProperty(marker, "innerMarker", {
            enumerable: false,
            get() {
              return innerMarker;
            },
          });
        }
        return marker.innerMarker;
      });
    },
    location() {
      return this.longitude && this.latitude
        ? [this.longitude, this.latitude]
        : this.defaultGaodeConfig.center;
    },
  },
  watch: {
    location: {
      immediate: true,
      deep: true,
      async handler(v) {
        await this.waitMap();
        this.scene.setCenter(v);
      },
    },
    chartsOption: {
      immediate: true,
      async handler(v) {
        await this.$nextTick();
        this.charts.setOption(v, {
          notMerge: true,
          replaceMerge: ["series"],
        });
      },
    },
    innerMarkers: {
      immediate: true,
      async handler(markers = [], oldMarkers = []) {
        await this.waitMap();
        const rmMarkers = oldMarkers.filter((om) => !markers.includes(om));
        const newMarkers = markers.filter((om) => !oldMarkers.includes(om));
        rmMarkers.forEach((marker) => marker.remove());
        newMarkers.forEach((marker) => {
          this.scene.addMarker(marker);
        });
        // todo: 可以优化
      },
    },
    selectedMarkers: {
      immediate: true,
      async handler(userMarkers) {
        await this.$nextTick();
        this.markers.forEach((m) => this.checkMarker(m, false));
        userMarkers.forEach((um) => this.checkMarker(um, true));
        // todo: 这块可以优化
      },
    },
  },
  methods: {
    initECharts() {
      // 基于准备好的dom，初始化echarts实例
      return echarts.init(this.$refs.charts);
    },
    initGaodeMap() {
      const scene = new Scene({
        id: "map",
        map: new GaodeMap(this.defaultGaodeConfig),
      });

      scene.on("loaded", () => {
        const drawControl = new DrawControl(scene, {
          position: "topright",
          layout: "horizontal", // horizontal vertical
          controls: {
            polygon: true,
            rect: true,
            delete: true,
          },
        });
        drawControl.on("draw.create", (ev) => {
          const selectMarkers = this.markers.filter((marker) => {
            const polygon = ev.feature.geometry.coordinates[0];
            return pointInPolygon([marker.longitude, marker.latitude], polygon);
          });
          this.proxyValue = selectMarkers.map((m) => m.id);
        });
        scene.addControl(drawControl);

        // const colors = ["white", "red"];

        // new DrillDownLayer(scene, {
        //   data: [],
        //   viewStart: "Country",
        //   viewEnd: "City",
        //   zIndex: 10,
        //   fill: {
        //     color: {
        //       field: "NAME_CHN",
        //       values: colors,
        //     },
        //   },
        //   popup: {
        //     enable: true,
        //     Html: (props) => {
        //       return `<span>${props.NAME_CHN}</span>`;
        //     },
        //   },
        // });

        console.log(scene);

        // scene.on("click", (e) => {
        //   console.log(e);
        // });
      });
      this.waitMapLoaded = function onLoaded() {
        return new Promise((resolve) => {
          scene.on("loaded", function onLoaded() {
            resolve();
            scene.off("loaded", onLoaded);
          });
        });
      };
      // const flydata = [
      //   {
      //     name: "path1",
      //     coord: [
      //       [58.00781249999999, 32.84267363195431],
      //       [85.78125, 25.16517336866393],
      //     ],
      //   },
      // ];

      return scene;
    },
    async waitMap() {
      await this.$nextTick();
      await this.waitMapLoaded();
    },
    async onClickMarker(e) {
      if (e.select) {
        this.proxyValue = [...this.proxyValue, e.data.id];
      } else {
        const index = this.proxyValue.indexOf(e.data.id);
        const _temp = this.proxyValue.slice();
        _temp.splice(index, 1);
        this.proxyValue = _temp;
      }
    },
    async checkMarker(userMarker, bool) {
      select(userMarker.innerMarker, bool);
      if (bool) {
        const serie = await this.onLoadSerie(userMarker);
        this.addSerie(userMarker, serie);

        const line = await this.onLoadLine(userMarker);
        this.addLine(userMarker, line);
      } else {
        this.delSerie(userMarker);
        this.delLine(userMarker);
      }
    },
    addSerie(userMarker, serie) {
      if (serie && serie.type) {
        this.series.push(this.bindSerie(serie, userMarker));
      } else {
        console.warn("not serie");
      }
    },
    delSerie(userMarker) {
      const index = this.series.findIndex(
        (item) => userMarker.serie === item || userMarker.id == item.id
      );
      if (index !== -1) this.series.splice(index, 1);
    },
    bindSerie(serie, data) {
      if (data.id) serie.id = data.id;
      serie.itemStyle = { color: data.color };
      return serie;
    },
    addLine(userMarker, line) {
      if (line && line.length) {
        this.lines.push(this.bindLine(line, userMarker));
        this.scene.addLayer(line.layer);
      } else {
        console.warn("not line");
      }
    },
    delLine(userMarker) {
      const index = this.lines.findIndex(
        (item) => userMarker.line === item || userMarker.id == item.id
      );
      if (index !== -1) {
        const line = this.lines[index];
        this.lines.splice(index, 1);
        this.scene.removeLayer(line.layer);
      }
    },
    bindLine(line, data) {
      if (data.id) line.id = data.id;
      if (data.color) line.color = data.color;
      const lineLayer = createMarkerLine(data);
      line.layer = lineLayer;
      return line;
    },
    async updateCharts() {
      for (let i = 0; i < this.series.length; i++) {
        const oldSerie = this.series[i];
        const marker = this.markers.find(
          (m) => oldSerie === m.serie || ("id" in m && m.id === oldSerie.id)
        );

        if (marker) {
          const serie = await this.onLoadSerie(
            marker,
            marker.innerMarker.getExtData()
          );
          this.$set(this.series, i, this.bindSerie(serie, marker));
        }
      }
    },
  },
  mounted() {
    this.charts = this.initECharts();

    this.scene = this.initGaodeMap();
  },
  destroyed() {
    this.charts.dispose();
    this.scene.destroy();
  },
};

function toInnerMarker(marker, clickCallback) {
  const data = {
    select: false,
    data: marker,
  };

  const m = new Marker({
    extData: data,
    element: createMarker(marker),
    offsets: [0, 10],
  }).setLnglat({
    lng: marker.longitude,
    lat: marker.latitude,
  });

  m.on("click", (e) => {
    select(m);
    clickCallback(e.data);
  });

  return m;
}

function select(marker, bool) {
  var data = marker.getExtData();
  const classList = marker.getElement().classList;
  data.select = typeof bool === "boolean" ? bool : !data.select;
  classList.toggle("select", data.select);
}

function createMarker({ text, color }) {
  const markerWrapper = document.createElement("div");
  markerWrapper.className = "marker";
  const markerText = document.createElement("div");
  markerText.className = "marker-text";
  markerText.textContent = text;
  const markerPoint = document.createElement("div");
  markerPoint.style.background = color;
  markerPoint.className = "marker-point";
  markerWrapper.appendChild(markerText);
  markerWrapper.appendChild(markerPoint);

  return markerWrapper;
}

function createMarkerLine(marker) {
  const layer = new LineLayer({ zIndex: 100 })
    .source(marker.line, {
      parser: {
        type: "json",
        coordinates: "path",
      },
    })
    .color(marker.color)
    .shape("arc")
    .size(2)
    .active(true)
    .animate({
      interval: 2,
      trailLength: 2,
      duration: 1,
    })
    .style({
      opacity: 1,
    });
  return layer;
}
</script>
<style>
.map-charts {
  overflow: hidden;
  height: 100vh;
  width: 100%;
}
#map {
  position: absolute;
  width: 100%;
  height: 100%;
}
.charts {
  position: absolute;
  width: 100%;
  height: 100%;
  padding-top: 10px;
}
.marker:not(.select) .marker-point {
  background: #999 !important;
}

.marker-text {
  min-width: 30px;
  height: 20px;
  line-height: 20px;
  text-align: center;
  font-size: 10px;
  border-radius: 8%;
  padding: 0 5px;
  background: #fff;
  box-shadow: #999 0 0 10px 0;
  transition: 0.36s all;
}

.marker-text:hover {
  box-shadow: #ddd 0 0 0 0;
}

.marker-point {
  display: block;
  height: 10px;
  width: 10px;
  border-radius: 50%;
  background: rgb(231, 84, 65);
  position: absolute;
  bottom: -15px;
  left: 50%;
  transform: translateX(-50%);
}
</style>
