<!-- eslint-disable no-debugger -->
<template>
  <div class="wrapper">
    <div class="controls">
      <label for="uiSelectCanvasSize">Ratio:</label>
      <select id="uiSelectCanvasSize" @change="uiSelectCanvasSize">
        <option
          v-for="(screen, idx) in screens"
          :key="'screen_' + idx"
          :value="screen.name"
          :selected="screen.name === currentScreen"
        >
          {{ screen.name }}
        </option>
      </select>

      <label for="uiSelectGridSize">Grid:</label>
      <select id="uiSelectGridSize" @change="uiSelectGridSize">
        <option value="3x3">3x3</option>
        <option value="5x5" selected>5x5</option>
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
        height: height + 'px',
      }"
    >
      <VideoClip
        v-for="(clip, index) in clips"
        :id="clip.id"
        :key="'clip_' + clip.id"
        :width="clip.width + 'px'"
        :height="clip.height + 'px'"
        :mode="clip.mode"
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
        :stickSide="clip.stickSide"
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
        :key="'line_' + index"
        :class="gridClass"
      ></GridLine>

      <div
        :class="'center-zone' + gridClass"
        :style="{
          width: centerZone.width + 'px',
          left: centerZone.left + 'px',
          top: centerZone.top + 'px',
          height: centerZone.height + 'px',
        }"
      ></div>

      <div
        :class="'zone' + gridClass"
        v-for="zone in zones"
        :key="'zone_' + zone.id"
        :style="{
          left: zone.left + 'px',
          top: zone.top + 'px',
          width: cellWidth + 'px',
          height: cellHeight + 'px',
        }"
      >
        {{ zone.id }}
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import VideoClip from "./VideoClip.vue";
import GridLine from "./GridLine.vue";

const GRID_SIZE = 5;

const STICKY_TOLERANCE = 8;

const RESIZE_MODE_FIT = "fit";
const RESIZE_MODE_FILL = "fill";

const CLIP_MODE_TEXT = "text";
const CLIP_MODE_GRAPHICS = "graphics";

const screenSizes = [
  {
    name: "4:3",
    width: 16,
    height: 12,
    scale: 1,
  },

  {
    name: "1:1",
    width: 12,
    height: 12,
    scale: 1.1,
  },

  {
    name: "9:16",
    width: 6,
    height: 10.6,
    scale: 0.6,
  },

  {
    name: "3:4",
    width: 9,
    height: 12,
    scale: 1,
  },

  {
    name: "21:9",
    width: 21,
    height: 9,
    scale: 1,
  },

  {
    name: "16:9",
    width: 16,
    height: 9,
    scale: 1,
  },
];

