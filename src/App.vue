<template>
   <VCard>
    <VCard class="wizfit-header">
      <img src="../src/assets/banner.png" alt="WizFit Challenge Logo" class="logo" />
    </VCard>
    <VCard class="player-container">
      <img src="../src/assets/player.png" alt="WizFit Challenge Logo" class="playerpng"/>
    </VCard>
    <!-- Main content area -->
    <div class="main-content">

<!-- Column Chooser Modal -->
<div v-if="showColumnChooser" class="modal-overlay">
  <div class="modal-content card">
    <div class="modal-header">
      <h3>Choose Columns</h3>
      <button class="close-btn" @click="showColumnChooser = false">&times;</button>
    </div>
    <div class="checkbox-container">
      <label v-for="column in availableColumns" :key="column.field" class="column-checkbox">
        <input
          type="checkbox"
          v-model="selectedColumns"
          :value="column.field"
        />
        {{ column.label }}
      </label>
    </div>
    <div class="modal-footer">
      <button @click="showColumnChooser = false">Close</button>
    </div>
  </div>
</div>

<!-- Query Builder Modal -->
<div v-if="showQueryBuilder" class="modal-overlay">
  <div class="modal-content card">
    <div class="modal-header">
      <h3>Search Schools</h3>
      <button class="close-btn" @click="showQueryBuilder = false">&times;</button>
    </div>
    
    <div class="modal-body">
      <div v-for="(filter, index) in filters" :key="index" class="filter-row">
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
        <button @click="showQueryBuilder = false" class="secondary-btn">Cancel</button>
        <button @click="applyFiltersAndClose" class="primary-btn">Apply Filters</button>
      </div>
    </div>
  </div>
</div>

<!-- Table -->
<div class="table-container card">
  <div class="table-header">
    <div class="left-section">
      <h3>School List</h3>
    </div>
    <div class="right-section">
      <button class="column-settings-btn" @click="showColumnChooser = true">
        <span>Column Settings</span>
      </button>
      <button class="query-builder-btn" @click="showQueryBuilder = true">
        <span>Search Schools</span>
      </button>
    </div>
  </div>
  
  <div v-if="isLoading" class="loading-state">
    Loading schools...
  </div>

  <!-- Add table here -->
  <div class="table-wrapper">
    <table v-if="!isLoading">
      <thead>
        <tr>
          <th v-for="column in visibleColumns" :key="column.field" @click="sortBy(column.field)">
            {{ column.label }}
            <span class="sort-indicator">{{ getSortIndicator(column.field) }}</span>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(school, index) in filteredData" :key="index">
          <td v-for="column in visibleColumns" :key="column.field">
            {{ school[column.field] }}
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</div>
</div>
   </VCard>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import axios from 'axios';
import './App.css'

const apiUrl = 'https://api.devharlemwizardsinabox.com/campaign/campaign_school_list/?format=json&search=';

// State variables
const schoolData = ref([]);
const filters = ref([
  { field: '', operator: '', value: '' },
]);
const filteredData = ref([]);

// Add these new state variables
const availableColumns = ref([
  { field: 'user_name', label: 'Name' },
  { field: 'user_email', label: 'Email' },
  { field: 'city_name', label: 'City' },
  { field: 'state_name', label: 'State' },
  { field: 'sales_rep_name', label: 'Sales Rep' },
  { field: 'school_name', label: 'School Name' },
  { field: 'campaign_id', label: 'campaign_id' },
  { field: 'campaign_type', label: 'campaign_type' },
]);

const selectedColumns = ref([
  'user_name',
  'user_email',
  'city_name',
  'state_name',
  'sales_rep_name'
]);

// Computed property for visible columns
const visibleColumns = computed(() => {
  return availableColumns.value.filter(column => 
    selectedColumns.value.includes(column.field)
  );
});

// Add new state variables
const error = ref(null);
const isLoading = ref(false);

// Add new sorting state
const sortField = ref('');
const sortDirection = ref('asc');

// Add sorting functions
const sortBy = (field) => {
  if (sortField.value === field) {
    // Toggle direction if clicking the same field
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc';
  } else {
    // New field, default to ascending
    sortField.value = field;
    sortDirection.value = 'asc';
  }
  
  // Apply sorting
  filteredData.value = [...filteredData.value].sort((a, b) => {
    const aVal = (a[field] || '').toString().toLowerCase();
    const bVal = (b[field] || '').toString().toLowerCase();
    
    if (sortDirection.value === 'asc') {
      return aVal.localeCompare(bVal);
    } else {
      return bVal.localeCompare(aVal);
    }
  });
};

const getSortIndicator = (field) => {
  if (sortField.value !== field) return '↕';
  return sortDirection.value === 'asc' ? '↑' : '↓';
};

// Fetch data
const fetchData = async () => {
  error.value = null;
  isLoading.value = true;
  
  try {
    const response = await axios.get(apiUrl);
    if (response.data?.school_list) {
      schoolData.value = response.data.school_list;
      filteredData.value = schoolData.value;
    } else {
      throw new Error('Invalid data format received');
    }
  } catch (err) {
    error.value = err.response?.data?.message || 'Failed to fetch schools. Please try again later.';
    schoolData.value = [];
    filteredData.value = [];
  } finally {
    isLoading.value = false;
  }
};

// Add filter
const addFilter = () => {
  filters.value.push({ field: '', operator: '', value: '' });
};

// Remove filter button
const removeFilter = (index) => {
  filters.value.splice(index, 1);
  applyFilters(); 
};

// Apply filters
const applyFilters = () => {
  error.value = null;
  try {
    filteredData.value = schoolData.value.filter((school) => {
      return filters.value.every((filter) => {
        if (!filter.field || !filter.operator || !filter.value) return true;

        const fieldValue = school[filter.field]?.toString().toLowerCase();
        if (!fieldValue) return false;

        const filterValue = filter.value.toLowerCase();

        if (filter.operator === 'contains') {
          return fieldValue.includes(filterValue);
        }
        if (filter.operator === 'equals') {
          return fieldValue === filterValue;
        }
        if (filter.operator === 'starts_with') {
          return fieldValue.startsWith(filterValue);
        }
        return true;
      });
    });
  } catch (err) {
    error.value = 'Error applying filters. Please check your search criteria.';
    filteredData.value = [];
  }
};

// Add this new ref for modal control
const showColumnChooser = ref(false);

// Add new ref for query builder modal
const showQueryBuilder = ref(false);

// Modified apply filters function
const applyFiltersAndClose = () => {
  applyFilters();
  showQueryBuilder.value = false;
};

onMounted(fetchData);
</script>
