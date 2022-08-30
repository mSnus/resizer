<template>
  <div class="wrapper">
    <div class="controls">
      <label>Ratio:</label>
      <select id="selectCanvasSize" @change="selectCanvasSize">
        <option value="16:9" selected>16:9</option>
        <option value="1:1">1:1</option>
        <option value="4:3">4:3</option>
        <option value="9:16">9:16</option>
        <option value="3:4">3:4</option>
      </select>
      <label>Grid:</label>
      <select id="selectGridSize" @change="selectGridSize">
        <option value="3x3" selected>3x3</option>
        <option value="5x5">5x5</option>
      </select>
      <label>Central zone:</label>
      <select id="selectCenterFactor" @change="selectCenterFactor">
        <option value="0">0%</option>
        <option value="0.05">5%</option>
        <option value="0.1" selected>+10%</option>
        <option value="0.15">+15%</option>
        <option value="0.2">+20%</option>
        <option value="0.3">+30%</option>
      </select>
    </div>
    <div
      class="canvas"
      ref="canvas"
      @drop="onDrop($event)"
      @dragover.prevent
      @dragenter.prevent
      :style="{
        width: width,
        height: height
      }"
    >
      <VideoClip
        v-for="(clip, index) in clips"
        :id="clip.id"
        :key="clip.id"
        :width="clip.width"
        :height="clip.height"
        :color="clip.color"
        :radius="clip.radius"
        :left="clip.left"
        :top="clip.top"
        :center="clip.center"
        :dragPoint="clip.dragPoint"
        :zIndex="100 + index"
        :dx="clip.delta?.x ?? 0"
        :dy="clip.delta?.y ?? 0"
        :zone="clip.zone"
        @drag-started="clipDragStarted(clip, $event)"
        @init-clip-size="initClipSize(clip, $event)"
      >
      </VideoClip>

      <div class="center-point"></div>

      <GridLine
        v-for="(gridline, index) in grid"
        :left="gridline.left"
        :top="gridline.top"
        :height="gridline.height"
        :width="gridline.width"
        :zIndex="10 + index"
        :key="index"
      ></GridLine>

      <div
        class="center-zone"
        :style="{
          width: centerZone.width + 'px',
          left: centerZone.left + 'px',
          top: centerZone.top + 'px',
          height: centerZone.height + 'px'
        }"
      ></div>

      <div
        class="zone"
        v-for="(zone, index) in zones"
        :key="index + Math.random(0, 1000)"
        :style="{
          left: zone.left + 'px',
          top: zone.top + 'px',
          width: cellWidth + 'px',
          height: cellHeight + 'px'
        }"
      >
        {{ zone.index }}
      </div>
    </div>
  </div>
</template>

<script>
import VideoClip from './VideoClip.vue'
import GridLine from './GridLine.vue'

const GRID_SIZE = 3

