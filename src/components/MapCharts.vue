<template>
  <div class="map-charts">
    <div id="map"></div>
    <div class="charts" ref="charts"></div>
  </div>
</template>
<script>
import { Marker, Scene } from "@antv/l7";
import { GaodeMap } from "@antv/l7-maps";

import * as echarts from "echarts";

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
        zoom: 14.89,
        minZoom: 10,
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
  },
  data() {
    return {
      series: [],
      select: this.value || [],
    };
  },
  computed: {
    proxyValue: {
      get() {
        return this.value || this.select;
      },
      set(v) {
        this.select = v;
        this.$emit("input", v);
      },
    },
    selectMarker() {
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
        // [
        //   {
        //     name: "销量",
        //     type: "bar",
        //     data: [5, 20, 36, 10, 10, 20],
        //   },
        //   {
        //     name: "销量2",
        //     type: "bar",
        //     data: [5, 20, 36, 10, 10, 20],
        //   },
        // ],
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
      async handler(v) {
        await this.$nextTick();
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
        await this.$nextTick();
        const rmMarkers = oldMarkers.filter((om) => !markers.includes(om));
        const newMarkers = markers.filter((om) => !oldMarkers.includes(om));
        rmMarkers.forEach((marker) => marker.remove());
        newMarkers.forEach((marker) => {
          this.scene.addMarker(marker);
        });
      },
    },
  },
  methods: {
    initECharts() {
      // 基于准备好的dom，初始化echarts实例
      return echarts.init(this.$refs.charts);
    },
    initGaodeMap() {
      return new Scene({
        id: "map",
        map: new GaodeMap(this.defaultGaodeConfig),
      });
    },
    initMarker() {
      this.innerMarkers.forEach((item) => {
        if (this.proxyValue.includes(item.getExtData().data.id)) {
          select(item);
          this.onClickMarker(item.getExtData());
        }
      });
    },
    bindSerie(serie, data) {
      if (data.id) serie.id = data.id;
      serie.itemStyle = { color: data.color };
      return serie;
    },
    async onClickMarker(e) {
      if (e.select) {
        this.proxyValue = [...this.proxyValue, e.data.id];
        const serie = await this.onLoadSerie(e.data, e);
        if (serie && serie.type) {
          this.series.push(this.bindSerie(serie, e.data));
        } else {
          console.warn("not serie");
        }
      } else {
        const valIndex = this.proxyValue.indexOf(e.data.id);
        const _value = [...this.proxyValue];
        _value.splice(valIndex, 1);
        if (valIndex !== -1) this.proxyValue = _value;

        const index = this.series.findIndex(
          (item) => e.data.serie === item || e.data.id == item.id
        );
        if (index !== -1) this.series.splice(index, 1);
      }
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

    this.initMarker();
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
</script>
<style>
.map-charts {
  overflow: hidden;
  height: 100vh;
  width: 100%;
}
#map {
  position: absolute;
  width: 50%;
  height: 100%;
}
.charts {
  position: absolute;

  right: 0;
  width: 50%;
  height: 100%;
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
