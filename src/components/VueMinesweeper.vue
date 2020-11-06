<template>
  <div class="game-wrapper">
    <div 
      class="game-board"
      :style="`width: ${30 * gameBoardSize + 6}px`"
    >
      <div class="infoPanel">
        <div 
          v-if="gameOver"
          class="game-over"
        >
          GAME OVER
          <v-btn
            class="ma-2"
            color="primary"
            size="small"
            @click="chooseDifficulty"
            v-text="'new game'"
          />
        </div>
        <div 
          v-else-if="selectingDifficulty"
        >
          <v-btn
            class="ma-2"
            color="primary"
            size="small"
            @click="newGame('easy')"
            v-text="'easy'"
          />

          <v-btn
            class="ma-2"
            color="primary"
            size="small"
            @click="newGame('normal')"
            v-text="'normal'"
          />

          <v-btn
            class="ma-2"
            color="primary"
            size="small"
            @click="newGame('hard')"
            v-text="'hard'"
          />

          <v-btn
            class="ma-2"
            color="primary"
            size="small"
            @click="newGame('impossible')"
            v-text="'impossible'"
          />
        </div>

        

        <div 
          v-else-if="gameWin"
          class="game-win"
        >
          YOU WIN!!!
          <div>
            <span class="timeElapsed">{{ `Score: ${timeElapsed}` }}</span>
            <v-btn
              class="ma-2"
              color="primary"
              size="small"
              @click="chooseDifficulty"
              v-text="'new game'"
            />
          </div>
        </div>

        <div v-else>
          <div>Total Mines: <span class="numberOfMines">{{ `${totalMines}` }}</span></div>
          <div>Time Elapsed:  <span class="timeElapsed">{{ `${timeElapsed}` }}</span></div>
        </div>
      </div>

      <div class="grid">
        <div 
          v-for="(row, y) in gameBoard"
          :key="`row-${y}`"
          class="row"
        >
          <div 
            v-for="(block, x) in row"
            :key="`block-${y}-${x}`"
            class="block"
            @click="clickBlock(x,y)"
          >
            <div 
              v-if="!block.isVisible"
              class="hiddenBlock"
              @click.right="toggleFlagBlock(x,y)"
              @contextmenu.prevent
            >
              <div 
                v-show="block.isFlagged"
                class="flag"
              >
                🚩
              </div>
            </div>
            <div 
              v-else-if="block.isMine"
              class="mine"
              :style="`background: ${mineStyle(x, y)}`"
            >
              💣
            </div>

            <div 
              v-else-if="block.minesInProximity"
              :style="`color: ${findColor(block)}`"
            >
              {{ block.minesInProximity }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>

export default {
  name: 'BridgeSweeper',
  data() {
    return {
      gameBoard: [],
      initialMove: true,
      gameBoardSize: 5,
      totalMines: 0,
      blocksRevealed: 0,
      gameOver: false,
      gameWin: false,
      selectingDifficulty: false,
      explodedMine: {
        x: null,
        y: null,
      },
      difficulty: {
        easy: 8,
        normal: 11,
        hard: 15,
        impossible: 20,
      },
      selectedDifficulty: 'easy',
      timeElapsed: 0,
      timer: null,
    };
  },
  computed: {
    totalBlocks() {
      return this.gameBoardSize**2;
    },
    blocksRemaining() {
      return (this.totalBlocks - this.totalMines - this.blocksRevealed);
    },
  },
  created() {
    this.gameBoardSize = Math.floor(window.innerHeight / 45);
    this.generateGrid();
  },
  methods: {
    startTimer() {
      this.timer = setInterval(this.incrementTimer, 1000);
    },
    incrementTimer() {
      this.timeElapsed++;
    },
    stopTimer() {
      clearInterval(this.timer);
    },
    clearTimer() {
      this.timeElapsed = 0;
    },
    toggleFlagBlock(x, y) {
      this.gameBoard[y][x].isFlagged = !this.gameBoard[y][x].isFlagged;
    },
    chooseDifficulty() {
      this.gameOver = false;
      this.selectingDifficulty = true;
    },
    newGame(difficulty) {
      this.timeElapsed = 0;
      this.selectedDifficulty = difficulty;
      this.selectingDifficulty = false;
      this.gameBoard = [];
      this.initialMove = true;
      this.totalMines = 0;
      this.blocksRevealed = 0;
      this.gameOver = false;
      this.gameWin = false;
      this.explodedMine = {
        x: null,
        y: null,
      };
      this.generateGrid();
    },
    mineStyle(x, y) {
      let backgroundColor;
      if (this.explodedMine.x === x && this.explodedMine.y === y) {
        backgroundColor = 'red';
      } else if (this.gameWin) {
        backgroundColor = 'limegreen';
      } else {
        backgroundColor = 'rgb(235, 235, 235)';
      }

      return backgroundColor;
    },
    findColor(block) {
      let style = 'black';

      if (!block.isMine) {
        switch(block.minesInProximity) {
          case 1:
            style = 'blue';
            break;
          case 2:
            style = 'green';
            break;
          case 3:
            style = 'red';
            break;
          default:
            style = 'navy';
        }
      }

      return style;
    },

    generateGrid() {
      for (let column = 0; column < this.gameBoardSize; column++) {
        const newRow = [];
        for (let row = 0; row < this.gameBoardSize; row++) {
          newRow.push(
            {
              isMine: Math.random() < this.difficulty[this.selectedDifficulty]/100,
              minesInProximity: 0,
              isVisible: false,
              isFlagged: false,
            },
          );
        }
        this.gameBoard.push(newRow);
      }

      this.mapMineProximityValuesOfBlocks();
    },

    mapMineProximityValuesOfBlocks() {
      this.gameBoard.forEach((row, y) => {
        row.forEach( (block, x) => {
          if (block.isMine) {
            this.totalMines++;
            this.iterateProximityValues(x, y, 1);
          }
        });
      });
    },

    iterateProximityValues(x, y, increment) {
      
      //checks if on first row, if not iterates previous row's mines in proximity count
      if (y > 0) {
        if (x > 0) {
          this.gameBoard[y - 1][x - 1].minesInProximity += increment;
        }
     
        this.gameBoard[y - 1][x].minesInProximity += increment;
        
        if (x < this.gameBoardSize - 1) {
          this.gameBoard[y - 1][x + 1].minesInProximity += increment;
        }
      }

      //iterates current row's mines in proximity count
      if (x > 0) {
        this.gameBoard[y][x - 1].minesInProximity += increment;
      }
          
      if (x < this.gameBoardSize - 1) {
        this.gameBoard[y][x + 1].minesInProximity += increment;
      }

      //checks if on last row, if not iterates following row's mines in proximity count
      if (y < this.gameBoardSize - 1) {
        if (x > 0) {
          this.gameBoard[y + 1][x - 1].minesInProximity += increment;
        }
     
        this.gameBoard[y + 1][x].minesInProximity += increment;
        
        if (x < this.gameBoardSize - 1) {
          this.gameBoard[y + 1][x + 1].minesInProximity += increment;
        }
      }
    },

    clickBlock(x, y) {
      if (this.initialMove) {
        this.initialMove = false;
        this.startTimer();
        if (this.gameBoard[y][x].isMine) {
          this.gameBoard[y][x].isMine = false;
          this.totalMines--;
          this.iterateProximityValues(x, y, -1);
        } 
      }

      if (!this.gameBoard[y][x].isVisible) {
        this.gameBoard[y][x].isVisible = true;
        this.blocksRevealed++;

        if (this.gameBoard[y][x].isMine){
          this.explodedMine = {x, y};
          this.gameOver = true;
          this.stopTimer();
          this.revealAllBlocks();
        } else if (this.gameBoard[y][x].minesInProximity === 0) {
          this.revealSurroundingBlocks(x, y);
        }
      }

      if (this.blocksRemaining === 0) {
        this.gameWin = true;
        this.stopTimer();
        this.revealAllBlocks();
      }
    },

    revealSurroundingBlocks(x, y) {
      //checks if on first row and if block is mine or has been revealed already
      if (y > 0) {
        if (!this.gameBoard[y - 1][x].isVisible && !this.gameBoard[y - 1][x].isMine) {
          const click = () => this.clickBlock(x, (y - 1));
          setTimeout(click, 1);
        }
      }

      //checks if block is mine or has been revealed already
      if (x > 0) {
        if (!this.gameBoard[y][x - 1].isVisible && !this.gameBoard[y][x - 1].isMine) {
          const click = () => this.clickBlock((x - 1), y);
          setTimeout(click, 1);
        }
      }
          
      if (x < this.gameBoardSize - 1) {
        if (!this.gameBoard[y][x + 1].isVisible && !this.gameBoard[y][x + 1].isMine) {
          const click = () => this.clickBlock((x + 1), y);
          setTimeout(click, 1);
        }
      }

      //checks if on last row and checks if block is mine or has been revealed already
      if (y < this.gameBoardSize - 1) {
        if (!this.gameBoard[y + 1][x].isVisible && !this.gameBoard[y + 1][x].isMine) {
          const click = () => this.clickBlock(x, (y + 1));
          setTimeout(click, 1);
        }
      }

      if (this.blocksRemaining === 0) {
        this.gameWin = true;
        this.revealAllBlocks();
      }
    },
    revealAllBlocks() {
      this.gameBoard.forEach(row => {
        row.forEach( block => {
          block.isVisible = true;
        });
      });
    },
  },
};
</script>

<style scoped lang="scss">
.grid {
  width: fit-content;
  transform: translateX(12px)
}

.game-wrapper {
  height: 100%;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: white;
  overflow:scroll;
}

.game-board {
  height: fit-content;
  width: fit-content;
  border: 3px solid grey;
  border-radius: $border-radius-md;
  background: lightgrey;
}

.infoPanel {
  height: 100px;
  width: 100%;
  background: lightgrey;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: $size-6 0;
  font-size: $font-size-sm;
}

.block {
  width: 30px;
  height: 30px;
  background: rgb(235, 235, 235);
  border: 1px solid black;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  font-size: $font-size-md;
  cursor: pointer;
  color: 'black';
}

.hiddenBlock  {
  width: 100%;
  height: 100%;
  background: darkgrey;
  display: flex;
  justify-content: center;
  align-items: center;
}

.mine {
  font-size: 1.25rem;
  padding: 0.25rem 0 0 0.25rem;
  background: rgb(235, 235, 235);
}

.flag {
  font-size: $size-5;
  padding-left: $size-2;
}

.game-over {
  color: red;
  font-size: $size-7;
}

.game-win {
  color: green;
  font-size: $size-7;
  display:flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.timeElapsed {
  color: navy;
}

.numberOfMines {
  color: red;
}

.row {
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;
  white-space: nowrap;
}
</style>