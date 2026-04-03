<template>
  <div class="app">
    <h1 class="title">Jeopardy!</h1>

    <div v-if="setupPhase" class="config">
      <button class="btn primary" @click="startFromSetup">Start Game</button>
    </div>

    <div v-else-if="isLoading" class="loader-wrap">
      <div class="loader">
        <div class="spinner" aria-hidden="true"></div>
        <p class="loader-title">{{ loadMessage }}</p>
        <p class="loader-sub" v-if="totalCells">({{ loadedCells }}/{{ totalCells }})</p>
        <div class="bar"><div class="bar-fill" :style="{ width: progressPct + '%' }"></div></div>
      </div>
    </div>

    <template v-else>
      <div class="topbar">
        <PlayerPanel :players="players" :currentPlayerIndex="currentPlayerIndex" />
        <button class="btn" @click="backToSetup">New Game</button>
      </div>

    <GameBoard
      v-if="isBoardReady"
      :categories="categories"
      :board="board"
      @select-cell="onSelectCell"
    />
      <section class="status" v-if="statusText"><p v-html="statusText"></p></section>

    <QuestionModal
      v-if="activeCell && !activeCell.asked"
      :category-name="activeCell.categoryName"
      :amount="activeCell.amount"
      :question="decodeHtml(activeCell.question)"
      :type="activeCell.type"
      :correct-answer="decodeHtml(activeCell.correct_answer ?? activeCell.correctAnswer ?? '')"
      @answer="onAnswer"
    />

      <section class="result" v-if="lastResultMessage">
        <p :class="{ correct: lastResultWasCorrect, incorrect: !lastResultWasCorrect }">{{ lastResultMessage }}</p>
      </section>

      <section v-if="gameOver" class="winner">
        <h2>Player {{ winningPlayerIndex + 1 }} has won the game!</h2>
      </section>
    </template>
  </div>
</template>

<script>
import GameBoard from "./components/GameBoard.vue";
import QuestionModal from "./components/QuestionModal.vue";
import PlayerPanel from "./components/PlayerPanel.vue";
import { EXCLUDED_CATEGORY_IDS } from './config'


