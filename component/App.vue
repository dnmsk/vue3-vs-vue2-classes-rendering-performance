<template>
  <div class="life-game" ref="game_container">
    <div class="life-game__squares-container">
      <div
        v-for="(rowSquares, rowId) in array.squares"
        :key="`row_${rowId}`"
        class="life-game__square-row"
        :class="{'life-game__square-row_first': rowId == 0}"
      >
        <div
          v-for="(square, cellId) in rowSquares"
          :key="`cell_${rowId}_${cellId}`"
          :class="{'life-game__live-square': square.l, 'life-game__square_first': cellId == 0}"
          class="life-game__square"
          @mousedown="mouseDown(rowId, cellId)"
          @mouseup="mouseUp(rowId, cellId)"
          @mouseover="mouseOver(rowId, cellId)"
        />
      </div>
    </div>
    <div class="life-game__controls">
      <div
        class="life-game__run-btn"
        @click="runLoops"
      >
        {{canWork ? 'stop' : 'run'}}
      </div>
      <div>Epoch: {{ stat.epoch }}</div>
      <div>Live: {{ stat.live }}</div>
      <div>eps: {{ stat.avgEps }}</div>
    </div>
  </div>
</template>

<script>
  const lookups = [-1, 0, 1];

  export default {
    components: {},
    data() {
      let cellSize = 8;
      return {
        container: {
          height: 0,
          width: 0,
          cellSize,
        },
        array: {
          squares: [[]],
          rows: 0,
          cols: 0,
        },
        squaresArray: [],
        hashLastStates: [],
        hashLastStatesCnt: 16,
        hashMaxSameSteps: 3,
        epoch: 0,
        isMouseDown: false,
        canWork: false,
        stat: {
          live: 0,
          epoch: 0,
          frameStat: [],
          frameStatMaxQty: 5,
          avgEps: 0,
        },
      };
    },
    methods: {
      runEpoch() {
        const changes = [];
        const totalRows = this.array.rows;
        const totalCols = this.array.cols;
        const lookUpBackRowIndex = totalRows - 1;
        const lookUpBackCellIndex = totalCols - 1;

        if (!this.nativeArray) {
          this.nativeArray = [];
          this.array.squares.forEach((row, rowId) => {
            const nativeRow = this.nativeArray[rowId] = [];

            row.forEach((cell, cellId) => {
              nativeRow[cellId] = { ...cell };
            });
          });
        }
        const nativeArray = this.nativeArray;
        nativeArray.forEach((row, rowId) => {
          row.forEach((cell, cellId) => {
            let liveCnt = 0;
            lookups.forEach(rowLookup => {
              lookups.forEach(cellLookup => {
                let lookupRowId = rowId + rowLookup;
                let lookupCellId = cellId + cellLookup;
                if (lookupRowId < 0) {
                  lookupRowId = lookUpBackRowIndex;
                }
                if (lookupRowId >= totalRows) {
                  lookupRowId = 0;
                }
                if (lookupCellId < 0) {
                  lookupCellId = lookUpBackCellIndex;
                }
                if (lookupCellId >= totalCols) {
                  lookupCellId = 0;
                }
                const otherCell = nativeArray[lookupRowId][lookupCellId];

                if (otherCell != cell && otherCell.l) {
                  liveCnt++;
                }
              });
            });
            if (cell.l) {
              if (liveCnt > 3 || liveCnt < 2) {
                changes.push([rowId, cellId]);
              }
            } else {
              if (liveCnt == 3) {
                changes.push([rowId, cellId]);
              }
            }
          });
        });
        let liveStatDelta = 0;
        changes.forEach(change => {
          const r = this.array.squares[change[0]][change[1]].l =
            this.nativeArray[change[0]][change[1]].l ^= 1;
          liveStatDelta += r ? 1 : -1;
        });
        this.stat.live += liveStatDelta;
        return changes;
      },
      runLoops() {
        this.canWork ^= true;
        const f = () => {
          if (this.canWork) {
            if (!this.isMouseDown) {
              const startedAt = new Date();
              const r = this.runEpoch();
              this.stat.avgEps = this.getAvgEps(new Date() - startedAt);

              const s = r.reduce((acc, v) => acc + `${v[0]}:${v[1]};`, '');
              this.hashLastStates.unshift(s);
              if (this.hashLastStates.length > this.hashLastStates.length) {
                this.hashLastStates.pop();
              }
              let foundSame = 0;
              for(let i = this.hashMaxSameSteps; i < this.hashLastStates.length && foundSame < this.hashMaxSameSteps; i++) {
                foundSame = 0;
                for(let j = 0; j + i < this.hashLastStates.length && j < this.hashMaxSameSteps; j++) {
                  if (this.hashLastStates[j] == this.hashLastStates[i + j]) {
                    foundSame++;
                    continue;
                  }
                  break;
                }
              }
              this.canWork = foundSame < this.hashMaxSameSteps;
              if (r.length) {
                this.stat.epoch++;
              }
            }
            setTimeout(f, 0);
          }
        };
        f();
      },
      getAvgEps(ellapsedMs) {
        this.stat.frameStat[this.stat.frameStat.length % this.stat.frameStatMaxQty] = ellapsedMs;
        const fpsFloat = 1000 / (this.stat.frameStat.reduce((a, b) => a + b, 0) / this.stat.frameStat.length);
        return parseInt(fpsFloat * 100) / 100;
      },
      mouseDown(rowId, cellId) {
        this.isMouseDown = true;
        const r = this.array.squares[rowId][cellId].l ^= 1;
        this.stat.live += r ? 1 : -1;
      },
      mouseOver(rowId, cellId) {
        if (this.isMouseDown) {
          this.mouseDown(rowId, cellId);
        }
      },
      mouseUp() {
        this.isMouseDown = false;
      },
      windowResize() {
        const l = false;
        let gameContainer = this.$refs.game_container;
        const newCols = parseInt(gameContainer.clientWidth / this.container.cellSize)
          + (gameContainer.clientWidth % this.container.cellSize > 0 ? 1 : 0);
        const newRows = parseInt(gameContainer.clientHeight / this.container.cellSize)
          + (gameContainer.clientHeight % this.container.cellSize > 0 ? 1 : 0);
        if (this.array.rows < newRows) {
          while (this.array.squares.length < newRows) {
            this.array.squares.push(new Array(this.array.cols).fill(0, {l}));
          }
          this.array.rows = newRows;
        }
        if (this.array.cols < newCols) {
          this.array.squares.forEach(row => {
            while (row.length < newCols) {
              row.push({l});
            }
          });
          this.array.cols = newCols;
        }
        this.nativeArray = null;
      },
    },
    mounted() {
      this.windowResize();
      const initialState = [
        [0, 1, 1],
        [1, 1, 0],
        [0, 1, 0],
      ];
      const fromX = parseInt(this.array.cols / 2) - parseInt(initialState.length / 2);
      const fromY = parseInt(this.array.rows / 2) - parseInt(initialState[0].length / 2);
      initialState.forEach((row, rowId) => {
        row.forEach((cell, cellId) => {
          if (initialState[rowId][cellId]) {
            this.mouseDown(fromY + rowId, fromX + cellId);
          }
        });
      });
      this.mouseUp();
    },
  }
</script>

<style lang="scss">
* {
  box-sizing: border-box;
}
html, body, #app {
  height: 100%;
  margin: 0;
  padding: 0;
}

.life-game {
  height: 100%;
  position: relative;

  &__controls {
    background-color: white;
    border-radius: 1rem;
    border: 1px solid black;
    padding: 1rem;
    position: absolute;
    right: 2rem;
    top: 2rem;
  }

  &__run-btn {
    cursor: pointer;
    padding: 0.5rem 2rem;
    border-radius: 1rem;
    border: 1px solid black;
  }

  &__squares-container {
    border: 1px solid gray;
    bottom: 0;
    left: 0;
    overflow: hidden;
    position: absolute;
    right: 0;
    top: 0;
  }

  &__square-row {
    font-size: 0;
    white-space: nowrap;

    &_first {
      .life-game__live-square {
        border-top: 1px solid gray;
      }
    }
  }

  &__square {
    display: inline-block;
    height: 8px;
    width: 8px;

    &_first {
      .life-game__live-square {
        border-left: 1px solid gray;
        width: 9px;
      }
    }
  }

  &__live-square {
    background-color: black;
    border-bottom: 1px solid gray;
    border-right: 1px solid gray;

  }
}
</style>
