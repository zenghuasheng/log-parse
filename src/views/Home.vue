<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { useRouter } from 'vue-router'

const router = useRouter()

const logFile = ref(null)
const logPattern = ref('^(\\S+) - - \\[(.*?)\\] "(.*?)" (\\d+) (\\d+) "(.*?)" "(.*?)" "(.*?)" "([\\d\\.]+)"$')
const schemaFields = ref('ip_address:ID,timestamp:TEXT,endpoint:TEXT,status:NUMERIC,size:NUMERIC,referer:TEXT,user_agent:TEXT,request_ip:ID,response_time:NUMERIC')
const md5Hash = ref('')
const first100Parsed = ref(null)
const error = ref(null)
const parse_finish = ref(false)
const total_lines_parsed = ref(null)

const parsedLogList = ref([])

// Fetch parsed log list from backend on component mount
onMounted(async () => {
  try {
    const response = await fetch('/api/parsed-log-list')
    if (!response.ok) {
      throw new Error('Failed to fetch parsed log list')
    }
    parsedLogList.value = await response.json()
  } catch (error) {
    console.error('Error fetching parsed log list:', error)
  }
})

const handleFileUpload = (event) => {
  logFile.value = event.target.files[0]
}

const uploadLogFile = async () => {
  if (!logFile.value || !logPattern.value || !schemaFields.value) {
    error.value = 'Please provide log file, pattern, and schema fields'
    return
  }

  const formData = new FormData()
  formData.append('logfile', logFile.value)
  formData.append('log_pattern', logPattern.value)
  formData.append('fields', schemaFields.value)

  try {
    const response = await axios.post('/api/upload', formData, {
      headers: {
        'Content-Type': 'multipart/form-data'
      }
    })
    if (response.data.error) {
      error.value = response.data.error
      return
    }
    md5Hash.value = response.data.md5
    first100Parsed.value = response.data.first_100_parsed
    error.value = null
  } catch (err) {
    error.value = 'Error uploading file: ' + err.message
  }
}

const confirmParse = async () => {
  if (!md5Hash.value || !logPattern.value || !schemaFields.value) {
    error.value = 'Please provide MD5 hash, pattern, and schema fields'
    return
  }

  const formData = new FormData()
  formData.append('md5', md5Hash.value)
  formData.append('log_pattern', logPattern.value)
  formData.append('fields', schemaFields.value)

  try {
    const response = await axios.post('/api/confirm', formData)
    error.value = response.data.error
    if (error.value) {
      return
    }
    parse_finish.value = true
    total_lines_parsed.value = response.data.total_lines_parsed
  } catch (err) {
    error.value = 'Error confirming parse: ' + err.message
  }
}
</script>

<template>
  <div class="container">
    <div>
      <h3>已解析日志列表</h3>
      <ul>
        <li v-for="log in parsedLogList" :key="log.hash">
          <router-link :to="{ name: 'Detail', params: { hash: log.hash } }">{{ log.filename?log.filename:log.hash }}</router-link>
        </li>
      </ul>
    </div>
    <h3>解析日志</h3>
    <input type="file" @change="handleFileUpload" />
    <input type="text" v-model="logPattern" placeholder="Log Pattern" />
    <input type="text" v-model="schemaFields" placeholder="Schema Fields" />
    <button @click="uploadLogFile">Upload</button>

    <div v-if="error" class="error">
      <pre>{{ error }}</pre>
    </div>
    <div v-if="parse_finish">
      <span>{{ total_lines_parsed }} parsed.</span>
      <span>
        <router-link :to="{ name: 'Detail', params: { hash: md5Hash } }">日志详情页</router-link>
      </span>
    </div>

    <div v-if="first100Parsed">
      <h2>First 10 Parsed Lines</h2>
      <button @click="confirmParse">Confirm and Parse All</button>
      <pre style="text-align: left">{{ first100Parsed }}</pre>
    </div>
  </div>
</template>

<style scoped>
.error {
  color: red;
  text-align: left;
}
.container{
  width: 70rem;
}
input {
  display: block;
  width: 100%;
  height: 26px;
  margin-bottom: 10px;
}
</style>
