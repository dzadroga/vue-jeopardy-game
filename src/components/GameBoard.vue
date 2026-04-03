<template>
  <div class="board">
    <!-- header row -->
    <div class="row header">
      <div class="cell" v-for="(cat, col) in categories" :key="'h'+col">
        <div class="title" :title="cat">{{ cat }}</div>
      </div>
    </div>

    <div class="row" v-for="row in 5" :key="'r'+row">
      <div class="cell" v-for="(cat, col) in categories" :key="'c'+col">
        <button
          class="clue"
          :class="{
            revealed: board[col]?.[row-1]?.revealed,
            correct: board[col]?.[row-1]?.wasCorrect,
            incorrect: board[col]?.[row-1]?.wasCorrect === false
          }"
          :disabled="!board[col]?.[row-1] || board[col][row-1].revealed"
          @click="$emit('select-cell', { col, row: row - 1 })"
        >
          <template v-if="!board[col]?.[row-1]?.revealed">
            ${{ board[col]?.[row-1]?.amount ?? '' }}
          </template>

          <template v-else>
            P{{ board[col][row-1].owner }}
          </template>
        </button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'GameBoard',
  props: {
    categories: { type: Array, required: true },
    board: { type: Array, required: true }, 
  },
  emits: ['select-cell'],
}
</script>

<style scoped>
.board {
  width: 100%;
  max-width: 1100px;
  margin: 0 auto;
}

.row {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
}

.cell {
  background: #0a3a78;
  border: 1px solid #153a6b;
  padding: 0;              
  height: 90px;           
  display: flex;
  align-items: center;
  justify-content: center;
  box-sizing: border-box;
}

.header .cell {
  background: #1b4f99;
  height: 60px;
  padding: 4px;
}

.title {
  color: #d7e8ff;
  font-weight: 600;
  text-align: center;
  white-space: normal;     
  word-break: break-word;  
  line-height: 1.2;
}

.clue {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  background: transparent;
  color: #ffc400;
  border: 0;
  cursor: pointer;
}

.clue:disabled {
  opacity: 0.8;
  cursor: not-allowed;
}

/* states */
.clue.revealed.correct {
  background: #0ca10c;
  color: #fff;
}
.clue.revealed.incorrect {
  background: #d02626;
  color: #fff;
}

</style>
