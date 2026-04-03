<template>
  <div class="backdrop" @click.self="$emit('close')">
    <div class="question">
      <h3>{{ categoryName }} — ${{ amount }}</h3>
      <p class="q">{{ question }}</p>

      <!-- choices -->
      <div class="actions" v-if="type === 'boolean'">
        <button class="btn t" @click="choose('True')">True</button>
        <button class="btn f" @click="choose('False')">False</button>
      </div>

      <div class="actions" v-else>
        <button
          v-for="(c, i) in (choices && choices.length ? choices : ['True','False'])"
          :key="i"
          class="btn"
          @click="choose(c)"
        >{{ c }}</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'QuestionModal',
  props: {
    categoryName:  { type: String, required: true },
    amount:        { type: Number, required: true },
    question:      { type: String, required: true },
    type:          { type: String, default: 'boolean' },
    correctAnswer: { type: String, default: '' },  
    choices:       { type: Array,  default: () => [] }
  },
  emits: ['answer', 'close'],
  methods: {
    choose(choice) {
      const norm = s => (s ?? '').trim().toLowerCase()
      const correct = norm(choice) === norm(this.correctAnswer)
      this.$emit('answer', { correct, choice })
    }
  }
}
</script>


<style scoped>
.backdrop {
  position: fixed; inset: 0; background: rgba(0,0,0,.35);
  display: grid; place-items: center; z-index: 50;
}
.question {
  margin: 14px; max-width: 900px; width: min(900px, 92vw);
  padding: 16px; border: 1px solid #e2e8f0; border-radius: 10px; background: #fff;
  box-shadow: 0 12px 30px rgba(0,0,0,.15);
}
.q { font-size: 1.05rem; margin: 10px 0 14px; }
.actions { display: flex; flex-wrap: wrap; gap: 10px; justify-content: center; }
.btn { padding: 8px 14px; border-radius: 8px; border: 1px solid #1b3247; cursor: pointer; background: #fff; }
.btn.t { background: #e7f8ea; }
.btn.f { background: #ffecec; }
.btn:hover { filter: brightness(0.98); }
</style>
