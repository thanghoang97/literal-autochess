<template>
  <div class="gameview">
    <span
      ref="board"
      class="board outlines"
      :style="{
      'grid-template-columns': `repeat(${dimensions.x}, ${columnWidth}px)`, 
      'grid-template-rows': `repeat(${dimensions.y}, ${columnWidth}px)`,
    }"
    >
      <div
        v-for="i in dimensions.x * dimensions.y"
        :key="'outline'+i"
        :class="{shade: shouldShadeGridElement(i)}"
      ></div>
    </span>
    <template v-if="gameData">
      <Stats
        class="statspane"
        :gameData="gameData"
        :playbackPosition="playbackPosition"
        :currentEvent="currentEvent"
      />

      <WinBanner
        v-if="winner"
        :winner="winner"
        :player="player"
        :gameData="gameData"
        @playbackPosition="setPlaybackPosition"
        @next="$emit('next')"
      />

      <transition-group
        class="board"
        name="grid"
        :style="{
          'grid-template-columns': `repeat(${dimensions.x}, ${columnWidth}px)`, 
          'grid-template-rows': `repeat(${dimensions.y}, ${columnWidth}px)`,
        }"
      >
        <Piece
          v-for="piece, index in currentGameState"
          :key="piece.id"
          :x="piece.x"
          :y="piece.y"
          :indicator="piece.indicator"
          :type="piece.type"
          :color="piece.color"
          :id="piece.id"
          :moving="movingId === piece.id"
          :damage="damageId === piece.id"
        />
      </transition-group>

      <div
        v-for="popover in activePopovers"
        :key="popover.id"
        class="eventpopover"
        :style="{top: popover.y * (100/dimensions.y) + (50/dimensions.y) + '%', left:popover.x * (100/dimensions.x) + (50/dimensions.x) + '%' }"
      >{{popover.amount}}</div>
    </template>

    <template v-else>
      <span
        class="board"
        :class="{dropzone: dragging}"
        ref="board"
        :style="{
          'grid-template-columns': `repeat(${dimensions.x}, ${columnWidth}px)`, 
          'grid-template-rows': `repeat(${dimensions.y}, ${columnWidth}px)`,
        }"
      >
        <Piece
          v-for="piece, index in enemy.pieces"
          :key="piece.id"
          :x="piece.x"
          :y="piece.y"
          :indicator="piece.indicator"
          :type="piece.type"
          :color="piece.color"
          :id="piece.id"
        />
        <MovablePiece
          v-for="piece, index in player.pieces"
          :key="piece.id"
          :x="piece.x"
          :y="piece.y"
          :indicator="piece.indicator"
          :type="piece.type"
          :color="piece.color"
          :id="piece.id"
        />
      </span>
    </template>

    <Bench
      ref="bench"
      :dimensions="dimensions"
      :player="player"
      :gameData="gameData"
      :columnWidth="columnWidth"
    />

    <Shop ref="shop" class="shopholder" :gameData="gameData" :player="player" @notify="notify" />
    <DropWatcher
      :elementsToWatch="[
        {name: 'board', el: $refs.board}, 
        {name: 'bench', el: $refs.bench}, 
        {name: 'shop', el: $refs.shop}
      ]"
      @drop="handleDrop"
    />
  </div>
</template>

<script>
import Piece from '~/components/Piece'
import MovablePiece from '~/components/MovablePiece'
import Stats from '~/components/Stats'
import firestore from '~/assets/firestore'
import Shop from '~/components/Shop'
import Bench from '~/components/Bench'
import WinBanner from '~/components/WinBanner'
import DropWatcher from '~/components/DropWatcher'

