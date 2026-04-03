<template>
<div class="players">
<div
v-for="(p, idx) in players"
:key="p.id"
:class="['player', { current: idx === currentPlayerIndex }]"
>
<strong>Player {{ idx + 1 }}:</strong>
<span class="money">${{ p.balance }}</span>
</div>
</div>
</template>


<script>
export default {
name: 'PlayerPanel',
props: {
players: { type: Array, required: true },
currentPlayerIndex: { type: Number, required: true }
}
}
</script>


<style scoped>
.players { display: flex; gap: 1rem; margin-bottom: 1rem; }
.player { padding: .5rem .75rem; border-radius: 8px; border: 1px solid #e0e0e0; background: #f8f9fb; }
.player.current { outline: 2px solid #1e56a0; background: #eef4ff; }
.money { margin-left: .25rem; font-weight: 700; }
</style>