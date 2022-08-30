<template>
  <div
    class="clip"
    ref="container"
    :style="{
      backgroundColor: color ?? '#262344',
      width: width ?? '450px',
      height: height ?? '300px',
			borderRadius: radius ?? '0',
      left: left ?? 'calc(50% - ' + parseFloat(width ?? '450px') / 2 + 'px)',
      top: top ?? 'calc(50% - ' + parseFloat(height ?? '300px') / 2 + 'px)',
      opacity: 0.8,
      zIndex: zIndex
    }"
    draggable
    @dragstart="startDrag($event)"
  >
    <slot></slot>
    <div class="center-point"></div>
    <div class="clip-info" v-show="showInfo">zone: {{ zone }}</div>
    <div class="clip-info" v-show="showInfo">
      &Delta;x: {{ (dx * 100).toFixed(1) }}%
			<br />
      &Delta;y: {{ (dy * 100).toFixed(1) }}%
    </div>
  </div>
</template>

<script>
export default {
  name: 'VideoClip',
  props: [
    'id',
    'width',
    'height',
		'color',
		'radius',
    'left',
    'top',
    'zIndex',
    'zone',
		'dragPoint',
		'showInfo',
    'dx',
    'dy'
  ],
  data: function () {
    return {}
  },

  methods: {
    startDrag (evt) {
      evt.dataTransfer.dropEffect = 'move'
      evt.dataTransfer.effectAllowed = 'move'
      evt.dataTransfer.setData('itemID', this.id)

      const insideClipX = parseInt(
        evt.clientX - this.$refs.container.getBoundingClientRect().left
      )
      const insideClipY = parseInt(
        evt.clientY - this.$refs.container.getBoundingClientRect().top
      )

      this.$emit('drag-started', {
        x: insideClipX,
        y: insideClipY
      })
    }
  },

  mounted () {
    // console.log('clip mounted', this.$refs.container.getBoundingClientRect().left, this.$refs.container.getBoundingClientRect().width)
    this.$emit('init-clip-size', {
      width: this.$refs.container.getBoundingClientRect().width,
      height: this.$refs.container.getBoundingClientRect().height,
      top: this.$refs.container.getBoundingClientRect().top,
      left: this.$refs.container.getBoundingClientRect().left
    })
  }
}
</script>

<style scoped>
.clip {
  position: absolute;
  overflow: hidden;
  cursor: move;
  color: #000000;
  display: flex;
  flex-wrap: wrap;
  font-size: 1.3rem;
  color: rgb(208, 236, 208);
  justify-content: center;
  align-items: center;
  box-shadow: 2px 2px 10px #00000070;
}

.clip-info {
  min-width: 100%;
}

.center-point {
  background-color: #ff0000;
  opacity: 0.5;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  position: absolute;
  top: calc(50% - 3px);
  left: calc(50% - 3px);
}
</style>
