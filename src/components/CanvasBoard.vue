<template>
  <div class="wrapper">
    <div class="controls">
      <label for="uiSelectCanvasSize">Ratio:</label>
      <select id="uiSelectCanvasSize" @change="uiSelectCanvasSize">
        <option value="16:9" selected>16:9</option>
        <option value="1:1">1:1</option>
        <option value="4:3">4:3</option>
        <option value="9:16">9:16</option>
        <option value="3:4">3:4</option>
      </select>

      <label for="uiSelectGridSize">Grid:</label>
      <select id="uiSelectGridSize" @change="uiSelectGridSize">
        <option value="3x3" selected>3x3</option>
        <option value="5x5">5x5</option>
      </select>

      <label for="uiSelectCenterFactor">Central zone:</label>
      <select id="uiSelectCenterFactor" @change="uiSelectCenterFactor">
        <option value="0">0%</option>
        <option value="0.05">5%</option>
        <option value="0.1" selected>+10%</option>
        <option value="0.15">+15%</option>
        <option value="0.2">+20%</option>
        <option value="0.3">+30%</option>
        <option value="0.4">+40%</option>
        <option value="0.5">+50%</option>
      </select>

      <label for="uiToggleGrid">Show grid & info:</label>
      <input type="checkbox" id="uiToggleGrid" checked @click="uiToggleGrid" />
    </div>
    <div
      class="canvas"
      ref="canvas"
      @drop="onDrop($event)"
      @dragover.prevent
      @dragenter.prevent
      :style="{
        width: width + 'px',
        height: height + 'px'
      }"
    >
      <VideoClip
        v-for="(clip, index) in clips"
        :id="clip.id"
        :key="clip.id"
        :width="clip.width + 'px'"
        :height="clip.height + 'px'"
        :color="clip.color"
        :showInfo="gridClass === ''"
        :radius="clip.radius"
        :left="clip.left + 'px'"
        :top="clip.top + 'px'"
        :center="clip.center"
        :dragPoint="clip.dragPoint"
        :zIndex="100 + index"
        :dx="clip.delta?.x ?? 0"
        :dy="clip.delta?.y ?? 0"
        :zone="clip.zone"
        @drag-started="clipDragStarted(clip, $event)"
        @update-clip-size="setClipSize(clip, $event)"
        @set-fit-mode="setFitFillMode(clip, $event)"
      >
      </VideoClip>

      <div class="center-point"></div>

      <GridLine
        v-for="(gridline, index) in grid"
        :left="gridline.left"
        :top="gridline.top"
        :height="gridline.height"
        :width="gridline.width"
        :zIndex="1000 + index"
        :key="index"
        :class="gridClass"
      ></GridLine>

      <div
        :class="'center-zone' + gridClass"
        :style="{
          width: centerZone.width + 'px',
          left: centerZone.left + 'px',
          top: centerZone.top + 'px',
          height: centerZone.height + 'px'
        }"
      ></div>

      <div
        :class="'zone' + gridClass"
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

const STICKY_TOLERANCE = 8
const STICK_SIDE_WIDTH = 'w'
const STICK_SIDE_HEIGHT = 'h'

