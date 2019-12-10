<template>
  <div class="main-container">
    <button
      class="newgame-button"
      :class="{ lose: gameOver === 'lose', win: gameOver === 'win' }"
      @click="newGame()"
    ></button>

    <div class="help-button" @click="helpClick()">Help me</div>

    <div class="difficulty-container">
      <label>Difficulty</label>
      <button
        v-for="item in difficultyOptions"
        :key="'diff-' + item"
        :class="{ disabled: selectedDifficulty === item }"
        @click="selectedDifficulty = item"
      >{{ item }}</button>
    </div>

    <div class="game-container">
      <div v-for="(row, rowIndex) in map" :key="'row-' + rowIndex" class="row">
        <button
          v-for="(tile, columnIndex) in row"
          :key="'row-' + columnIndex"
          :class="{
              disabled: tile !== '□',
              flagged: tile === '#',
              boom: tile === '*'
            }"
          @click="selectTile(rowIndex, columnIndex)"
          @contextmenu.prevent="flagTile(rowIndex, columnIndex)"
        >{{ ["□", "0", "*", "#"].indexOf(tile) === -1 ? tile : "" }}</button>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";

@Component
export default class MineSweeper extends Vue {
  @Prop() private msg!: string;
  difficultyOptions: number[] = [1, 2, 3, 4];
  selectedDifficulty: number = 1;
  connection!: WebSocket;
  map: Array<string>[] = [];
  flagMap!: Array<boolean>[];
  gameOver: string = "";

  /**
   * @description Constructor function - initializes the component
   */
  created() {
    const url = "wss://hometask.eg1236.com/game1/";
    this.connection = new WebSocket(url);
    this.connection.onopen = () => {
      this.connection.send("new " + this.selectedDifficulty);
    };

    this.connection.onmessage = e => {
      if (!e.data.startsWith("map")) {
        this.connection.send("map");
        if (e.data.indexOf("You lose") !== -1) this.gameOver = "lose";
        if (e.data.indexOf("You win") !== -1) {
          this.gameOver = "win";
          alert(e.data);
        }
      } else {
        this.updateMap(e.data);
      }
    };
  }

  /**
   * @description Updates the game map
   * @param {string} message - the input message received from websocket map response
   */
  updateMap(message: string) {
    // split message into rows by newline character
    const rows = message.split(/\r?\n/);

    // remove first and last line
    rows.shift();
    rows.pop();

    // split columns
    rows.forEach((row, rowIndex) => {
      const newRow = row.split("").map((item, columnIndex) => {
        if (
          item === "□" &&
          this.flagMap &&
          this.flagMap[rowIndex][columnIndex]
        ) {
          return "#";
        }
        return item;
      });
      Vue.set(this.map, rowIndex, newRow);
    });

    // remove extra rows if difficulty has been lowered
    this.map.splice(rows.length);

    // initialize flagMap if it has not been set
    if (!this.flagMap) {
      this.flagMap = [];
      this.map.forEach(row => {
        this.flagMap.push(row.map(item => false));
      });
    }
  }

  /**
   * @description Restarts the game
   */
  newGame() {
    this.connection.send("new " + this.selectedDifficulty);
    this.gameOver = "";
    delete this.flagMap;
  }

  /**
   * @description reveals tiles around the selected tile if amount of flags around the tile matches the surrounding bomb count
   * @param {number} tileNumber - number of bombs around the tile
   * @param {number} row - row index
   * @param {number} column - column index
   */
  revealSurroundingTiles(tileNumber: number, row: number, column: number) {
    const tilesToReveal: any[] = [];
    let flagCount = 0;

    // loop through 3x3 grid around tile
    for (let x = -1; x < 2; x++) {
      for (let y = -1; y < 2; y++) {
        if (this.map[row + y] && this.map[row + y][column + x]) {
          if (this.map[row + y][column + x] === "#") {
            flagCount++;
          } else if (this.map[row + y][column + x] === "□") {
            tilesToReveal.push({
              row: row + y,
              column: column + x
            });
          }
        }
      }
    }

    // if all bombs are flagged, reveal surrounding empty tiles
    if (tileNumber === flagCount) {
      tilesToReveal.forEach(tile => {
        this.connection.send(`open ${tile.column} ${tile.row}`);
      });
    }
  }

  /**
   * @description handles click event of grid tiles
   * @param {number} row - row index
   * @param {number} column - column index
   */
  selectTile(row: number, column: number) {
    const tileNumber = parseInt(this.map[row][column]);

    // start new game if the game is over
    if (this.gameOver) {
      this.newGame();

      // open tile if it is empty
    } else if (this.map[row][column] === "□") {
      this.connection.send(`open ${column} ${row}`);

      // reveal surrounding tiles if a number tile greater than 0 was clicked
    } else if (tileNumber) {
      this.revealSurroundingTiles(tileNumber, row, column);
    }
  }

