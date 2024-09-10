<!-- ModalDialog.vue -->
<script setup>
  const emit = defineEmits(['confirm', 'cancel'])
</script>

<template>
  <div class="modal center" ref="modal">
    <!-- The modal component body -->
     <h3>Reset ?</h3>
    <button @click="emit('confirm')">Confirm</button>
    <button @click="emit('cancel')">Annuler</button>
  </div>
</template>

<style scoped>

.modal {
  background: #b46143;
  border-radius: 10px;
  padding: 1em;
  z-index: 2;
}

h3 {
  text-align: center;
  margin-bottom: 0.5-em;
}

button {
  margin-left: 0.5em;
  margin-right: 0.5em;
}
.center {
  position: absolute;
  left: 38%;
  top: 50%;
  transform: translate(-50%, -50%);
}

</style>