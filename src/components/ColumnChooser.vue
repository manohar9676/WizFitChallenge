<template>
    <div v-if="show" class="modal-overlay">
      <div class="modal-content card">
        <div class="modal-header">
          <h3>Choose Columns</h3>
          <button class="close-btn" @click="close">&times;</button>
        </div>
        <div class="checkbox-container">
          <label v-for="column in availableColumns" :key="column.field" class="column-checkbox">
            <input
              type="checkbox"
              v-model="selectedColumnsLocal"
              :value="column.field"
              @change="updateColumns"
            />
            {{ column.label }}
          </label>
        </div>
        <div class="modal-footer">
          <button @click="close">Close</button>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, watch } from 'vue';
  
  const props = defineProps({
    show: Boolean,
    availableColumns: Array,
    selectedColumns: Array
  });
  
  const emit = defineEmits(['update:show', 'update:selectedColumns']);
  
  const selectedColumnsLocal = ref(props.selectedColumns ? [...props.selectedColumns] : []);
  
  const close = () => {
    emit('update:show', false);
  };
  
  const updateColumns = () => {
    emit('update:selectedColumns', selectedColumnsLocal.value);
  };
  
  watch(() => props.selectedColumns, (newVal) => {
    selectedColumnsLocal.value = [...newVal];
  });
  </script>