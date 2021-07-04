<template>
  <div class="map-charts">
    <div id="map"></div>
    <Window :icon="countIcon">
      <div class="charts" ref="charts"></div>
    </Window>
    <div class="select-box">
      <div class="select-item">
        <v-slider v-model="slider" step="10" thumb-label ticks></v-slider>
      </div>
      <div class="select-item">
        <v-select
          v-model="mapStat.province"
          :items="mapCode"
          item-text="label"
          item-value="value"
          label="选择省份"
          dense
          solo
        ></v-select>
      </div>
      <div class="select-item">
        <v-select
          v-model="mapStat.level"
          :items="levelCode"
          item-text="label"
          item-value="value"
          label="选择级别"
          dense
          solo
        ></v-select>
      </div>
      <div class="select-item">
        <v-select
          v-model="stat"
          :items="statList"
          item-text="label"
          item-value="value"
          class="select-item"
          label="数据分布"
          dense
          solo
        ></v-select>
      </div>
    </div>
  </div>
</template>
<script>
import { Marker, Scene, LineLayer } from "@antv/l7";
import { GaodeMap } from "@antv/l7-maps";
import { DrawControl } from "@antv/l7-draw";
import { CountyLayer } from "@antv/l7-district";

import { pointInPolygon } from "geometric";
import * as echarts from "echarts";

// import mapCode from "../assets/mapCode";

import Window from "./Window.vue";

export default {
  props: {
    longitude: Number,
    latitude: Number,
    defaultGaodeConfig: {
      type: Object,
      default: () => ({
        pitch: 40,
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

    selectItems: Array,
  },
  components: { Window },
  data() {
    let showCounty = false;
    return {
      // @ts-ignore
      countIcon: require("../assets/count.svg"),
      mapCode: [],
      levelCode: [
        {
          label: "市级",
          value: 2,
        },
        {
          label: "县级",
          value: 3,
        },
      ],
      statList: [
        {
          label: "请选择统计数据",
          value: -1,
        },
        {
          label: "猪肉",
          value: 0,
        },
        {
          label: "羊肉",
          value: 1,
        },
        {
          label: "牛肉",
          value: 2,
        },
      ],
      mapStat: {
        province: "",
        level: 2,
      },
      stat: -1,

      series: [],
      lines: [],
      select: this.value || [],
      slider: 30,

      menus: [
        {
          // @ts-ignore
          icon: require("../assets/fb.svg"),
          handle: () => {
            if (showCounty) {
              this.mapStatLayer.hide();
            } else {
              this.mapStatLayer.show();
            }
            showCounty = !showCounty;
          },
        },
      ],
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
    slider(slider) {
      this.proxyValue = this.innerMarkers
        .filter((item) => item.getExtData().data.value <= slider)
        .map((item) => item.getExtData().data.id);
    },
    stat(stat) {
      if (stat === -1) return;
      const mapList = { 2: getCity, 3: getCount };
      const data = mapList[this.mapStat.level](
        this.mapCode,
        this.mapStat.province
      )
        .map((item) => item.value)
        .map((code) => {
          return {
            code,
            count: Math.floor(Math.random() * 100),
          };
        });
      this.mapStatLayer.updateData(data);
    },
    mapStat: {
      deep: true,
      handler(mapStat) {
        const mapList = { 2: getCity, 3: getCount };
        const colors = ["orange", "green", "green"];
        this.stat = -1;
        this.mapStatLayer?.destroy();
        this.mapStatLayer = new CountyLayer(this.scene, {
          autoFit: false,
          zIndex: -1,
          visible: true,
          stroke: "red",
          data: [],
          joinBy: ["adcode", "code"],
          fill: {
            color: {
              field: "count",
              values: colors,
            },
          },
          label: {
            enable: false,
            opacity: 0.8,
            textAllowOverlap: false,
          },
          popup: {
            Html: (prop) => `${prop.NAME_CHN}<br>年产量：${prop.count}只`,
          },
          depth: mapStat.level,
          adcode: mapList[mapStat.level](this.mapCode, mapStat.province).map(
            (item) => item.value
          ),
        });
      },
    },
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
        initControl.call(this);
        initCountyLayer.call(this);

        function initControl() {
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
            // @ts-ignore
            const selectMarkers = this.markers.filter((marker) => {
              const polygon = ev.feature.geometry.coordinates[0];
              return pointInPolygon(
                [marker.longitude, marker.latitude],
                polygon
              );
            });
            this.proxyValue = selectMarkers.map((m) => m.id);
          });
          scene.addControl(drawControl);
        }
        async function initCountyLayer() {
          // todo: 后期可以需要
        }
      });
      this.waitMapLoaded = function onLoaded() {
        return new Promise((resolve) => {
          scene.on("loaded", function onLoaded() {
            resolve();
            scene.off("loaded", onLoaded);
          });
        });
      };
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
    onChangeSelect() {},
    onChangeProvinceSelect(code) {
      console.log(getCount(this.mapCode, code));
      this.mapStatLayer.updateDistrict(
        getCity(this.mapCode, code).map((n) => n.value)
      );
    },

    onChangeLevelSelect() {},
  },
  async beforeCreate() {
    this.mapCode = await import("../assets/mapCode.js").then(
      (res) => res.default
    );
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
  const layer = new LineLayer({ zIndex: 1, blend: "max" })
    .source(marker.line, {
      parser: {
        type: "json",
        coordinates: "path",
      },
    })
    .color(marker.color)
    .shape("arc3d")
    .size(3)
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

function getCity(mapCode = [], code) {
  return mapCode.find((item) => item.value === code).children;
}

function getCount(mapCode = [], code) {
  return mapCode
    .find((item) => item.value === code)
    .children.reduce((prev, item) => {
      prev = prev.concat(item.children);
      return prev;
    }, []);
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

.select-box {
  width: 100%;
  display: flex;
  align-items: center;
  position: fixed;
  z-index: 100;
  right: 10px;
  bottom: 0;
}

.select-item {
  margin-left: 10px;
  flex: 1;
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