export default {
  name: 'CanvasBoard',
  components: {
    VideoClip,
    GridLine
  },
  props: {},
  data () {
    return {
      clips: [
        {
          id: 'clip1',
          width: '400px',
          height: '200px',
          color: '#262366',
          radius: '0',
          left: '20%',
          top: '10%'
        },
        {
          id: 'clip2',
          width: '300px',
          height: '200px',
          color: '#ffcc00',
          radius: '0',
          left: '60%',
          top: '80%'
        },
        {
          id: 'clip3',
          width: '120px',
          height: '120px',
          radius: '50%',
          color: 'lime',
          left: '10%',
          top: '65%'
        }
      ],

      center: {
        x: 0,
        y: 0
      },

      width: '800px',
      height: '450px',
      gridSize: GRID_SIZE,
      centerZoneSnapFactor: 0.1,
      centerZone: {},
      cellWidth: 0,
      cellHeight: 0,

      zones: [],
      grid: []
    }
  },

  methods: {
    onDrop (evt) {
      const itemID = evt.dataTransfer.getData('itemID')
      const clip = this.clips.find(clip => clip.id == itemID)

      this.dargMoveClip(clip, evt)
      this.updateClipZone(clip)
    },

    clipDragStarted (clip, evt) {
      /* задаем координаты точки, за которую потащили */
      clip.dragPoint = {
        x: evt.x,
        y: evt.y
      }
    },

    selectCanvasSize (evt) {
      let newSize =
        evt.target.selectedIndex > -1
          ? evt.target.options[evt.target.selectedIndex].value
          : '16:9'
      const unit = 50

      switch (newSize) {
        case '4:3':
          this.width = 16 * unit + 'px'
          this.height = 12 * unit + 'px'
          break

        case '1:1':
          this.width = 12 * unit + 'px'
          this.height = 12 * unit + 'px'
          break

        case '9:16':
          this.width = 9 * unit + 'px'
          this.height = 16 * unit + 'px'
          break

        case '3:4':
          this.width = 9 * unit + 'px'
          this.height = 12 * unit + 'px'
          break

        default:
        case '16:9':
          this.width = 16 * unit + 'px'
          this.height = 9 * unit + 'px'
          break
      }

      setTimeout(this.initialSetup, 100, true)
    },

    selectGridSize (evt) {
      let newSize =
        evt.target.selectedIndex > -1
          ? evt.target.options[evt.target.selectedIndex].value
          : '3x3'

      switch (newSize) {
        case '5x5':
          this.gridSize = 5
          break

        default:
        case '3x3':
          this.gridSize = 3
          break
      }

      setTimeout(this.initialSetup, 100)
    },

    selectCenterFactor (evt) {
      let newSize =
        evt.target.selectedIndex > -1
          ? evt.target.options[evt.target.selectedIndex].value
          : '0.1'

      this.centerZoneSnapFactor = parseFloat(newSize)

      setTimeout(this.initialSetup, 100)
    },

    /**
     * Вызывается при окончании перетаскивания,
     * чтобы установить координаты относительно
     * точки, за которую тащили
     * */
    dargMoveClip (clip, evt = { clientX: 0, clientY: 0 }) {
      const canvasX = this.$refs.canvas.getBoundingClientRect().left
      const canvasY = this.$refs.canvas.getBoundingClientRect().top

      /*
				client - координаты точки, в которой был курсор, когда драг закончился в КС канвы
				canvas - смещение угла канвы от края экрана в КС экрана
				insideClip - координаты точки, за которую начали тащить, в КС внутри клипа
			 */

      this.setClipSize(clip, {
        left: evt.clientX - canvasX - clip.dragPoint?.x ?? 0,
        top: evt.clientY - canvasY - clip.dragPoint?.y ?? 0
      })
    },

    /**
     * Считает, в какую из зон попал клип
     * приоритет - у центральной зоны
     * */
    updateClipZone (clip, keepDelta = false) {
			let matchedZone = -1

			console.log(clip)

      if (
        clip.center.y > this.centerZone.top &&
        clip.center.y < this.centerZone.bottom &&
        clip.center.x > this.centerZone.left &&
        clip.center.x < this.centerZone.right
      ) {
        matchedZone = 'center'
      } else {
        let matchX = -1
        let matchY = -1

        for (let zoneX = 0; zoneX < this.gridSize; zoneX++) {
          for (let zoneY = 0; zoneY < this.gridSize; zoneY++) {
            let zoneId = zoneX + zoneY * this.gridSize
            let zone = this.zones[zoneId]

            if (clip.center.x < zone.right && matchX == -1) {
              matchX = zoneX
            }

            if (clip.center.y < zone.bottom && matchY == -1) {
              matchY = zoneY
            }
          }
        }

        if (matchX == -1) {
          matchX = this.gridSize - 1
        }
        if (matchY == -1) {
          matchY = this.gridSize - 1
        }

        matchedZone = matchX + matchY * this.gridSize
      }

      let finalZone =
        matchedZone === 'center'
          ? this.zones[this.centerZone.index]
          : this.zones[matchedZone]

      if (keepDelta) {
				let oldZone = this.zones[clip.zone]

        this.setClipSize({
          left: oldZone.center.x + clip.delta.x * this.cellWidth,
          top: oldZone.center.y + clip.delta.y * this.cellHeight
        })
      }

      //расстояние от центра клипа до центра зоны в процентах от размеров ячейки
      clip.delta = {
        x: (clip.center.x - finalZone.center.x) / this.cellWidth,
        y: (clip.center.y - finalZone.center.y) / this.cellHeight
      }

      clip.zone = matchedZone
    },

    /**
     * Получаем размеры клипа в px, когда он mounted
     * Необхожимо, т.к. изначально размеры могут быть указаны в css в процентах
     * */
    initClipSize (clip, evt) {
      clip.width = evt.width
      clip.height = evt.height

      this.setClipSize(clip, {
        left:
          evt.left - parseFloat(this.$refs.canvas.getBoundingClientRect().left),
        top: evt.top - parseFloat(this.$refs.canvas.getBoundingClientRect().top)
      })
    },

    /**
     * Устанавливаем размеры клипа, координаты центра высчитываются автоматически
     * */
    setClipSize (clip, rect) {
      if (rect?.width) clip.width = rect.width
      if (rect?.height) clip.height = rect.height

			clip.center = {
        x: parseFloat(clip.left) + parseFloat(clip.width) / 2,
        y: parseFloat(clip.top) + parseFloat(clip.height) / 2
			}

      clip.left = rect.left + 'px'
      clip.top = rect.top + 'px'
    },

    /**
     * Рассчитываем сетку,  размеры ячеек, делим на зоны, считаем размер увеличенной центральной зоны
     * */
    initGridAndZones () {
      console.log('drawing grid and zones...')
      if (this.gridSize % 2 !== 1) console.error('gridSize: number must be odd')

      //calculate cell sizes

      const cellWidth =
        this.$refs.canvas.getBoundingClientRect().width / this.gridSize
      const cellHeight =
        this.$refs.canvas.getBoundingClientRect().height / this.gridSize

      this.cellWidth = cellWidth
      this.cellHeight = cellHeight

      //setup zones

      this.zones.splice(0, this.zones.length)
      this.grid.splice(0, this.grid.length)

      for (let zoneX = 0; zoneX < this.gridSize; zoneX++) {
        for (let zoneY = 0; zoneY < this.gridSize; zoneY++) {
          const zoneNumber = zoneX + zoneY * this.gridSize

          //calculate zone right and bottom points
          this.zones[zoneNumber] = { right: 0, bottom: 0 }

          const left =
            (parseFloat(this.$refs.canvas.getBoundingClientRect().width) /
              this.gridSize) *
            zoneX

          const top =
            (parseFloat(this.$refs.canvas.getBoundingClientRect().height) /
              this.gridSize) *
            zoneY

          const right = left + cellWidth
          const bottom = top + cellHeight

          this.zones[zoneNumber].left = left
          this.zones[zoneNumber].top = top
          this.zones[zoneNumber].right = right
          this.zones[zoneNumber].bottom = bottom
          this.zones[zoneNumber].center = {
            x: left + cellWidth / 2,
            y: top + cellHeight / 2
          }
          this.zones[zoneNumber].index = zoneNumber

          //grid lines
          if (zoneY == 0 && zoneX < this.gridSize - 1) {
            this.grid.push({
              left: right + 'px',
              top: '0px',
              height: '100%',
              width: '1px'
            })
          }

          if (zoneX == 0 && zoneY < this.gridSize - 1) {
            this.grid.push({
              left: '0px',
              top: bottom + 'px',
              height: '1px',
              width: '100%'
            })
          }
        }
      }

      //setup and enlarge your center zone!
      const zoneCenterIndex = Math.floor(this.gridSize ** 2 / 2)
      const zoneCenter = this.zones[zoneCenterIndex]

      this.centerZone = {
        index: zoneCenterIndex,
        right: Math.floor(
          zoneCenter.right + cellWidth * this.centerZoneSnapFactor
        ),
        bottom: Math.floor(
          zoneCenter.bottom + cellHeight * this.centerZoneSnapFactor
        ),
        left: Math.floor(
          zoneCenter.right - cellWidth * (1 + this.centerZoneSnapFactor)
        ),
        top: Math.floor(
          zoneCenter.bottom - cellHeight * (1 + this.centerZoneSnapFactor)
        ),
        center: {
          x: zoneCenter.center.x,
          y: zoneCenter.center.y
        }
      }

      this.centerZone.width = this.centerZone.right - this.centerZone.left
      this.centerZone.height = this.centerZone.bottom - this.centerZone.top
    },

    initialSetup (keepDelta = false) {
      console.log(`init... keepDelta: ${keepDelta}`)

      this.center.x = parseInt(
        parseFloat(this.$refs.canvas.getBoundingClientRect().width) / 2
      )
      this.center.y = parseInt(
        parseFloat(this.$refs.canvas.getBoundingClientRect().height) / 2
      )
      this.initGridAndZones()
      this.clips.forEach(el => {
        this.updateClipZone(el, keepDelta)
      })
    }
  },

  mounted () {
    this.initialSetup()
    window.addEventListener('resize', this.initialSetup)
  },

  destroyed () {
    window.removeEventListener('resize', this.initialSetup)
  }
}
</script>

<style scoped>
.wrapper {
  width: 100vw;
  height: 100vh;
}

.canvas {
  position: relative;
  border: solid 2px #ccc;
  margin: auto;
  overflow: hidden;
  background: rgb(223, 222, 236);
  background: linear-gradient(
    90deg,
    rgba(223, 222, 236, 1) 0%,
    rgba(227, 232, 238, 1) 35%,
    rgba(197, 232, 238, 1) 100%
  );
}

.center-point {
  background-color: #ff00d9;
  opacity: 0.5;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  position: relative;
  top: calc(50% - 6px);
  left: calc(50% - 6px);
}

.center-zone {
  border: solid 1px #ff000050;
  background-color: #ff000020;
  opacity: 0.6;
  position: absolute;
}

.zone {
  background-color: #7b65d450;
  opacity: 0.15;
  position: absolute;
  display: flex;
  font-size: 3rem;
  color: #fff;
  justify-content: center;
  align-items: center;
}

.controls {
  padding-bottom: 1rem;
}

.controls select {
  font-size: 1rem;
  margin: 0px 0.5rem;
}
</style>