export default {
  name: "App",
  components: { GameBoard, QuestionModal, PlayerPanel },

  data() {
    return {
      //config
      cfgPlayers: 3,
      cfgCategories: 4,

      //state
      setupPhase: true,
      isLoading: false,
      loadMessage: "",
      loadedCells: 0,
      totalCells: 0,

      players: [],
      currentPlayerIndex: 0,
      categories: [],
      board: [],              
      packs: [],

      activeCell: null,
      statusText: "",
      lastResultMessage: "",
      lastResultWasCorrect: null,
      gameOver: false,
      winningPlayerIndex: 0,

      //API/session
      _token: null,           //session token
      _overallTimer: null,
      _initInFlight: false,   //guard against double
      _resultPool: [],        
    };
  },

  computed: {
    progressPct() {
      if (!this.totalCells) return 0;
      return Math.min(100, Math.round((this.loadedCells / this.totalCells) * 100));
    },
    isBoardReady() {
      return (
        Array.isArray(this.board) &&
        this.board.length === this.cfgCategories &&
        this.board.every(col => Array.isArray(col) && col.length === 5)
      );
    },
  },

  methods: {
    //public controls
    startFromSetup() {
      this.cfgPlayers = 3;
      this.cfgCategories = 4;
      this.players = Array.from({ length: this.cfgPlayers }, (_, i) => ({ id: i + 1, balance: 0 }));
      this.setupPhase = false;
      this.initGame();
    },

    backToSetup() {
      this.setupPhase = true;
      this.isLoading = false;
      this.categories = [];
      this.board = [];
      this.statusText = "";
      this.lastResultMessage = "";
      this.gameOver = false;
      this.packs = [];
      if (this._overallTimer) clearTimeout(this._overallTimer);
    },

    //main flow
    async initGame() {
      if (this._initInFlight) return;
      this._initInFlight = true;

      //reset for the new game
      this.currentPlayerIndex = 0;
      this.categories = [];
      this.board = [];
      this.activeCell = null;
      this.statusText = "";
      this.lastResultMessage = "";
      this.lastResultWasCorrect = null;
      this.gameOver = false;
      this.packs = [];
      this.winningPlayerIndex = 0;

      this.isLoading = true;
      this.loadMessage = "Fetching questions…";
      this.totalCells = this.cfgCategories * 5;
      this.loadedCells = 0;
      this._resultPool = [];

      try {
        this._token = await this.requestToken().catch(() => null);

        const qs = new URLSearchParams({ amount: "60", type: "boolean" });
        if (this._token) qs.set("token", this._token);
        const url = `https://opentdb.com/api.php?${qs.toString()}`;

        try {
          const data = await this.fetchJSON(url, { retries: 2, backoff: 400, timeoutMs: 5000 });
          if (Array.isArray(data?.results)) this._resultPool.push(...data.results);
          const packs = this.buildPacksFromSinglePull(this._resultPool);
          this.buildBoardFromPacks(packs);
        } catch { /* ignore; we’ll loop below */ }

        while (!this.isBoardReady) {
          this.loadMessage = "Finding the Hardest Questions...";
          await this.sleep(900 + Math.floor(Math.random() * 500));

          try {
            const data2 = await this.fetchJSON(url, { retries: 1, backoff: 400, timeoutMs: 5000 }).catch(() => null);
            const results2 = Array.isArray(data2?.results) ? data2.results : [];
            if (results2.length) {
              this._resultPool.push(...results2);
              const packs = this.buildPacksFromSinglePull(this._resultPool);
              this.buildBoardFromPacks(packs); // updates progress
            }
          } catch { /* keep looping */ }
        }

        this.loadedCells = this.totalCells;
        this.isLoading = false;
        this.statusText = `Player ${this.currentPlayerIndex + 1} to select`;
      } finally {
        this._initInFlight = false;
      }
    },

    buildPacksFromSinglePull(results) {
      //group name
      const groups = new Map();
      for (const q of results) {
        const key = q.category || "Misc";
        if (!groups.has(key)) groups.set(key, []);
        groups.get(key).push(q);
      }

      //keep categories with 5 boolean questions
      const candidates = Array.from(groups.entries())
        .map(([name, arr]) => ({ name, arr }))
        .filter(g => g.arr.length >= 5);

      //shuffle
      const shuffled = this.shuffle(candidates).slice(0, this.cfgCategories);
      const packs = [];

      for (const g of shuffled) {
        //by difficulty
        const easy = g.arr.filter(x => x.difficulty === "easy");
        const med  = g.arr.filter(x => x.difficulty === "medium");
        const hard = g.arr.filter(x => x.difficulty === "hard");

        const pick = (arr, n) => this.shuffle(arr).slice(0, Math.min(n, arr.length));
        let chosen = [];

        const e2 = pick(easy, 2);
        const m2 = pick(med, 2);
        const h1 = pick(hard, 1);
        chosen = [...e2, ...m2, ...h1];

        //if still nor five get from any difficulty
        if (chosen.length < 5) {
          const used = new Set(chosen);
          const rest = g.arr.filter(q => !used.has(q));
          chosen = [...chosen, ...pick(rest, 5 - chosen.length)];
        }

        const five = chosen.slice(0, 5).map((q, i) => ({
          ...q,
          amount: (i + 1) * 100,
          asked: false,
          revealed: false,
        }));

        if (five.length === 5) {
          packs.push({ category: { name: g.name }, five });
        }
      }

      return packs.slice(0, this.cfgCategories);
    },


    //run tasks with concurrency
    async withLimit(limit, tasks) {
      const results = [];
      let i = 0;
      const runners = Array.from({ length: Math.min(limit, tasks.length) }, async () => {
        while (i < tasks.length) {
          const idx = i++;
          results[idx] = await tasks[idx]();
        }
      });
      await Promise.all(runners);
      return results;
    },


    onAnswer({ correct }) {
      const { col, row, amount } = this.activeCell;
      const playerIdx = this.currentPlayerIndex;
      const player = this.players[playerIdx];

      if (correct) {
        player.balance += amount;
        this.lastResultWasCorrect = true;
        this.lastResultMessage = `Correct! +$${amount}`;
        //stay on same player
      } else {
        player.balance -= amount;
        this.lastResultWasCorrect = false;
        this.lastResultMessage = `Incorrect. -$${amount}`;
        this.currentPlayerIndex = (this.currentPlayerIndex + 1) % this.players.length; //next player
      }

      const cell = this.board[col][row];
      cell.revealed = true;
      cell.asked = true;
      cell.owner = playerIdx + 1;     
      cell.wasCorrect = !!correct;    

      this.activeCell = null;
      this.statusText = `Player ${this.currentPlayerIndex + 1} to select`;

      this.activeCell = null;

      //checks game over
      if (this.boardComplete()) {
        this.gameOver = true;
        this.winningPlayerIndex = this.winnerIndex();
        this.statusText = ""; // stop showing "Player X to select"
      } else {
        this.statusText = `Player ${this.currentPlayerIndex + 1} to select`;
      }
    },

    decodeHtml(s) {
      const txt = document.createElement("textarea");
      txt.innerHTML = s ?? "";
      return txt.value;
     },

    boardComplete() {
      return !this.board.flat().some(c => !c.revealed);
    },

    winnerIndex() {
      //highest score wins
      return this.players
        .map((p, idx) => ({ idx, bal: p.balance }))
        .sort((a, b) => b.bal - a.bal)[0].idx;
    },

    onSelectCell({ col, row }) {
      const cell = this.board?.[col]?.[row];
      if (!cell || cell.revealed) return;
      this.activeCell = {
        ...cell,
        categoryName: this.categories[col],
        correct_answer: cell.correct_answer ?? cell.correctAnswer ?? ''
      };
    },

    //API helpers
    async requestToken() {
      const data = await this.fetchJSON("https://opentdb.com/api_token.php?command=request", { retries: 4, backoff: 600 });
      return data?.response_code === 0 ? data.token : null;
    },

    async fetchCategoryPack(category) {
      const qs = new URLSearchParams({
      amount: "5",
      category: String(category.id),
      type: "boolean",
      });
      if (this._token) qs.set("token", this._token);

      const url = `https://opentdb.com/api.php?${qs.toString()}`;
      const data = await this.fetchJSON(url, { retries: 4, backoff: 700 });
      if (!data || data.response_code !== 0 || !Array.isArray(data.results) || data.results.length < 5) {
        return null;
      }

      const five = data.results.slice(0, 5).map((q, i) => ({
        ...q,                        
        amount: (i + 1) * 100,
        asked: false,
        revealed: false,
      }));

      return { category, five };
    },


    async fetchCategories() {
      const data = await this.fetchJSON("https://opentdb.com/api_category.php", { retries: 4, backoff: 600 });
      return Array.isArray(data?.trivia_categories) ? data.trivia_categories : [];
    },

    //fetch with retry and handling for 429 error
    async fetchJSON(url, { retries = 3, backoff = 600, timeoutMs = 5000 } = {}) {
      for (let attempt = 0; attempt <= retries; attempt++) {
        const controller = new AbortController();
        const tid = setTimeout(() => controller.abort(), timeoutMs);

        try {
          const res = await fetch(url, { cache: "no-store", signal: controller.signal });
          clearTimeout(tid);

          if (res.ok && res.status !== 429) {
            return await res.json();
          }

          if (res.status === 429 && attempt < retries) {
            const wait = backoff * Math.pow(2, attempt) + Math.floor(Math.random() * 300);
            await this.sleep(wait);
            continue;
          }
        } catch {
          clearTimeout(tid);
        }

        if (attempt === retries) throw new Error("fetch failed");
        const wait = backoff * Math.pow(2, attempt) + Math.floor(Math.random() * 200);
        await this.sleep(wait);
      }
    },


    //board building
    buildBoardFromPacks(packs) {
      //category titles 
      this.categories = packs.map(p =>
        this.decodeHtml(p.category?.name ?? `Category ${p.category?.id ?? ""}`)
      );

      this.board = packs.map((p, col) =>
        p.five.map((cell, row) => ({
          ...cell,
          col, row,
          amount: cell.amount ?? (row + 1) * 100,
          revealed: !!cell.revealed,
          asked: !!cell.asked,
        }))
      );

      //Update progress
      this.loadedCells = Math.min(this.totalCells, this.board.flat().length);
    },


    //utils
    sleep(ms) {
      return new Promise(r => setTimeout(r, ms));
    },

    shuffle(arr) {
      const a = arr.slice();
      for (let i = a.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [a[i], a[j]] = [a[j], a[i]];
      }
      return a;
    },
  },
};
</script>