export default {
  name: "CanvasBoard",
  components: {
    VideoClip,
    GridLine,
  },
  props: {},
  data() {
    return {
      clips: [
        {
          id: "clip1",
          mode: CLIP_MODE_TEXT,
          width: 400,
          height: 150,
          color: "unset",
          radius: "0",
          left: 48,
          top: 10,
        },
        {
          id: "clip2",
          mode: CLIP_MODE_GRAPHICS,
          width: 200,
          height: 300,
          color: "#ffcc00",
          radius: "0",
          left: 520,
          top: 120,
        },
        {
          id: "clip3",
          mode: CLIP_MODE_GRAPHICS,
          width: 160,
          height: 180,
          radius: "5%",
          color: "#42cb00",
          left: 160,
          top: 180,
        },
        {
          id: "clip4",
          mode: CLIP_MODE_GRAPHICS,
          width: 420,
          height: 120,
          radius: "0",
          color: "#4200cb",
          left: 60,
          top: 320,
        },
      ],

      currentScreen: "16:9",
      screens: screenSizes,

      center: {
        x: 0,
        y: 0,
      },

      keepStickyness: false,

      width: 800,
      height: 450,

      gridSize: GRID_SIZE,
      centerZoneSnapFactor: 0.1,
      centerZone: {},

      cellWidth: 0,
      cellHeight: 0,

      savedClipSizes: [],

      gridClass: "",
      zones: [],
      grid: [],
    };
  },

  methods: {
    onDrop(evt) {
      const itemID = evt.dataTransfer.getData("itemID");
      const clip = this.clips.find((clip) => clip.id == itemID);

      this.dargMoveClip(clip, evt);
      this.calculateMatchedZone(clip);
      this.calculateStickSides(clip);
    },

    clipDragStarted(clip, evt) {
      /* задаем координаты точки, за которую потащили */
      clip.dragPoint = {
        x: evt.x,
        y: evt.y,
      };
    },

    /**
     * Меняем размеры канвы
     * @param evt
     */
    uiSelectCanvasSize(evt) {
      let newSize =
        evt.target.selectedIndex > -1 ? evt.target.options[evt.target.selectedIndex].value : "16:9";
      const unit = 50;

      let oldScreen = this.screens.find((el) => el.name === this.currentScreen);
      let newScreen = this.screens.find((el) => el.name === newSize);
      console.log(oldScreen, newScreen);

      this.width = newScreen.width * unit;
      this.height = newScreen.height * unit;

      this.currentScreen = newScreen.name;

      setTimeout(() => {
        this.initGridAndZones();

        this.clips.forEach((clip) => {
          this.rescaleWithCanvas(clip, oldScreen.scale, newScreen.scale);
          this.applyClipSticking(clip);
          this.calculateStickSides(clip);
        });
      }, 100);
    },

    uiSelectGridSize(evt) {
      let newSize =
        evt.target.selectedIndex > -1 ? evt.target.options[evt.target.selectedIndex].value : "3x3";

      switch (newSize) {
        case "5x5":
          this.gridSize = 5;
          break;

        default:
        case "3x3":
          this.gridSize = 3;
          break;
      }

      this.keepStickyness = false;
      setTimeout(this.initGridAndZones, 100);
    },

    uiSelectCenterFactor(evt) {
      let newSize =
        evt.target.selectedIndex > -1 ? evt.target.options[evt.target.selectedIndex].value : "0.1";

      this.centerZoneSnapFactor = parseFloat(newSize);

      this.keepStickyness = false;
      setTimeout(this.initGridAndZones, 100);
    },

    uiToggleGrid() {
      this.gridClass = this.gridClass === "" ? " hidden" : "";
    },

    uiSaveClipSizes() {
      this.savedClipSizes.splice(0, this.savedClipSizes.length);

      this.clips.forEach((clip) => {
        this.savedClipSizes.push({ left: clip.left, top: clip.top, width: clip.width, height: clip.height });
      });
    },

    /**
     * Вызывается при окончании перетаскивания,
     * чтобы установить координаты относительно
     * точки, за которую тащили
     * */
    dargMoveClip(clip, evt = { clientX: 0, clientY: 0 }) {
      const canvasX = this.$refs.canvas.getBoundingClientRect().left;
      const canvasY = this.$refs.canvas.getBoundingClientRect().top;

      /*
				client - координаты точки, в которой был курсор, когда драг закончился в КС канвы
				canvas - смещение угла канвы от края экрана в КС экрана
				insideClip - координаты точки, за которую начали тащить, в КС внутри клипа
			 */

      this.setClipSize(clip, {
        left: evt.clientX - canvasX - clip.dragPoint?.x ?? 0,
        top: evt.clientY - canvasY - clip.dragPoint?.y ?? 0,
      });
    },

    updateClipCenter(clip) {
      clip.center = {
        x: clip.left + clip.width / 2,
        y: clip.top + clip.height / 2,
      };
    },

    /**
     * Считает, в какую из зон попал клип
     * приоритет - у центральной зоны
     * */
    calculateMatchedZone(clip) {
      console.warn("calculateMatchedZone>>");
      let matchedZone = -1;

      if (
        clip.center.y > this.centerZone.top &&
        clip.center.y < this.centerZone.bottom &&
        clip.center.x > this.centerZone.left &&
        clip.center.x < this.centerZone.right
      ) {
        matchedZone = this.centerZone.index;
      } else {
        let matchX = -1;
        let matchY = -1;

        for (let zoneX = 0; zoneX < this.gridSize; zoneX++) {
          for (let zoneY = 0; zoneY < this.gridSize; zoneY++) {
            let zoneId = zoneX + zoneY * this.gridSize;
            let zone = this.zones[zoneId];

            if (clip.center.x < zone.right && matchX == -1) {
              matchX = zoneX;
            }

            if (clip.center.y < zone.bottom && matchY == -1) {
              matchY = zoneY;
            }
          }
        }

        if (matchX == -1) {
          matchX = this.gridSize - 1;
        }
        if (matchY == -1) {
          matchY = this.gridSize - 1;
        }

        matchedZone = matchX + matchY * this.gridSize;
      }

      //расстояние от центра клипа до центра зоны в процентах от размеров ячейки
      clip.delta = {
        x: (clip.center.x - this.zones[matchedZone].center.x) / this.cellWidth,
        y: (clip.center.y - this.zones[matchedZone].center.y) / this.cellHeight,
      };

      clip.zone = matchedZone;
    },

    /**
     * Применяет прилипание - меняет размеры и положение клипа в зависимости от того,
     * какими сторонами и к чему (зона, канва) он был прилеплен
     * @param {*} clip
     */
    applyClipSticking(clip) {
      console.warn("applyClipSticking>>");
      let oldZone = this.zones[clip.zone];

      const canvasRect = this.$refs.canvas.getBoundingClientRect();
      const canvasToClipRatioW = canvasRect.width / clip.width;
      const canvasToClipRatioH = canvasRect.height / clip.height;

      let widthIsSticky = clip.stickModes?.left && clip.stickModes?.right;
      let heightIsSticky = clip.stickModes?.top && clip.stickModes?.bottom;

      let newClipSize = {
        left: undefined,
        top: undefined,
        width: undefined,
        height: undefined,
      };

      if (widthIsSticky) {
        newClipSize = {
          width: canvasRect.width,
          height: clip.height * canvasToClipRatioW,
          left: 0,
          top: clip.stickModes?.center
            ? canvasRect.height / 2 - clip.height / 2
            : oldZone.center.y + clip.delta.y * this.cellHeight - clip.height / 2,
        };
      } else if (heightIsSticky) {
        newClipSize = {
          width: clip.width * canvasToClipRatioH,
          height: canvasRect.height,
          left: clip.stickModes?.center
            ? canvasRect.width / 2 - clip.width / 2
            : oldZone.center.x + clip.delta.x * this.cellWidth - clip.width / 2,
          top: 0,
        };
      } else if (
        clip.stickModes?.left ||
        clip.stickModes?.right ||
        clip.stickModes?.top ||
        clip.stickModes?.bottom
      ) {
        if (clip.stickModes?.left) {
          newClipSize.left = 0;
        }

        if (clip.stickModes?.right) {
          newClipSize.left = canvasRect.width - clip.width;
        }

        if (clip.stickModes?.top) {
          newClipSize.top = 0;
        }

        if (clip.stickModes?.bottom) {
          newClipSize.top = canvasRect.height - clip.height;
        }
      } else if (clip.stickModes?.center) {
        newClipSize.left = canvasRect.width / 2 - clip.width / 2;
        newClipSize.top = canvasRect.height / 2 - clip.height / 2;
      }

      //clip is aligned to its zone by default

      if (newClipSize.left === undefined) {
        newClipSize.left = oldZone.center.x + clip.delta.x * this.cellWidth - clip.width / 2;
      }

      if (newClipSize.top === undefined) {
        newClipSize.top = oldZone.center.y + clip.delta.y * this.cellHeight - clip.height / 2;
      }

      /**
       * Fit text clips to canvas if they are out of canvas after resize
       */
      if (clip.mode === CLIP_MODE_TEXT) {
        if (newClipSize.left < 0) {
          newClipSize.left = STICKY_TOLERANCE;
        }

        if (newClipSize.top < 0) {
          newClipSize.top = STICKY_TOLERANCE;
        }

        if (newClipSize.width > canvasRect.width) {
          newClipSize.width = canvasRect.width - STICKY_TOLERANCE * 2;
        }

        if (newClipSize.height > canvasRect.height) {
          newClipSize.height = canvasRect.height - STICKY_TOLERANCE * 2;
        }
      }

      this.setClipSize(clip, newClipSize);
    },

    /**
     * Проверяет, совпадает ли положение и размеры клипа с какой-то из сторон канвы или её центром
     * и выставляет у клипа нужный флаг
     */
    calculateStickSides(clip) {
      console.warn("calculateStickSides>>");
      let leftIsSticky = Math.abs(clip.left) < STICKY_TOLERANCE;
      let rightIsSticky = Math.abs(this.width - clip.left - clip.width) < STICKY_TOLERANCE;
      let topIsSticky = Math.abs(clip.top) < STICKY_TOLERANCE;
      let bottomIsSticky = Math.abs(this.height - clip.top - clip.height) < STICKY_TOLERANCE;

      let centerIsSticky =
        Math.abs(this.height / 2 - (clip.top + clip.height / 2)) < STICKY_TOLERANCE * 3 &&
        Math.abs(this.width / 2 - (clip.left + clip.width / 2)) < STICKY_TOLERANCE * 3;

      clip.stickModes = {
        left: leftIsSticky,
        right: rightIsSticky,
        top: topIsSticky,
        bottom: bottomIsSticky,
        center: centerIsSticky,
      };

      //выводит информацию о прилипших сторонах поверх клипа
      {
        let stickSides = [];

        if (leftIsSticky && rightIsSticky) {
          stickSides.push("width");
          if (centerIsSticky) stickSides.push("center");
        } else if (topIsSticky && bottomIsSticky) {
          stickSides.push("height");
          if (centerIsSticky) stickSides.push("center");
        } else if (leftIsSticky || rightIsSticky || topIsSticky || bottomIsSticky) {
          if (leftIsSticky) {
            stickSides.push("left");
          }
          if (rightIsSticky) {
            stickSides.push("right");
          }
          if (topIsSticky) {
            stickSides.push("top");
          }
          if (bottomIsSticky) {
            stickSides.push("bottom");
          }
          stickSides.push("zone " + clip.zone);
        } else if (centerIsSticky) {
          stickSides.push("center");
        } else {
          stickSides.push("zone " + clip.zone);
        }

        clip.stickSide = stickSides.join("+");
      }
    },

    rescaleWithCanvas(clip, oldScale, newScale) {
      console.warn("rescaleWithCanvas>>");
      if (newScale !== oldScale) {
        let newWidth = clip.width * (1 / oldScale) * newScale;
        // clip.left = clip.left + clip.width / 2 - newWidth / 2;

        clip.width = newWidth;

        let newHeight = clip.height * (1 / oldScale) * newScale;
        // clip.top = clip.top + clip.height / 2 - newHeight / 2;

        clip.height = newHeight;
      }
    },

    /**
     * Растягивает клип в режиме заполнения всей канвы или подгонки по лучшей стороне
     * @param {*} mode Режим fit или fill
     */
    setFitFillMode(clip, mode) {
      const canvasRect = this.$refs.canvas.getBoundingClientRect();
      const width = clip.width;
      const height = clip.height;
      const canvasToClipRatioW = canvasRect.width / width;
      const canvasToClipRatioH = canvasRect.height / height;

      let rect = {};
      if (mode === RESIZE_MODE_FIT) {
        if (width * canvasToClipRatioH <= canvasRect.width) {
          rect = {
            width: width * canvasToClipRatioH,
            height: height * canvasToClipRatioH,
            top: 0,
            left: clip.left,
          };
        } else {
          rect = {
            width: width * canvasToClipRatioW,
            height: height * canvasToClipRatioW,
            top: clip.top,
            left: 0,
          };
        }
      } else if (mode === RESIZE_MODE_FILL) {
        // mode === fill
        if (width * canvasToClipRatioH <= canvasRect.width) {
          rect = {
            width: width * canvasToClipRatioW,
            height: height * canvasToClipRatioW,
            top: (canvasRect.height - height * canvasToClipRatioW) / 2,
            left: (canvasRect.width - width * canvasToClipRatioW) / 2,
          };
        } else {
          rect = {
            width: width * canvasToClipRatioH,
            height: height * canvasToClipRatioH,
            top: (canvasRect.height - height * canvasToClipRatioH) / 2,
            left: (canvasRect.width - width * canvasToClipRatioH) / 2,
          };
        }
      } else {
        throw new Error(`setFitFillMode: unknown fit mode: ${mode}`);
      }

      this.setClipSize(clip, rect);
    },

    /**
     * Устанавливаем размеры и положение клипа, координаты центра задаются автоматически по ним
     * rect.left, rect.top: required
     * rect.width, rect.height: optional
     * */
    setClipSize(clip, rect) {
      console.warn("setClipSize>>");
      if (rect?.width && rect?.width > 0) clip.width = rect.width;
      if (rect?.height && rect?.height > 0) clip.height = rect.height;

      clip.center = {
        x: rect.left + clip.width / 2,
        y: rect.top + clip.height / 2,
      };

      clip.left = rect.left;
      clip.top = rect.top;
    },

    /**
     * Рассчитываем сетку,  размеры ячеек, делим на зоны, считаем размер увеличенной центральной зоны
     * */
    initGridAndZones() {
      this.center.x = this.width / 2;
      this.center.y = this.height / 2;

      if (this.gridSize % 2 !== 1) console.error("gridSize: number must be odd");

      //calculate cell sizes

      const cellWidth = this.width / this.gridSize;
      const cellHeight = this.height / this.gridSize;

      this.cellWidth = cellWidth;
      this.cellHeight = cellHeight;

      //setup zones

      this.zones.splice(0, this.zones.length);
      this.grid.splice(0, this.grid.length);

      for (let zoneY = 0; zoneY < this.gridSize; zoneY++) {
        for (let zoneX = 0; zoneX < this.gridSize; zoneX++) {
          const zoneNumber = zoneX + zoneY * this.gridSize;

          //calculate zone right and bottom points
          this.zones[zoneNumber] = { right: 0, bottom: 0 };

          const left = (this.width / this.gridSize) * zoneX;
          const top = (this.height / this.gridSize) * zoneY;
          const right = left + cellWidth;
          const bottom = top + cellHeight;

          this.zones[zoneNumber].left = left;
          this.zones[zoneNumber].top = top;
          this.zones[zoneNumber].right = right;
          this.zones[zoneNumber].bottom = bottom;
          this.zones[zoneNumber].center = {
            x: left + cellWidth / 2,
            y: top + cellHeight / 2,
          };
          this.zones[zoneNumber].id = zoneNumber;

          //grid lines
          if (zoneY == 0 && zoneX < this.gridSize - 1) {
            this.grid.push({
              left: right + "px",
              top: "0px",
              height: "100%",
              width: "1px",
            });
          }

          if (zoneX == 0 && zoneY < this.gridSize - 1) {
            this.grid.push({
              left: "0px",
              top: bottom + "px",
              height: "1px",
              width: "100%",
            });
          }
        }
      }

      //setup and enlarge your center zone!
      const zoneCenterIndex = Math.floor(this.gridSize ** 2 / 2);
      const zoneCenter = this.zones[zoneCenterIndex];

      this.centerZone = {
        index: zoneCenterIndex,
        right: Math.floor(zoneCenter.right + cellWidth * this.centerZoneSnapFactor),
        bottom: Math.floor(zoneCenter.bottom + cellHeight * this.centerZoneSnapFactor),
        left: Math.floor(zoneCenter.right - cellWidth * (1 + this.centerZoneSnapFactor)),
        top: Math.floor(zoneCenter.bottom - cellHeight * (1 + this.centerZoneSnapFactor)),
        center: {
          x: zoneCenter.center.x,
          y: zoneCenter.center.y,
        },
      };

      this.centerZone.width = this.centerZone.right - this.centerZone.left;
      this.centerZone.height = this.centerZone.bottom - this.centerZone.top;
    },
  },

  mounted() {
    this.initGridAndZones();

    this.clips.forEach((clip) => {
      this.updateClipCenter(clip);
      this.calculateMatchedZone(clip);
      this.calculateStickSides(clip);
    });
  },
};
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
