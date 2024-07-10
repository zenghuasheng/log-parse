<template>
  <div>
    <h1>Log List</h1>
    <p>MD5 Hash: {{ hash }}</p>

    <div>
      <label for="search">Search: </label>
      <input id="search" v-model="query" @input="fetchLogs" />
    </div>

    <div>
      <label for="sortField">Sort by: </label>
      <select id="sortField" v-model="sortField" @change="fetchLogs">
        <option v-for="field in schemaFields" :key="field" :value="field">
          {{ field }}
        </option>
      </select>

      <label for="isReverse">Reverse: </label>
      <input id="isReverse" type="checkbox" v-model="isReverse" @change="fetchLogs" />
    </div>

    <div v-if="error" class="error">{{ error }}</div>

    <div v-if="totalCount !== null">Total Count: {{ totalCount }}</div>

    <table v-if="logs.length">
      <thead>
      <tr>
        <th v-for="field in schemaFields" :key="field">{{ field }}</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="log in logs" :key="log.id">
        <td v-for="field in schemaFields" :key="field">{{ log[field] }}</td>
      </tr>
      </tbody>
    </table>

    <div v-else>No logs found</div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import axios from 'axios'

const route = useRoute()
const hash = route.params.hash
const query = ref('')
const sortField = ref('')
const isReverse = ref(false)
const logs = ref([])
const schemaFields = ref([])
const totalCount = ref(null)
const error = ref(null)

const fetchLogs = async () => {
  try {
    error.value = null
    const response = await axios.get('/api/list', {
      params: {
        md5: hash,
        q: query.value,
        sort_field: sortField.value,
        is_reverse: isReverse.value
      }
    })
    logs.value = response.data.results
    totalCount.value = response.data.total_count
    schemaFields.value = response.data.schema_fields
  } catch (err) {
    error.value = err.response ? err.response.data.error : err.message
  }
}

onMounted(fetchLogs)
</script>

<style scoped>
.error {
  color: red;
}
table {
  width: 100%;
  border-collapse: collapse;
}
th, td {
  border: 1px solid black;
  padding: 8px;
  text-align: left;
  max-width: 120px;
  word-wrap: break-word;
}
</style>
