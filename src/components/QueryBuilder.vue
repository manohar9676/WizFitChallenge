<template>
    <div v-if="show" class="modal-overlay">
      <div class="modal-content card">
        <div class="modal-header">
          <h3>Search Schools</h3>
          <button class="close-btn" @click="close">&times;</button>
        </div>
        
        <div class="modal-body">
          <div v-for="(filter, index) in filtersLocal" :key="index" class="filter-row">
            <select v-model="filter.field" class="field-select">
              <option value="" disabled>Select Field</option>
              <option value="user_name">Name</option>
              <option value="user_email">Email</option>
              <option value="city_name">City</option>
              <option value="state_name">State</option>
            </select>
  
            <select v-model="filter.operator" class="operator-select">
              <option value="" disabled>Select Operator</option>
              <option value="contains">Contains</option>
              <option value="equals">Equals</option>
              <option value="starts_with">Starts With</option>
            </select>
  
            <input
              v-model="filter.value"
              type="text"
              class="value-input"
              placeholder="Enter value"
            />
  
            <button @click="removeFilter(index)" class="remove-btn">Remove</button>
          </div>
        </div>
  
        <div class="modal-footer">
          <button @click="addFilter" class="secondary-btn">Add Filter</button>
          <div class="footer-right">
            <button @click="close" class="secondary-btn">Cancel</button>
            <button @click="applyFilters" class="primary-btn">Apply Filters</button>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, watch } from 'vue';
  
  const props = defineProps({
    show: Boolean,
    filters: {
      type: Array,
      default: () => []
    }
  });
  
  const emit = defineEmits(['update:show', 'update:filters', 'apply-filters']);
  
  const filtersLocal = ref(props.filters ? [...props.filters] : []);
  
  const close = () => {
    emit('update:show', false);
  };
  
  const addFilter = () => {
    filtersLocal.value.push({ field: '', operator: '', value: '' });
  };
  
  const removeFilter = (index) => {
    filtersLocal.value.splice(index, 1);
  };
  
  const applyFilters = () => {
    emit('update:filters', filtersLocal.value);
    emit('apply-filters');
    close();
  };
  
  watch(() => props.filters, (newVal) => {
    filtersLocal.value = [...newVal];
  });
  </script>