<style scoped>
.app { max-width: 1100px; margin: 0 auto; font-family: Arial, Helvetica, sans-serif; padding: 1rem; }
.title { margin: 0 0 .75rem; text-align: center; font-size: 2.4rem; color: #1b3247; }
.topbar { display: flex; align-items: center; justify-content: space-between; gap: 12px; margin-bottom: 8px; }
.config { display: flex; gap: 16px; justify-content: center; align-items: center; margin-bottom: 16px; flex-wrap: wrap; }
.config label { display: inline-flex; align-items: center; gap: 8px; font-weight: 600; }
.config input { width: 64px; padding: 6px 8px; text-align: center; border: 1px solid #c9d4e5; border-radius: 8px; }
.btn.primary { background: #1e56a0; color: #fff; border-color: #1e56a0; }
.loader-wrap { display: grid; place-items: center; height: 60vh; }
.loader { width: 520px; max-width: 90vw; background: #fff; border: 1px solid #e6ecf5; border-radius: 14px; padding: 20px 24px; box-shadow: 0 6px 24px rgba(27,50,71,.08); text-align: center; }
.spinner { width: 44px; height: 44px; margin: 8px auto 10px; border-radius: 50%; border: 4px solid #e7eef7; border-top-color: #1e56a0; animation: spin 1s linear infinite; }
@keyframes spin { to { transform: rotate(360deg) } }
.loader-title { font-weight: 700; color: #1b3247; margin: 8px 0 4px; }
.loader-sub { color: #4a6075; margin: 0 0 10px; }
.bar { height: 10px; background: #eef4ff; border-radius: 999px; overflow: hidden; border: 1px solid #d8e4ff; margin: 6px 0 10px; }
.bar-fill { height: 100%; width: 0%; background: #1e56a0; transition: width .25s ease; }
.btn { margin-top: 8px; padding: 6px 12px; border-radius: 8px; border: 1px solid #1e56a0; background: #fff; cursor: pointer; }
.btn:hover { background: #f4f8ff; }
.status { margin: 1rem 0; font-weight: 600; font-size: 1.5rem; text-align: center; }
.result p { margin: .25rem 0; text-align: center; }
.correct { color: #0a7a0a; font-weight: bold; }
.incorrect { color: #b00020; font-weight: bold; }
.winner { margin-top: 1rem; padding: .75rem; background: #f3f6ff; border: 1px solid #dbe4ff; border-radius: 8px; text-align: center; }
</style>
