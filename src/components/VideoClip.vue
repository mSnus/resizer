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
    <div class="controls">
      <button @click="setModeFit">fit</button>
      <button @click="setModeFill">fill</button>
      <button @click="setModeReset">reset</button>
    </div>
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
    return {
      oldWidth: 0,
      oldHeight: 0,
      oldTop: 0,
      oldLeft: 0
    }
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
    },

    setMode (mode) {
      const parentRect = this.$refs.container.parentElement.getBoundingClientRect()
      const width = parseFloat(this.width)
      const height = parseFloat(this.height)
      const parentToClipRatioW = parentRect.width / width
      const parentToClipRatioH = parentRect.height / height

      let rect = {}
      if (mode === 'fit') {
        if (width * parentToClipRatioH <= parentRect.width) {
          rect = {
            width: width * parentToClipRatioH,
            height: height * parentToClipRatioH,
            top: 0,
            left: parseFloat(this.left)
          }
        } else {
          rect = {
            width: width * parentToClipRatioW,
            height: height * parentToClipRatioW,
            top: parseFloat(this.top),
            left: 0
          }
        }
      } else {
        // mode === fill
        if (width * parentToClipRatioH <= parentRect.width) {
          rect = {
            width: width * parentToClipRatioW,
            height: height * parentToClipRatioW,
            top: 0, //--need to center
            left: 0
          }
        } else {
          rect = {
            width: width * parentToClipRatioH,
            height: height * parentToClipRatioH,
            top: 0,
            left: 0 //--need to center
          }
        }
      }

      this.$emit('update-clip-size', rect)
    },

    setModeFit () {
      this.setMode('fit')
    },

    setModeFill () {
      this.setMode('fill')
    },

    setModeReset () {
      let rect = {
        width: this.oldWidth,
        height: this.oldHeight,
        top: this.oldTop,
        left: this.oldLeft
      }

      this.$emit('update-clip-size', rect)
    }
  },

  mounted () {
    this.oldHeight = parseFloat(this.height)
    this.oldWidth = parseFloat(this.width)
    this.oldTop = parseFloat(this.top)
    this.oldLeft = parseFloat(this.left)
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

  background-size: cover;
  background-repeat: no-repeat;
}

.clip-info {
  min-width: 100%;
  text-shadow: 2px 1px 1px black;
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

.clip:nth-child(1) {
  background-image: url('@/assets/img1.jpg');
}

.clip:nth-child(2) {
  background-image: url('@/assets/img2.jpg');
}

.clip:nth-child(3) {
  background-image: url('@/assets/img3.jpg');
}
</style>
