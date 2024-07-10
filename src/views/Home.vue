<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { useRouter } from 'vue-router'

const router = useRouter()

const logFile = ref(null)
const logPattern = ref('^(\\S+) - - \\[(.*?)\\] "(.*?)" (\\d+) (\\d+) "(.*?)" "(.*?)" "(.*?)" "([\\d\\.]+)"$')
const schemaFields = ref('ip_address:ID,timestamp:TEXT,endpoint:TEXT,status:NUMERIC,size:NUMERIC,referer:TEXT,user_agent:TEXT,request_ip:ID,response_time:NUMERIC')
const formatFields = ref('def format_fields(match, schema_fields):\n' +
    '    line_map = {}\n' +
    '    for i, field in enumerate(schema_fields):\n' +
    '        field_value = match.group(i + 1)\n' +
    '        if schema_fields[field] == "NUMERIC":\n' +
    '            # parsed_lines.append({field: int(field_value)})\n' +
    '            # 尝试转为 int,如果失败，转为 float\n' +
    '            try:\n' +
    '                field_value = int(field_value)\n' +
    '                line_map[field] = field_value\n' +
    '            except ValueError:\n' +
    '                field_value = float(field_value)\n' +
    '                field_value = int(field_value * 1000)\n' +
    '                line_map[field] = field_value\n' +
    '        else:\n' +
    '            line_map[field] = field_value\n' +
    '    return line_map\n')
const parse_example = ref(
'project-api:'+
    '\n\\[(.*?)\\] (\\d{4}/\\d{2}/\\d{2} \\d{2}:\\d{2}:\\d{2}) (\\d{2}/\\d{2} - \\d{2}:\\d{2}:\\d{2}) (\\d{3}) (\\d{1,}.\\d{1,})(m?s) (\\S+) (\\S+) (.*$)' +
    '\nlog_level:ID,log_time1:TEXT,log_time2:TEXT,status:NUMERIC,response_time:NUMERIC,request_ip:ID,method:ID,endpoint:TEXT' +
    '\ndef format_fields(match, schema_fields):\n' +
    '    line_map = {}\n' +
    '    for i, field in enumerate(schema_fields):\n' +
    '        field_value = match.group(i + 1)\n' +
    '        if schema_fields[field] == "NUMERIC":\n' +
    '            # parsed_lines.append({field: int(field_value)})\n' +
    '            # 尝试转为 int,如果失败，转为 float\n' +
    '            if field == "response_time":\n' +
    '                # 带有单位，比如 1.23ms 1.23s 1min2s\n' +
    '                if "ms" in field_value:\n' +
    '                    field_value = field_value.replace("ms", "")\n' +
    '                    field_value = float(field_value)\n' +
    '                    field_value = int(field_value)\n' +
    '                elif "s" in field_value:\n' +
    '                    field_value = field_value.replace("s", "")\n' +
    '                    field_value = float(field_value)\n' +
    '                    field_value = int(field_value * 1000)\n' +
    '                elif "min" in field_value:\n' +
    '                    field_value = field_value.replace("min", "")\n' +
    '                    field_value = field_value.split("s")\n' +
    '                    field_value = int(field_value[0]) * 60 + int(field_value[1])\n' +
    '                line_map[field] = field_value\n' +
    '            else:\n' +
    '                try:\n' +
    '                    field_value = int(field_value)\n' +
    '                    line_map[field] = field_value\n' +
    '                except ValueError:\n' +
    '                    field_value = float(field_value)\n' +
    '                    field_value = int(field_value * 1000)\n' +
    '                    line_map[field] = field_value\n' +
    '        else:\n' +
    '            line_map[field] = field_value\n' +
    '    return line_map\n'
)
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
    <input type="file" @change="handleFileUpload" />
    <input type="text" v-model="logPattern" placeholder="Log Pattern" />
    <input type="text" v-model="schemaFields" placeholder="Schema Fields" />
    <textarea class="format-field-code" v-model="formatFields" placeholder="Format Fields Function" />
    <button @click="uploadLogFile">Upload</button>
    <div>
      <a href="https://regex101.com/" target="_blank">正则表达式网站</a>
      <pre>
        {{ parse_example }}
      </pre>
    </div>

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