const RESIZE_MODE_FIT = 'fit'
const RESIZE_MODE_FILL = 'fill'

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
          width: 400,
          height: 150,
          color: '#262366',
          radius: '0',
          left: 48,
          top: 10
        },
        {
          id: 'clip2',
          width: 200,
          height: 300,
          color: '#ffcc00',
          radius: '0',
          left: 520,
          top: 120
        },
        {
          id: 'clip3',
          width: 220,
          height: 220,
          radius: '50%',
          color: '#42cb00',
          left: 82,
          top: 294
        }
      ],

      center: {
        x: 0,
        y: 0
      },

      width: 800,
      height: 450,

      gridSize: GRID_SIZE,
      centerZoneSnapFactor: 0.1,
      centerZone: {},

      cellWidth: 0,
      cellHeight: 0,

      gridClass: '',
      zones: [],
      grid: []
    }
  },

  methods: {
    onDrop (evt) {
      const itemID = evt.dataTransfer.getData('itemID')
      const clip = this.clips.find(clip => clip.id == itemID)

      this.dargMoveClip(clip, evt)
      this.updateClipSticking(clip)
    },

    clipDragStarted (clip, evt) {
      /* задаем координаты точки, за которую потащили */
      clip.dragPoint = {
        x: evt.x,
        y: evt.y
      }
    },

    uiSelectCanvasSize (evt) {
      this.clips.forEach(clip => {
        this.checkAndSetStickSide(clip)
      })

      let newSize =
        evt.target.selectedIndex > -1
          ? evt.target.options[evt.target.selectedIndex].value
          : '16:9'
      const unit = 50

      switch (newSize) {
        case '4:3':
          this.width = 16 * unit
          this.height = 12 * unit
          break

        case '1:1':
          this.width = 12 * unit
          this.height = 12 * unit
          break

        case '9:16':
          this.width = 6 * unit
          this.height = 10.6 * unit
          break

        case '3:4':
          this.width = 9 * unit
          this.height = 12 * unit
          break

        default:
        case '16:9':
          this.width = 16 * unit
          this.height = 9 * unit
          break
      }

      setTimeout(this.initialSetup, 100, true)
    },

    uiSelectGridSize (evt) {
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

    uiSelectCenterFactor (evt) {
      let newSize =
        evt.target.selectedIndex > -1
          ? evt.target.options[evt.target.selectedIndex].value
          : '0.1'

      this.centerZoneSnapFactor = parseFloat(newSize)

      setTimeout(this.initialSetup, 100)
    },

    uiToggleGrid () {
      this.gridClass = this.gridClass === '' ? ' hidden' : ''
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
    updateClipSticking (clip, keepSticky = false) {
      let matchedZone = -1

      if (
        clip.center.y > this.centerZone.top &&
        clip.center.y < this.centerZone.bottom &&
        clip.center.x > this.centerZone.left &&
        clip.center.x < this.centerZone.right
      ) {
        matchedZone = this.centerZone.index
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

      //если меняем пропорции канвы, то позиции сохраняются,
      //если меняем сетку или центральную зону, надо пересчитывать
      if (keepSticky) {
        this.applyClipSticking(clip)
      } else {
        //расстояние от центра клипа до центра зоны в процентах от размеров ячейки
        clip.delta = {
          x:
            (clip.center.x - this.zones[matchedZone].center.x) / this.cellWidth,
          y:
            (clip.center.y - this.zones[matchedZone].center.y) / this.cellHeight
        }

        clip.zone = matchedZone

        this.checkAndSetStickSide(clip)
      }
    },

    /**
     * Применяет прилипание - либо к границам канвы, либо к зоне, в которой был клип
     * @param {*} clip
     */
    applyClipSticking (clip) {
      let oldZone = this.zones[clip.zone]

      const canvasRect = this.$refs.canvas.getBoundingClientRect()
      const canvasToClipRatioW = canvasRect.width / clip.width
      const canvasToClipRatioH = canvasRect.height / clip.height

      switch (clip.stickSide) {
        case STICK_SIDE_WIDTH:
          this.setClipSize(clip, {
            width: canvasRect.width,
            height: clip.height * canvasToClipRatioW,
            left: 0,
            top:
              oldZone.center.y +
              clip.delta.y * this.cellHeight -
              clip.height / 2
          })
          break
        case STICK_SIDE_HEIGHT:
          this.setClipSize(clip, {
            width: clip.width * canvasToClipRatioH,
            height: canvasRect.height,
            left:
              oldZone.center.x + clip.delta.x * this.cellWidth - clip.width / 2,
            top: 0
          })
          break
        default:
          this.setClipSize(clip, {
            left:
              oldZone.center.x + clip.delta.x * this.cellWidth - clip.width / 2,
            top:
              oldZone.center.y +
              clip.delta.y * this.cellHeight -
              clip.height / 2
          })
          break
      }
    },

    /**
     * Проверяет, совпадает ли положение и размеры клипа с какой-то из сторон канвы
     * и выставляет у клипа нужный флаг
     */
    checkAndSetStickSide (clip) {
      let widthIsSticky =
        Math.abs(this.width - clip.left - clip.width) < STICKY_TOLERANCE &&
        Math.abs(clip.left) < STICKY_TOLERANCE

      let heightIsSticky =
        Math.abs(this.height - clip.top - clip.height) < STICKY_TOLERANCE &&
        Math.abs(clip.top) < STICKY_TOLERANCE

      if (widthIsSticky) {
        clip.stickSide = STICK_SIDE_WIDTH
      } else if (heightIsSticky) {
        clip.stickSide = STICK_SIDE_HEIGHT
      } else {
        clip.stickSide = 'none'
      }
    },

    /**
     * Растягивает клип в режиме заполнения всей канвы или подгонки по лучшей стороне
     * @param {*} mode Режим fit или fill
     */
    setFitFillMode (clip, mode) {
      const canvasRect = this.$refs.canvas.getBoundingClientRect()
      const width = clip.width
      const height = clip.height
      const canvasToClipRatioW = canvasRect.width / width
      const canvasToClipRatioH = canvasRect.height / height

      let rect = {}
      if (mode === RESIZE_MODE_FIT) {
        if (width * canvasToClipRatioH <= canvasRect.width) {
          rect = {
            width: width * canvasToClipRatioH,
            height: height * canvasToClipRatioH,
            top: 0,
            left: clip.left
          }
        } else {
          rect = {
            width: width * canvasToClipRatioW,
            height: height * canvasToClipRatioW,
            top: clip.top,
            left: 0
          }
        }
      } else if (mode === RESIZE_MODE_FILL) {
        // mode === fill
        if (width * canvasToClipRatioH <= canvasRect.width) {
          rect = {
            width: width * canvasToClipRatioW,
            height: height * canvasToClipRatioW,
            top: (canvasRect.height - height * canvasToClipRatioW) / 2,
            left: (canvasRect.width - width * canvasToClipRatioW) / 2
          }
        } else {
          rect = {
            width: width * canvasToClipRatioH,
            height: height * canvasToClipRatioH,
            top: (canvasRect.height - height * canvasToClipRatioH) / 2,
            left: (canvasRect.width - width * canvasToClipRatioH) / 2
          }
        }
      } else {
        throw new Error(`setFitFillMode: unknown fit mode: ${mode}`)
      }

      this.setClipSize(clip, rect)
    },

    /**
     * Устанавливаем размеры и положение клипа, координаты центра задаются автоматически по ним
     * left, top: inevitable
     * width, height: optional
     * */
    setClipSize (clip, rect, keepSticky = false) {
      if (rect?.width) clip.width = rect.width
      if (rect?.height) clip.height = rect.height

      clip.center = {
        x: rect.left + clip.width / 2,
        y: rect.top + clip.height / 2
      }

      clip.left = rect.left
      clip.top = rect.top

      this.updateClipSticking(clip, keepSticky)
    },

    /**
     * Рассчитываем сетку,  размеры ячеек, делим на зоны, считаем размер увеличенной центральной зоны
     * */
    initGridAndZones () {
      console.log('drawing grid and zones...')
      if (this.gridSize % 2 !== 1) console.error('gridSize: number must be odd')

      //calculate cell sizes

      const cellWidth = this.width / this.gridSize
      const cellHeight = this.height / this.gridSize

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

          const left = (this.width / this.gridSize) * zoneX
          const top = (this.height / this.gridSize) * zoneY
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

    initialSetup (keepSticky = false) {
      console.log(`init...`)

      this.center.x = this.width / 2
      this.center.y = this.height / 2

      this.initGridAndZones()

      this.clips.forEach(clip => {
        //чтобы рассчитать центры
        this.setClipSize(clip, { left: clip.left, top: clip.top }, keepSticky)
      })
    }
  },

  mounted () {
    this.initialSetup()
    // window.addEventListener('resize', this.initialSetup)
  },

  destroyed () {
    // window.removeEventListener('resize', this.initialSetup)
  }
}
</script>

<style scoped>
.wrapper {
  width: 100vw;
  height: 100vh;
  position: absolute;
  overflow: auto;
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

.hidden {
  opacity: 0;
  transition: opacity 1s;
}
</style>
