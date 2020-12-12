<template>
  <div :style="{ background: '#f4eeed' }">
    <div class="settings">
      <form>
        <div class="form-row">
          <div class="col-md-6">
            <div class="form-group col-md-4">
              <label for="input-nrow">Rows</label>
              <input
                id="input-nrow"
                class="form-control"
                type="number"
                min="5"
                placeholder="9"
                v-model="nRow"
              />
            </div>
            <div class="form-group col-md-4">
              <label for="input-ncol">Columns</label>
              <input
                id="input-ncol"
                class="form-control"
                type="number"
                min="5"
                placeholder="9"
                v-model="nColumn"
              />
            </div>
            <!-- <div class="form-group col-md-4">
              <label for="input-ncol">Ghosts</label>
              <input
                id="input-ghosts"
                class="form-control"
                type="number"
                min="5"
                placeholder="9"
                v-model="ghosts"
              />
            </div> -->
          </div>
          <div class="form-group col-md-3">
            <label for="input-ncol"
              >Sensor Range - Near: {{ sensorRange.near }}</label
            >
            <input
              id="input-ghosts"
              class="form-control"
              type="range"
              min="1"
              :max="Math.max(nRow, nColumn)"
              placeholder="2"
              v-model="sensorRange.near"
            />
          </div>
          <div class="form-group col-md-3">
            <label for="input-ncol"
              >Sensor Range - Far: {{ sensorRange.far }}</label
            >
            <input
              id="input-ghosts"
              class="form-control"
              type="range"
              min="1"
              :max="Math.max(nRow, nColumn)"
              placeholder="3"
              v-model="sensorRange.far"
            />
          </div>
        </div>
      </form>
    </div>
    <div v-if="!isProcessing" class="main-game">
      <h4 style="font-weight: bold">
        Game Mode - {{ isCathcing ? "Catch Ghost Mode" : "Sensor Mode" }}
      </h4>
      <table class="game-board">
        <tbody>
          <tr
            v-for="rowNo in Number(nRow)"
            :key="'row-' + rowNo + '-' + Date.now()"
          >
            <td
              v-for="colNo in Number(nColumn)"
              :key="'col-' + colNo + '-' + Date.now()"
              class="game-cell"
              style="background: rgb(1, 23, 99); color: #ffd369"
              :style="
                hasUpdated && isLastClickedCells(rowNo - 1, colNo - 1)
                  ? getCellStyle(rowNo - 1, colNo - 1)
                  : {}
              "
              @click="handleCellClick(rowNo - 1, colNo - 1)"
            >
              <span
                :style="{ display: 'block', color: '#ffd369' }"
                v-if="showGhosts && checkGhost(rowNo - 1, colNo - 1)"
              >
                <i class="fab fa-snapchat-ghost fa-2x" aria-hidden="true"></i>
              </span>
              <span>{{
                (100 * boardProbs[rowNo - 1][colNo - 1]).toFixed(2)
              }}</span>
            </td>
          </tr>
        </tbody>
      </table>
      <div class="control-buttons">
        <button class="btn btn-lg btn-primary" @click="advanceTime">
          <i class="fa fa-forward" aria-hidden="true"></i> Advance
        </button>
        <button
          class="btn btn-lg"
          :class="{ 'btn-success': isCathcing, 'btn-primary': !isCathcing }"
          @click="isCathcing = !isCathcing"
        >
          <i
            class="fab"
            :class="{
              'fa-snapchat-square': isCathcing,
              'fa-snapchat-ghost': !isCathcing,
            }"
            aria-hidden="true"
          ></i>
          {{ isCathcing ? "Find" : "Catch" }}
        </button>
        <button
          class="btn btn-lg"
          :class="{ 'btn-success': showGhosts, 'btn-primary': !showGhosts }"
          @click="showGhosts = !showGhosts"
        >
          <i
            class="fa"
            :class="{
              'fa-eye': !showGhosts,
              'fa-eye-slash': showGhosts,
            }"
            aria-hidden="true"
          ></i>
          {{ showGhosts ? "Hide" : "Show" }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "Index",
  head() {
    return {
      title: "HMM Offline - Ghost Finder",
    };
  },
  created() {
    this.initializeGame();
  },
  watch: {
    nRow: function () {
      this.initializeGame();
    },
    nColumn: function () {
      this.initializeGame();
    },
    sensorRange: function () {
      this.initializeGame();
    },
  },
  data() {
    return {
      nRow: 9,
      nColumn: 9,
      ghosts: 1,
      moveProbs: {
        mid: {
          side: 0.24, // 4 sides
          diag: 0.009, // 4 diags
          stay: 0.004, // 1 hold
        },
        edge: {
          side: 0.32, // 3 sides
          diag: 0.018, // 2 diags
          stay: 0.004, // 1 hold
        },
        corner: {
          side: 0.48, // 2 sides
          diag: 0.036, // 1 diags
          stay: 0.004, // 1 hold
        },
      },
      sensorRange: {
        near: 2,
        far: 3,
      },
      boardProbs: [[]],
      ghostsPos: [[]],
      isProcessing: true,
      hasUpdated: true,
      isCathcing: false,
      showGhosts: false,
      lastClickedCells: [], // {x, y, proximity} prox -> 0: near, 1: mid, 2: far
    };
  },
  methods: {
    initializeGame() {
      this.isProcessing = true;
      const totalCells = Number(this.nRow) * Number(this.nColumn);
      this.boardProbs = Array(Number(this.nRow))
        .fill(0)
        .map((row) => Array(Number(this.nColumn)).fill(1 / totalCells));
      this.ghostsPos = [];
      for (let i = 0; i < Number(this.ghosts); ++i) {
        this.ghostsPos.push({
          x: Math.floor(Math.random() * Number(this.nRow)),
          y: Math.floor(Math.random() * Number(this.nColumn)),
        });
      }
      this.hasUpdated = true;
      this.isCathcing = false;
      this.showGhosts = false;
      this.lastClickedCells = [];
      this.isProcessing = false;
    },
    checkGhost(rowId, colId) {
      for (let gPos of this.ghostsPos) {
        if (gPos.x == rowId && gPos.y == colId) return true;
      }
      return false;
    },
    getCellLocationClass(x, y) {
      // mid , edge, corner
      if (
        (x == 0 || x == Number(this.nRow) - 1) &&
        (y == 0 || y == Number(this.nColumn) - 1)
      )
        return "corner";
      else if (
        ((x == 0 || x == Number(this.nRow) - 1) &&
          y > 0 &&
          y < Number(this.nColumn) - 1) ||
        (x > 0 &&
          x < Number(this.nColumn) - 1 &&
          (y == 0 || y == Number(this.nColumn) - 1))
      )
        return "edge";
      else return "mid";
    },
    propagateProbability() {
      let tempBoardProbs = Array(this.nRow)
        .fill(0)
        .map((row) => Array(this.nColumn).fill(0.0));

      for (let i = 0; i < Number(this.nRow); ++i) {
        for (let j = 0; j < Number(this.nColumn); ++j) {
          if (this.boardProbs[i][j] > 0) {
            const currentMoveProbs = this.moveProbs[
              this.getCellLocationClass(i, j)
            ];
            const currentCellProb = this.boardProbs[i][j];
            // side, diag, stay
            // for i-1
            if (i - 1 >= 0) {
              // for j-1
              if (j - 1 >= 0) {
                tempBoardProbs[i - 1][j - 1] +=
                  currentMoveProbs.diag * currentCellProb;
              }
              // for j
              tempBoardProbs[i - 1][j] +=
                currentMoveProbs.side * currentCellProb;
              // for j+1
              if (j + 1 < Number(this.nColumn)) {
                tempBoardProbs[i - 1][j + 1] +=
                  currentMoveProbs.diag * currentCellProb;
              }
            }
            // for i
            {
              // for j-1
              if (j - 1 >= 0) {
                tempBoardProbs[i][j - 1] +=
                  currentMoveProbs.side * currentCellProb;
              }
              // for j
              tempBoardProbs[i][j] += currentMoveProbs.stay * currentCellProb;
              // for j+1
              if (j + 1 < Number(this.nColumn)) {
                tempBoardProbs[i][j + 1] +=
                  currentMoveProbs.side * currentCellProb;
              }
            }
            // for i+1
            if (i + 1 < Number(this.nRow)) {
              // for j-1
              if (j - 1 >= 0) {
                tempBoardProbs[i + 1][j - 1] +=
                  currentMoveProbs.diag * currentCellProb;
              }
              // for j
              tempBoardProbs[i + 1][j] +=
                currentMoveProbs.side * currentCellProb;
              // for j+1
              if (j + 1 < Number(this.nColumn)) {
                tempBoardProbs[i + 1][j + 1] +=
                  currentMoveProbs.diag * currentCellProb;
              }
            }
          }
        }
      }

      this.boardProbs = tempBoardProbs;
    },
    getShortestDistanceFromGhosts(x, y) {
      let gDist = Number(this.nRow) + Number(this.nColumn);
      for (let gPos of this.ghostsPos) {
        gDist = Math.min(gDist, Math.abs(x - gPos.x) + Math.abs(y - gPos.y));
      }
      return gDist;
    },
    getCellStyle(x, y) {
      const isClicked = this.isLastClickedCells(x, y);
      if (isClicked >= 0) {
        const cellProx = this.lastClickedCells[isClicked].proximity;
        if (cellProx == 0) {
          return { background: "red", color: "white" };
        } else if (cellProx == 1) {
          return { background: "orange", color: "black" };
        } else if (cellProx == 2) {
          return { background: "green", color: "white" };
        }
      }

      return {
        background: "rgb(1, 23, 99)",
        color: "#ffd369",
      };
    },
    normalizeProbability(x, y, proximity) {
      const nearRange = Number(this.sensorRange.near);
      const midRange = nearRange + Number(this.sensorRange.far);
      let probableCellLoc = [];
      for (let i = 0; i < Number(this.nRow); ++i) {
        for (let j = 0; j < Number(this.nColumn); ++j) {
          if (!this.boardProbs[i][j]) continue;
          const distance = Math.abs(x - i) + Math.abs(y - j);

          if (proximity == 0) {
            if (distance <= nearRange) {
              probableCellLoc.push({ x: i, y: j });
            }
          } else if (proximity == 1) {
            if (distance > nearRange && distance <= midRange) {
              probableCellLoc.push({ x: i, y: j });
            }
          } else if (proximity == 2) {
            if (distance > midRange) {
              probableCellLoc.push({ x: i, y: j });
            }
          }
        }
      }
      if (probableCellLoc.length > 0) {
        this.boardProbs = Array(this.nRow)
          .fill(0)
          .map((row) => Array(this.nColumn).fill(0.0));
        for (let cellLoc of probableCellLoc) {
          this.boardProbs[cellLoc.x][cellLoc.y] = 1.0 / probableCellLoc.length;
        }
      }
    },
    moveGhosts() {
      for (let ghostNo = 0; ghostNo < this.ghostsPos.length; ++ghostNo) {
        const moveType = [
          Math.random() * 0.96, // side - 0
          Math.random() * 0.036, // diag - 1
          Math.random() * 0.004, // stay - 2
        ].reduce((a, b, i) => (a[0] < b ? [b, i] : a), [
          Number.MIN_VALUE,
          -1,
        ])[1];
        if (moveType == 2) continue;
        else if (moveType == 0) {
          let moves = [];
          if (this.ghostsPos[ghostNo].x - 1 >= 0)
            moves.push({
              x: this.ghostsPos[ghostNo].x - 1,
              y: this.ghostsPos[ghostNo].y,
            });
          if (this.ghostsPos[ghostNo].y - 1 >= 0)
            moves.push({
              x: this.ghostsPos[ghostNo].x,
              y: this.ghostsPos[ghostNo].y - 1,
            });
          if (this.ghostsPos[ghostNo].x + 1 < Number(this.nRow))
            moves.push({
              x: this.ghostsPos[ghostNo].x + 1,
              y: this.ghostsPos[ghostNo].y,
            });
          if (this.ghostsPos[ghostNo].y + 1 < Number(this.nColumn))
            moves.push({
              x: this.ghostsPos[ghostNo].x,
              y: this.ghostsPos[ghostNo].y + 1,
            });

          let movesProb = [];
          for (let i = 0; i < moves.length; ++i) {
            movesProb.push(Math.random());
          }
          const selectedMove = movesProb.reduce(
            (a, b, i) => (a[0] < b ? [b, i] : a),
            [Number.MIN_VALUE, -1]
          )[1];
          this.ghostsPos[ghostNo] = moves[selectedMove];
        } else if (moveType == 1) {
          let moves = [];
          if (
            this.ghostsPos[ghostNo].x - 1 >= 0 &&
            this.ghostsPos[ghostNo].y - 1 >= 0
          )
            moves.push({
              x: this.ghostsPos[ghostNo].x - 1,
              y: this.ghostsPos[ghostNo].y - 1,
            });
          if (
            this.ghostsPos[ghostNo].x - 1 >= 0 &&
            this.ghostsPos[ghostNo].y + 1 < Number(this.nColumn)
          )
            moves.push({
              x: this.ghostsPos[ghostNo].x - 1,
              y: this.ghostsPos[ghostNo].y + 1,
            });
          if (
            this.ghostsPos[ghostNo].x + 1 < Number(this.nRow) &&
            this.ghostsPos[ghostNo].y - 1 >= 0
          )
            moves.push({
              x: this.ghostsPos[ghostNo].x + 1,
              y: this.ghostsPos[ghostNo].y - 1,
            });
          if (
            this.ghostsPos[ghostNo].x + 1 < Number(this.nRow) &&
            this.ghostsPos[ghostNo].y + 1 < Number(this.nColumn)
          )
            moves.push({
              x: this.ghostsPos[ghostNo].x + 1,
              y: this.ghostsPos[ghostNo].y + 1,
            });

          let movesProb = [];
          for (let i = 0; i < moves.length; ++i) {
            movesProb.push(Math.random());
          }
          const selectedMove = movesProb.reduce(
            (a, b, i) => (a[0] < b ? [b, i] : a),
            [Number.MIN_VALUE, -1]
          )[1];
          this.ghostsPos[ghostNo] = moves[selectedMove];
        }
      }
    },
    advanceTime() {
      this.moveGhosts();
      if (this.showGhosts) {
        this.showGhosts = false;
        this.showGhosts = true;
      }
      this.lastClickedCells = [];
      this.propagateProbability();
    },
    isLastClickedCells(x, y) {
      for (let i = 0; i < this.lastClickedCells.length; ++i) {
        if (
          this.lastClickedCells[i].x === x &&
          this.lastClickedCells[i].y === y
        ) {
          return i;
        }
      }
      return -1;
    },
    handleCellClick(x, y) {
      if (this.isCathcing) {
        if (this.checkGhost(x, y)) {
          alert("Caught The Ghost!");
          this.initializeGame();
        } else {
          alert("Game Over");
        }
      } else {
        if (this.isLastClickedCells(x, y) >= 0) return;
        let dist = this.getShortestDistanceFromGhosts(x, y);
        let proximity = 2;
        const nearRange = Number(this.sensorRange.near);
        const midRange = nearRange + Number(this.sensorRange.far);
        if (dist <= nearRange) proximity = 0;
        else if (dist <= midRange) proximity = 1;
        this.lastClickedCells.push({ x, y, proximity });
        this.hasUpdated = false;
        this.hasUpdated = true;
        this.normalizeProbability(x, y, proximity);
      }
    },
  },
};
</script>

<style>
* {
  font-family: Roboto, sans-serif;
}

.settings {
  margin: 0 auto;
  min-height: 12vh;
  display: flex;
  flex-flow: column;
  justify-content: flex-end;
  align-items: center;
  text-align: center;
}

.main-game {
  margin: 0 auto;
  min-height: 88vh;
  display: flex;
  flex-flow: column;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.game-board {
  margin-top: 2rem;
  margin-bottom: 1.5rem;
}

.game-cell {
  height: 60px;
  width: 60px;
  border: 3px solid #eeeeee;
  cursor: pointer;
  font-weight: bold;
}

.control-buttons {
  width: 25vw;
  display: flex;
  justify-content: space-between;
  margin-top: 2rem;
}
</style>