export default {
  components: {
    Piece,
    MovablePiece,
    Stats,
    Shop,
    Bench,
    WinBanner,
    DropWatcher,
  },
  props: {
    dimensions: {
      type: Object,
      default: () => ({ x: 8, y: 8 }),
    },
    gameData: {
      type: Array,
      default: () => [],
    },
    player: {},
    enemy: {},
    gamePixelWidth: {},
  },
  data() {
    return {
      autoSpeed: 250,
      currentSpeed: 1,
      columnWidth: this.gamePixelWidth / this.player.dimensions.x,
      playbackPosition: -1,
      activePopovers: [],
      winner: null,
      auto: true,
      autoInterval: null,
      movingId: null,
      damageId: null,
      isScrubbing: false,
      levelRatio: 100,
    }
  },
  computed: {
    dragging() {
      return !!this.$store.state.draggingPiece
    },
    currentGameState() {
      if (!this.gameData || !this.gameData.length) return
      return JSON.parse(this.gameData[this.playbackPosition].gameState)
    },
    currentEvent() {
      if (
        !this.gameData ||
        !this.gameData.length ||
        !this.gameData[this.playbackPosition]
      )
        return
      let newEvent = JSON.parse(this.gameData[this.playbackPosition].event)
      if (!Object.keys(newEvent).length) return
      return newEvent
    },
  },
  watch: {
    async winner() {
      this.levelRatio = parseInt(
        (await firestore.levelRatio(this.player.level)) * 100
      )
    },
    gameData() {
      this.playbackPosition = -1
      this.winner = null
      this.auto = true
      this.movingId = null
      this.damageId = null
      this.currentSpeed = this.autoSpeed
      this.advance()
    },
    currentEvent(newEvent) {
      if (!newEvent) return
      this.movingId = null
      this.damageId = null
      if (['damage', 'kill', 'win'].includes(newEvent.type)) {
        this.movingId = newEvent.from.id
        this.damageId = newEvent.to.id
        // this.activePopovers.push({
        //   x: newEvent.to.x,
        //   y: newEvent.to.y,
        //   amount: newEvent.amount,
        //   id: `${Math.random()}`.substring(2),
        // })
        // setTimeout(() => this.activePopovers.shift(), 1000)
      }
      if (newEvent.attackInPlace) {
        // console.log('nomove')
      }
      if (newEvent.type === 'move') {
        this.movingId = newEvent.piece.id
      }
      if (newEvent.type === 'end') {
        this.movingId = newEvent.from.id
        if (this.winner) return
        this.winner = newEvent.winner
        this.$emit('playbackFinished', newEvent.winner === 'black')
      }
      if (newEvent.type === 'stalemate') {
        this.movingId = null
        if (this.winner) return
        this.winner = 'stalemate'
        this.$emit('playbackFinished')
      }
    },
    playbackPosition() {
      this.$emit('playbackPosition', this.playbackPosition)
    },
  },
  methods: {
    setPlaybackPosition(newPosition) {
      this.playbackPosition = newPosition
    },
    advance() {
      if (!this.gameData || this.winner) return
      this.currentSpeed =
        JSON.parse(this.gameData[this.gameData.length - 1].event).type ===
        'stalemate'
          ? //gradually speed up
            this.autoSpeed * 0.6 -
            this.autoSpeed *
              0.6 *
              ((this.playbackPosition / this.gameData.length) * 0.65)
          : //gradually slow down
            this.autoSpeed +
            this.autoSpeed *
              ((this.playbackPosition / this.gameData.length) * 0.5)

      setTimeout(this.advance, this.currentSpeed)
      if (this.playbackPosition < this.gameData.length - 1) {
        this.playbackPosition++
      }
    },

    handleDrop({ droppedEl, piece, xPercent, yPercent }) {
      if (!droppedEl || droppedEl.name === 'bench')
        this.$emit('sendToBench', piece.id)
      else if (droppedEl.name === 'board') {
        const newX = Math.round(this.dimensions.x * xPercent - 0.5)
        const newY = Math.round(this.dimensions.y * yPercent - 0.5)

        if (piece.bench === false)
          this.$emit('updatePiece', {
            homeX: newX,
            homeY: newY,
            id: piece.id,
          })
        else
          this.$emit('playFromBench', {
            x: newX,
            y: newY,
            id: piece.id,
          })
      } else if (droppedEl.name === 'shop') {
        this.$emit('sell', { id: piece.id, type: piece.type })
      }
    },

    shouldShadeGridElement(index) {
      if (this.dimensions.x % 2 === 1) return index % 2 === 1
      //even number
      const repeat = this.dimensions.x * 2
      while (index > repeat) index -= repeat
      if (index <= this.dimensions.x) return index % 2 === 0
      return index % 2 === 1
    },
    notify(notification) {
      this.$emit('notify', notification)
    },
  },
  mounted() {
    setTimeout(this.advance, this.currentSpeed)
  },
}
</script>

<style lang="scss" scoped>
.gameview {
  position: relative;

  .board {
    position: relative;
    display: grid;
    z-index: 1;
    transition: background 0.2s;
    background: var(--bg-shade);

    &.dropzone {
      background: linear-gradient(
        to bottom,
        var(--bg-fade) 50%,
        var(--highlight-light) 50%
      );
    }

    &.outlines {
      background: none;
      position: absolute;
      pointer-events: none;
      z-index: 0;
      border: none;

      div {
        border: 1px solid var(--bg-shade);

        &.shade {
          background: var(--bg-shade2);
        }
      }
    }
  }
}

.bench {
  margin-top: 10px;
  position: relative;
  display: grid;
  z-index: 1;
  transition: background 0.2s;
  background: var(--bg-shade);

  &.dropzone {
    background: var(--highlight-light);
  }
}

.shopholder {
  margin-top: 10px;

  & > * {
    width: 100%;
  }
}

.statspane {
  position: absolute;
  top: 0;
  left: 110%;
}

.nointeract {
  pointer-events: none;
  opacity: 0.6;
}

.eventpopover {
  font-size: 1.4em;
  font-weight: 800;
  position: absolute;
  color: var(--damage);
  transform: translateX(-50%) translateY(-50%);
  z-index: 3;
  animation-name: fadeup;
  animation-duration: 1s;
}

.winbanner {
  text-align: center;
  font-size: 1.5rem;
  position: absolute;
  top: 43%;
  left: 50%;
  background: rgba(white, 0.95);
  transform: translateX(-50%) translateY(-50%);
  padding: 20px 30px;
  z-index: 5;
  min-width: 85%;
  transition: all 0.2s;

  &.fade {
    opacity: 0.3;
  }
}
.scrubber {
  padding: 15px 0;
  width: 100%;
}

@keyframes fadeup {
  from {
    transform: translateX(-50%) translateY(-50%);
  }
  to {
    transform: translateX(-50%) translateY(-350%);
    opacity: 0;
  }
}

.grid-move {
  transition: transform 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  z-index: 2;
}
.grid-leave-active {
  display: none;
}
</style>
