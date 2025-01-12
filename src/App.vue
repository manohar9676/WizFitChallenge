<template>
  <div class="wizfit-container">
    <!-- Header section similar to image -->
    <div class="wizfit-header">
      <img src="../src/assets/banner.png" alt="WizFit Challenge Logo" class="logo" />
    </div>
    <div >

      <img src="../src/assets/player.png" alt="WizFit Challenge Logo" class="playerpng"/>

    </div>

    <!-- Add error message display -->
    <div v-if="error" class="error-message">
      {{ error }}
    </div>

    <!-- Add empty state message -->
    <div v-if="!error && filteredData.length === 0" class="empty-state">
      No schools found matching your criteria
    </div>

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

        <table v-else class="school-table">
          <thead>
            <tr>
              <th v-for="column in visibleColumns" 
                  :key="column.field" 
                  @click="sortBy(column.field)"
                  class="sortable-header"
              >
                {{ column.label }}
                <span class="sort-indicator">
                  {{ getSortIndicator(column.field) }}
                </span>
              </th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="school in filteredData" :key="school.id">
              <td v-for="column in visibleColumns" :key="column.field">
                {{ school[column.field] }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import axios from 'axios';

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

<style scoped>
/* Add these new styles while keeping your existing ones */
.wizfit-container {
  width: 100%;
  margin: 0;
  padding: 0;
  /* min-width: 100vw; */
}

.playerpng{
width: 800px;
}

.wizfit-header {
  /* background: linear-gradient(to right, #4a1e9e, #7c3aed); */
  color: white;
  border-radius: 0;
  margin-bottom: 30px;
  width: 100vw;
  position: relative;
  left: 50%;
  right: 50%;
  margin-left: -50vw;
  margin-right: -50vw;
  box-sizing: border-box;
}

.header-text h1 {
  font-size: 2.5rem;
  margin: 0;
}

.header-text h2, .header-text h3 {
  color: #ffd700;
  margin: 5px 0;
}

.main-content {
  display: grid;
  gap: 30px;
  padding: 0 20px;
  max-width: 100%;
  margin: 0 auto;
  box-sizing: border-box;
}

.card {
  background: white;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  padding: 20px;
}

/* Update existing styles */
.query-builder {
  margin-bottom: 0; /* Remove margin since we're using grid gap */
}

.school-table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  border-radius: 8px;
  overflow: hidden;
}

.school-table th {
  background-color: #4a1e9e;
  color: white;
  padding: 12px;
  text-align: left;
}

.school-table td {
  padding: 12px;
  border-bottom: 1px solid #eee;
}

.filter-row {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
}

.field-select,
.operator-select,
.value-input {
  /* padding: 8px 12px; */
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

button {
  background-color: #4a1e9e;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #7c3aed;
}

.logo {
  max-height: 100vw;
  width: auto;
  object-fit: contain;
  display: block;
  margin: 0 auto;
}

/* Add these new styles */
.column-chooser {
  margin-bottom: 20px;
}

.checkbox-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 10px;
  padding: 10px;
}

.column-checkbox {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
}

.column-checkbox input[type="checkbox"] {
  width: 16px;
  height: 16px;
  cursor: pointer;
}

.column-settings-btn {
  justify-self: end;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  min-width: 500px;
  max-width: 90%;
  max-height: 90vh;
  overflow-y: auto;
  position: relative;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 15px;
  border-bottom: 1px solid #eee;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #666;
  padding: 0;
}

.close-btn:hover {
  color: #333;
  background: none;
}

.modal-footer {
  border-top: 1px solid #eee;
  padding-top: 15px;
  margin-top: 15px;
  display: flex;
  justify-content: flex-end;
}

.checkbox-container {
  max-height: 400px;
  overflow-y: auto;
}

/* Add these new styles */
.action-buttons {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-bottom: 0;
}

.modal-body {
  padding: 20px 0;
}

.filter-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr auto;
  gap: 10px;
  margin-bottom: 10px;
  padding: 10px;
  background: #f8f9fa;
  border-radius: 8px;
}

.footer-right {
  display: flex;
  gap: 10px;
}

.secondary-btn {
  background-color: #6b7280;
}

.secondary-btn:hover {
  background-color: #4b5563;
}

.primary-btn {
  background-color: #4a1e9e;
}

.primary-btn:hover {
  background-color: #7c3aed;
}

.remove-btn {
  background-color: #dc2626;
}

.remove-btn:hover {
  background-color: #b91c1c;
}

.field-select,
.operator-select,
.value-input {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  width: 100%;
}

/* Update these styles */
.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  border-bottom: 1px solid #eee;
}

.table-header h3 {
  margin: 0;
  font-size: 1.2rem;
}

.right-section {
  display: flex;
  gap: 10px;
  align-items: center;
}

.column-settings-btn,
.query-builder-btn {
  height: 36px;
  padding: 0 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Add responsive adjustments for the table container */
.table-container {
  width: 100%;
  overflow-x: auto; /* Allows horizontal scrolling on mobile */
  -webkit-overflow-scrolling: touch; /* Smooth scrolling on iOS */
}

/* Adjust filter row layout for mobile */
@media screen and (max-width: 768px) {
  .filter-row {
    grid-template-columns: 1fr; /* Stack filters vertically on mobile */
    gap: 15px;
  }

  .modal-content {
    min-width: 90%; /* Adjust modal width for mobile */
    margin: 10px;
  }

  .checkbox-container {
    grid-template-columns: 1fr; /* Single column for checkboxes on mobile */
  }

  /* Adjust table header buttons for mobile */
  .table-header {
    flex-direction: column;
    gap: 15px;
  }

  .right-section {
    width: 100%;
    justify-content: space-between;
  }

  .column-settings-btn,
  .query-builder-btn {
    width: 48%; /* Make buttons fill space evenly */
    font-size: 14px;
  }
}

/* Add new styles */
.error-message {
  background-color: #fee2e2;
  border: 1px solid #ef4444;
  color: #dc2626;
  padding: 12px;
  border-radius: 6px;
  margin: 20px;
  text-align: center;
}

.empty-state {
  text-align: center;
  padding: 40px;
  color: #6b7280;
  font-style: italic;
}

.loading-state {
  text-align: center;
  padding: 40px;
  color: #4a1e9e;
}

/* Add these new styles */
.sortable-header {
  cursor: pointer;
  user-select: none;
  position: relative;
  padding-right: 25px; /* Make room for the sort indicator */
}

.sortable-header:hover {
  background-color: #5f35b4;
}

.sort-indicator {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
}
</style>