  /**
   * @description toggles flag status of a tile - triggered on right click on a tile
   * @param {number} row - row index
   * @param {number} column - column index
   */
  flagTile(row: number, column: number) {
    if (this.map[row][column] === "□" || this.map[row][column] === "#") {
      this.flagMap[row][column] = !this.flagMap[row][column];

      const mapRow = this.map[row];
      mapRow[column] = this.flagMap[row][column] ? "#" : "□";
      Vue.set(this.map, row, mapRow);
    }
  }

  /**
   * @description helper function - flags all tiles that 100% should be flagged and reveals empty tiles that can be revealed
   * @param {number} row - row index
   * @param {number} column - column index
   */
  helpClick() {
    // loop through all tiles
    for (let y = 0; y < this.map.length; y++) {
      for (let x = 0; x < this.map[y].length; x++) {
        const bombCount = parseInt(this.map[y][x]);
        if (bombCount) {
          let flagTileCount = 0;
          const surroundingTiles: any[] = [];

          // if this is a number tile greater than 0, loop through 3x3 grid around the tile
          for (let offsetY = -1; offsetY < 2; offsetY++) {
            for (let offsetX = -1; offsetX < 2; offsetX++) {
              // if surrounding tile is empty, add to list
              if (
                this.map[y + offsetY] &&
                this.map[y + offsetY][x + offsetX] === "□"
              ) {
                surroundingTiles.push({
                  x: x + offsetX,
                  y: y + offsetY
                });
                // if surrounding tile is a flag, increment flag count
              } else if (
                this.map[y + offsetY] &&
                this.map[y + offsetY][x + offsetX] === "#"
              )
                flagTileCount++;
            }
          }

          // flag all surrounding tiles if current flag amount + empty surrounding tiles is the same amount as bomb count
          if (
            surroundingTiles.length > 0 &&
            flagTileCount + surroundingTiles.length === bombCount
          ) {
            surroundingTiles.forEach(item => {
              this.flagMap[item.y][item.x] = true;
              const mapRow = this.map[item.y];
              mapRow[item.x] = "#";
              Vue.set(this.map, item.y, mapRow);
            });
          }

          // reveal all surrounding empty tiles if bomb count is the same as flag amount
          if (flagTileCount === bombCount) {
            surroundingTiles.forEach(item => {
              this.connection.send(`open ${item.x} ${item.y}`);
            });
          }
        }
      }
    }
  }
}
</script>

<style scoped>
/* general styling for buttons */
button {
  cursor: pointer;
  outline: none !important;
}

button:not(.disabled):focus,
button:not(.disabled):hover,
button:not(.disabled):active {
  border: 1px solid var(--color-blue);
  box-shadow: 0px 0px 6px 1px var(--color-blue);
}

.main-container {
  width: fit-content;
  margin: 30px auto 30px;
  border: 1px solid var(--color-gray);
  padding: 30px;
  border-radius: 4px;
  box-shadow: 0px 0px 10px 3px var(--color-gray);
  min-width: 300px;
  position: relative;
  background: var(--color-white);
}

.newgame-button {
  margin: 0 auto;
  width: 5em;
  height: 5em;
  border: 1px solid var(--color-gray);
  border-radius: 4px;
  background: var(--color-white) url("~@/assets/icons/sun.svg") center no-repeat;
  background-size: 4em;
  box-shadow: none;
  transition: all 0.3s ease-in-out;
}

.newgame-button.lose {
  background: var(--color-white) url("~@/assets/icons/lose.svg") center
    no-repeat;
  background-size: 4em;
}

.newgame-button.win {
  background: var(--color-white) url("~@/assets/icons/win.svg") center no-repeat;
  background-size: 4em;
}

.difficulty-container {
  position: absolute;
  top: 30px;
  right: 30px;
  display: flex;
}

.difficulty-container label {
  position: absolute;
  font-size: 12px;
  top: -15px;
  left: 3px;
}

.difficulty-container button {
  width: 20px;
  height: 20px;
  background: var(--color-white);
  border: 1px solid var(--color-gray);
  border-radius: 2px;
  margin: 0 3px;
  padding: 0;
  transition: all 0.3s ease-in-out;
}

.difficulty-container button.disabled {
  background: var(--color-blue);
  color: var(--color-white);
  cursor: unset;
}

.game-container .row {
  display: flex;
}

.game-container button {
  width: 30px;
  height: 30px;
  margin: 3px;
  padding: 0;
  border: 1px solid var(--color-gray);
  border-radius: 4px;
  transition: all 0.3s ease-in-out;
}

.game-container button.disabled {
  background: var(--color-gray);
}

.game-container button.flagged {
  background: var(--color-gray) url("~@/assets/icons/flag.svg") center no-repeat;
  background-size: 16px;
  cursor: unset;
}

.game-container button.boom {
  background: var(--color-gray) url("~@/assets/icons/explosion.svg") center
    no-repeat;
  cursor: unset;
}

.help-button {
  position: fixed;
  top: 30px;
  left: 30px;
  font-size: 0.8em;
  opacity: 0.1;
  transition: opacity 0.2s ease-in-out;
  cursor: pointer;

  border: 1px solid var(--color-gray);
  padding: 3px 8px;
  border-radius: 4px;
  user-select: none;
}
.help-button:hover {
  opacity: 1;
}
</style>
