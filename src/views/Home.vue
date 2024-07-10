<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { useRouter } from 'vue-router'

const router = useRouter()

const logFile = ref(null)
const logPattern = ref('')
const schemaFields = ref('')
const formatFields = ref('')
const parse_example = ('')


const logTypeOptions = ref([
  {
    name: 'nginx',
    logPattern: '^(\\S+) - - \\[(.*?)\\] "(.*?)" (\\d+) (\\d+) "(.*?)" "(.*?)" "(.*?)" "([\\d\\.]+)"$',
    schemaFields: 'ip_address:ID,timestamp:TEXT,endpoint:TEXT,status:NUMERIC,size:NUMERIC,referer:TEXT,user_agent:TEXT,request_ip:ID,response_time:NUMERIC',
    formatFields: `
def format_fields(match, schema_fields):
    line_map = {}
    for i, field in enumerate(schema_fields):
        field_value = match.group(i + 1)
        if schema_fields[field] == "NUMERIC":
            # parsed_lines.append({field: int(field_value)})
            # 尝试转为 int,如果失败，转为 float
            try:
                field_value = int(field_value)
                line_map[field] = field_value
            except ValueError:
                field_value = float(field_value)
                field_value = int(field_value * 1000)
                line_map[field] = field_value
        else:
            line_map[field] = field_value
    return line_map
    `
  },
  {
    name: 'project-api',
    logPattern: '\\[(.*?)\\] (\\d{4}/\\d{2}/\\d{2} \\d{2}:\\d{2}:\\d{2}) (\\d{2}/\\d{2} - \\d{2}:\\d{2}:\\d{2}) (\\d{3}) (\\S+) (\\S+) (\\S+) (.*$)',
    schemaFields: 'log_level:ID,log_time1:TEXT,log_time2:TEXT,status:NUMERIC,response_time:NUMERIC,request_ip:ID,method:ID,endpoint:TEXT',
    formatFields: `
def format_fields(match, schema_fields):
    line_map = {}
    for i, field in enumerate(schema_fields):
        field_value = match.group(i + 1)
        if schema_fields[field] == "NUMERIC":
            # parsed_lines.append({field: int(field_value)})
            # 尝试转为 int,如果失败，转为 float
            if field == "response_time":
                # 带有单位，比如 1.23µs 1.23ms 1.23s 1min2s
                if "µs" in field_value:
                    field_value = field_value.replace("µs", "")
                    field_value = float(field_value)
                    field_value = int(field_value)
                elif "ms" in field_value:
                    field_value = field_value.replace("ms", "")
                    field_value = float(field_value)
                    field_value = int(field_value)
                elif "s" in field_value:
                    field_value = field_value.replace("s", "")
                    field_value = float(field_value)
                    field_value = int(field_value * 1000)
                elif "min" in field_value:
                    field_value = field_value.replace("min", "")
                    field_value = field_value.split("s")
                    field_value = int(field_value[0]) * 60 + int(field_value[1])
                line_map[field] = field_value
            else:
                try:
                    field_value = int(field_value)
                    line_map[field] = field_value
                except ValueError:
                    field_value = float(field_value)
                    field_value = int(field_value * 1000)
                    line_map[field] = field_value
        else:
            line_map[field] = field_value
    return line_map
`
  }
])

const selectedLogType = ref(0)
const onLogTypeChange = () => {
  const selectedOption = logTypeOptions.value[selectedLogType.value]
  logPattern.value = selectedOption.logPattern
  schemaFields.value = selectedOption.schemaFields
  formatFields.value = selectedOption.formatFields
}


const md5Hash = ref('')
const first100Parsed = ref(null)
const error = ref(null)
const parse_finish = ref(false)
const total_lines_parsed = ref(null)

const parsedLogList = ref([])

// Fetch parsed log list from backend on component mount
onMounted(async () => {
  fetchParsedLogs()
  onLogTypeChange()
})

const fetchParsedLogs = () => {
  axios.get('/api/parsed-log-list')
      .then(response => {
        parsedLogList.value = response.data
      })
      .catch(error => {
        error.value = error.response.data.error || 'Failed to fetch parsed logs'
      })
}


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
  formData.append('format_fields', formatFields.value)

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
  formData.append('format_fields', formatFields.value)

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
    <select v-model="selectedLogType" @change="onLogTypeChange">
      <option v-for="(option, index) in logTypeOptions" :key="index" :value="index">
        {{ option.name }}
      </option>
    </select>
    <input type="file" @change="handleFileUpload" />
    <input type="text" v-model="logPattern" placeholder="Log Pattern" />
    <input type="text" v-model="schemaFields" placeholder="Schema Fields" />
    <textarea class="format-field-code" v-model="formatFields" placeholder="Format Fields Function" />
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
.format-field-code {
  height: 200px;
  width: 100%;
  margin-bottom: 10px;
}
</style>
