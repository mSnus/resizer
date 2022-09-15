<template>
  <div
    class="grid-line"
    :style="{
      left: left,
      top: top,
      height: height,
      width: width,
      zIndex: zIndex,
    }"
  ></div>
</template>

<script>
export default {
  props: ["left", "top", "height", "width", "zIndex"],
};
</script>

<style scoped>
.grid-line {
  background-color: #f47bee;
  opacity: 0.3;
  width: 1px;
  position: absolute;
}
</style>
