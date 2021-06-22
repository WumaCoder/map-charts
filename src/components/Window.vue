<template>
  <div
    class="scope"
    :class="{ 'scope-close': !moving && !showScope }"
    @mousemove="onMouseMove"
    @mouseup="onMouseUp"
  >
    <div
      class="window"
      :style="{
        left: this.left + 'px',
        top: this.top + 'px',
        width: width,
        height: height,
      }"
    >
      <div
        class="handle"
        @mousedown="onMouseDown"
        @click="showBody = !showBody"
      >
        <img :src="icon" draggable="false" />
      </div>
      <div class="tools">
        <img
          v-for="(item, index) in menus"
          :key="index"
          @click="item.handle"
          class="tools-item"
          :src="item.icon"
        />
      </div>
      <div class="body" ref="body">
        <slot></slot>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    width: String,
    height: String,
    icon: String,

    initLeft: Number,
    initTop: Number,

    menus: Array,
  },
  data() {
    return {
      moving: false,
      left: this.initLeft || 0,
      top: this.initTop || 0,

      downOffsetX: 0,
      downOffsetY: 0,

      showBody: true,
      showScope: false,
    };
  },
  methods: {
    onMouseMove(e) {
      if (this.moving) {
        this.left = e.x - this.downOffsetX;
        this.top = e.y - this.downOffsetY;
      }
    },
    onMouseDown(e) {
      this.moving = true;
      this.downOffsetX = e.offsetX + (this.$refs.body.clientWidth / 2 - 7);
      this.downOffsetY = e.offsetY + (this.$refs.body.clientHeight - 7);
    },
    onMouseUp() {
      this.moving = false;
    },
  },
};
</script>

<style scoped>
.window {
  position: absolute;
  z-index: 101;
  width: 30vw;
  height: 30vh;
}
.handle {
  position: absolute;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: #fff;
  box-shadow: #999 0 0 5px 0;
  left: 50%;
  bottom: -20px;
  transform: translate(-50%);
  cursor: move;
  z-index: 103;
  user-select: none;
}

.handle img {
  margin: 25%;
  width: 50%;
  height: 50%;
}

.body {
  position: absolute;
  z-index: 102;
  width: 100%;
  height: 100%;
  box-shadow: #999 0 0 10px 0;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.712);
}

.body:hover {
  background: #fff;
}

.scope {
  position: fixed;
  width: 98%;
  height: 98%;
  background: rgba(153, 153, 153, 0.5);
  left: 1%;
  top: 1%;
  z-index: 100;
}

.scope-close {
  width: 0;
  height: 0;
}

.tools {
  display: flex;
  flex-direction: column;
  position: absolute;
  top: 50%;
  right: -15px;
  transform: translateY(-50%);
  z-index: 103;
  user-select: none;
}

.tools-item {
  border-radius: 50%;
  box-shadow: #999 0 0 3px 0;
  background: #fff;
  padding: 5px;
  width: 20px;
  height: 20px;
  margin: 3px;
}
.tools-item:active {
  box-shadow: #999 0 0 0 0;
}
</style>
