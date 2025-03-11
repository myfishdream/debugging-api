<script setup>
import { ref, reactive, onMounted } from 'vue';
import axios from 'axios';
import { message } from 'ant-design-vue';
import { HistoryOutlined, SaveOutlined, CopyOutlined } from '@ant-design/icons-vue';

const requestMethod = ref('GET');
const requestUrl = ref('');
const activeTab = ref('params');
const bodyType = ref('none');
const bodyContent = ref('');
const loading = ref(false);
const serverConfigVisible = ref(false);
const useProxy = ref(false);
const serverConfig = reactive({
  baseUrl: localStorage.getItem('serverBaseUrl') || '',
});

// 历史记录相关
const historyVisible = ref(false);
const historyUrls = ref(JSON.parse(localStorage.getItem('historyUrls') || '[]'));
const maxHistoryLength = 20; // 最大历史记录数量

// 保存当前请求到历史记录
const saveToHistory = () => {
  if (!requestUrl.value) return;
  
  const newHistory = {
    url: requestUrl.value,
    method: requestMethod.value,
    timestamp: Date.now(),
    // 保存请求模板数据
    template: {
      params: params.value,
      headers: headers.value,
      bodyType: bodyType.value,
      bodyContent: bodyContent.value
    }
  };
  
  // 检查是否已存在相同的URL
  const existingIndex = historyUrls.value.findIndex(item => item.url === requestUrl.value);
  if (existingIndex !== -1) {
    // 如果存在，更新时间戳和请求方法
    historyUrls.value[existingIndex] = newHistory;
  } else {
    // 如果不存在，添加到开头
    historyUrls.value.unshift(newHistory);
    // 限制历史记录数量
    if (historyUrls.value.length > maxHistoryLength) {
      historyUrls.value = historyUrls.value.slice(0, maxHistoryLength);
    }
  }
  
  // 保存到localStorage
  localStorage.setItem('historyUrls', JSON.stringify(historyUrls.value));
  message.success('已保存到历史记录');
};

// 从历史记录中删除
const removeFromHistory = (url) => {
  historyUrls.value = historyUrls.value.filter(item => item.url !== url);
  localStorage.setItem('historyUrls', JSON.stringify(historyUrls.value));
  message.success('已从历史记录中删除');
};

// 清空历史记录
const clearHistory = () => {
  historyUrls.value = [];
  localStorage.setItem('historyUrls', JSON.stringify([]));
  message.success('已清空历史记录');
};

// 选择历史记录
const selectHistory = (item) => {
  requestUrl.value = item.url;
  requestMethod.value = item.method;
  
  // 恢复请求模板数据
  if (item.template) {
    params.value = JSON.parse(JSON.stringify(item.template.params));
    headers.value = JSON.parse(JSON.stringify(item.template.headers));
    bodyType.value = item.template.bodyType;
    bodyContent.value = item.template.bodyContent;
  }
  
  historyVisible.value = false;
};

// 响应数据
const response = reactive({
  status: '-',
  time: '-',
  data: '',
  headers: {},
});

const methods = [
  { value: 'GET', label: 'GET' },
  { value: 'POST', label: 'POST' },
  { value: 'PUT', label: 'PUT' },
  { value: 'DELETE', label: 'DELETE' },
];

// 保存服务器配置
const saveServerConfig = () => {
  localStorage.setItem('serverBaseUrl', serverConfig.baseUrl);
  message.success('服务器配置已保存');
  serverConfigVisible.value = false;
};

// 参数管理
const params = ref([]);
const headers = ref([]);

const addRow = (type) => {
  const target = type === 'params' ? params : headers;
  target.value.push({
    key: '',
    value: '',
    id: Date.now(),
  });
};

const removeRow = (type, id) => {
  const target = type === 'params' ? params : headers;
  target.value = target.value.filter(item => item.id !== id);
};

const updateRow = (type, id, field, value) => {
  const target = type === 'params' ? params : headers;
  const index = target.value.findIndex(item => item.id === id);
  if (index !== -1) {
    target.value[index][field] = value;
  }
};

// 创建axios实例
const axiosInstance = axios.create({
  timeout: 30000,
  withCredentials: false,
});

// 添加请求拦截器
axiosInstance.interceptors.request.use(
  (config) => {
    let finalUrl = config.url;

    // 如果URL不是完整的HTTP URL，则添加baseUrl
    if (!finalUrl.startsWith('http')) {
      // 检查是否配置了服务器地址
      if (!serverConfig.baseUrl) {
        throw new Error('请先配置服务器地址');
      }
      
      // 移除baseUrl末尾的斜杠（如果有）
      const baseUrl = serverConfig.baseUrl.endsWith('/') 
        ? serverConfig.baseUrl.slice(0, -1) 
        : serverConfig.baseUrl;
      
      // 确保url开头有斜杠
      const path = finalUrl.startsWith('/') ? finalUrl : `/${finalUrl}`;
      
      // 根据代理模式决定是否使用代理
      if (useProxy.value) {
        finalUrl = `/api${path}`;
      } else {
        finalUrl = baseUrl + path;
      }
    } else if (useProxy.value) {
      // 如果是完整URL且启用代理，使用代理
      finalUrl = `/proxy/${encodeURIComponent(finalUrl)}`;
      // 移除可能导致问题的请求头
      delete config.headers['Origin'];
      delete config.headers['Referer'];
    }
    // 如果不使用代理，保持原始URL不变
    
    config.url = finalUrl;
    console.log('发送请求:', config);
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// 添加响应拦截器
axiosInstance.interceptors.response.use(
  (response) => {
    return response;
  },
  async (error) => {
    console.error('响应错误:', error);
    
    // 处理CORS错误，自动切换到代理模式重试
    if (error.message?.includes('CORS') && !useProxy.value) {
      console.warn('检测到CORS错误，自动切换到代理模式重试');
      useProxy.value = true; // 开启代理模式
      message.info('检测到CORS限制，已自动切换到代理模式重试请求');
      
      try {
        // 使用相同的配置重试请求
        return await axiosInstance(error.config);
      } catch (retryError) {
        console.error('代理模式重试失败:', retryError);
        throw retryError;
      }
    }
    
    if (error.response) {
      // 服务器返回了错误状态码
      console.error('响应错误:', error.response);
      return Promise.reject(error);
    } else if (error.request) {
      // 请求发送成功，但没有收到响应
      console.error('无响应错误:', error.request);
      error.message = useProxy.value
        ? '代理服务器无响应，请稍后重试'
        : '服务器无响应，请检查网络连接';
      return Promise.reject(error);
    } else {
      // 请求配置出错
      console.error('请求错误:', error.message);
      return Promise.reject(error);
    }
  }
);

// 发送请求
const sendRequest = async () => {
  if (!requestUrl.value) {
    message.error('请输入请求URL');
    return;
  }

  loading.value = true;
  const startTime = Date.now();

  try {
    // 处理请求参数
    const config = {
      method: requestMethod.value,
      url: requestUrl.value,
      headers: {},
      // 添加跨域相关配置
      withCredentials: false,
      validateStatus: function (status) {
        return status >= 200 && status < 500; // 默认值
      },
      maxRedirects: 5, // 允许最大重定向次数
    };

    // 添加Headers
    headers.value.forEach(header => {
      if (header.key && header.value) {
        config.headers[header.key] = header.value;
      }
    });

    // 添加Query参数
    if (params.value.length > 0) {
      const queryParams = new URLSearchParams();
      params.value.forEach(param => {
        if (param.key && param.value) {
          queryParams.append(param.key, param.value);
        }
      });
      if (!config.url.includes('?')) {
        config.url += `?${queryParams.toString()}`;
      } else {
        config.url += `&${queryParams.toString()}`;
      }
    }

    // 添加请求体
    if (bodyType.value !== 'none' && ['POST', 'PUT', 'PATCH'].includes(requestMethod.value)) {
      if (bodyType.value === 'json') {
        try {
          config.data = JSON.parse(bodyContent.value);
          config.headers['Content-Type'] = 'application/json';
        } catch (e) {
          message.error('JSON格式错误');
          loading.value = false;
          return;
        }
      } else if (bodyType.value === 'form-data') {
        const formData = new FormData();
        const formRows = bodyContent.value.split('\n');
        formRows.forEach(row => {
          const [key, value] = row.split(':').map(item => item.trim());
          if (key && value) {
            formData.append(key, value);
          }
        });
        config.data = formData;
        // 让浏览器自动设置正确的 Content-Type
        delete config.headers['Content-Type'];
      } else if (bodyType.value === 'x-www-form-urlencoded') {
        const params = new URLSearchParams();
        const formRows = bodyContent.value.split('\n');
        formRows.forEach(row => {
          const [key, value] = row.split('=').map(item => item.trim());
          if (key && value) {
            params.append(key, value);
          }
        });
        config.data = params;
        config.headers['Content-Type'] = 'application/x-www-form-urlencoded';
      } else {
        config.data = bodyContent.value;
        config.headers['Content-Type'] = 'text/plain';
      }
    }

    console.log('发送请求配置:', config);
    const res = await axiosInstance(config);
    
    // 更新响应数据
    response.status = res.status;
    response.time = `${Date.now() - startTime}ms`;
    response.headers = res.headers;
    
    // 处理响应数据
    if (typeof res.data === 'object') {
      response.data = JSON.stringify(res.data, null, 2);
    } else {
      try {
        // 尝试解析响应数据为JSON
        const parsedData = JSON.parse(res.data);
        response.data = JSON.stringify(parsedData, null, 2);
      } catch (e) {
        // 如果解析失败，直接显示原始数据
        response.data = res.data;
      }
    }

    message.success('请求成功');
  } catch (error) {
    console.error('请求错误：', error);
    response.status = error.response?.status || '错误';
    response.time = `${Date.now() - startTime}ms`;
    response.headers = error.response?.headers || {};
    
    // 处理错误响应
    if (error.response?.data) {
      if (typeof error.response.data === 'object') {
        response.data = JSON.stringify(error.response.data, null, 2);
      } else {
        response.data = error.response.data;
      }
    } else {
      response.data = error.message;
    }
    
    message.error('请求失败：' + error.message);
  } finally {
    loading.value = false;
  }
};

// 获取请求体placeholder
const getBodyPlaceholder = () => {
  switch (bodyType.value) {
    case 'json':
      return '请输入JSON格式的请求体\n例如：\n{\n  "key": "value"\n}';
    case 'form-data':
      return '请按照 key: value 格式输入，每行一个参数\n例如：\nname: 张三\nage: 18';
    case 'x-www-form-urlencoded':
      return '请按照 key=value 格式输入，每行一个参数\n例如：\nname=张三\nage=18';
    default:
      return '请输入请求体内容';
  }
};

// 复制响应体
const copyResponse = async () => {
  try {
    await navigator.clipboard.writeText(response.data);
    message.success('已复制到剪贴板');
  } catch (err) {
    message.error('复制失败，请手动复制');
  }
};
</script>

<template>
  <a-layout class="layout">
    <a-layout-header class="header">
      <div class="header-content">
        <div class="title">
          <h2>API 测试工具</h2>
          <span class="subtitle">简单、高效的接口调试工具</span>
        </div>
        <a-button type="primary" @click="serverConfigVisible = true">
          服务器配置
        </a-button>
      </div>
    </a-layout-header>
    
    <a-layout-content class="content">
      <div class="request-section">
        <a-space>
          <a-select
            v-model:value="requestMethod"
            style="width: 120px"
            :options="methods"
          />
          <a-input-search
            v-model:value="requestUrl"
            :placeholder="serverConfig.baseUrl ? '请输入请求路径' : '请输入完整URL或配置服务器地址'"
            enter-button="发送请求"
            :loading="loading"
            style="width: 600px"
            @search="sendRequest"
          >
            <template #addonAfter>
              <a-space>
                <a-button type="link" @click="historyVisible = true">
                  <template #icon><history-outlined /></template>
                </a-button>
                <a-button type="link" @click="saveToHistory">
                  <template #icon><save-outlined /></template>
                </a-button>
              </a-space>
            </template>
          </a-input-search>
          <a-tooltip title="开启代理模式可以解决CORS问题，但如果目标服务器已支持CORS则可以关闭">
            <a-switch
              v-model:checked="useProxy"
              checked-children="代理模式"
              un-checked-children="直连模式"
            />
          </a-tooltip>
        </a-space>

        <div class="params-section">
          <a-tabs v-model:activeKey="activeTab">
            <a-tab-pane key="params" tab="Params">
              <a-table
                :columns="[
                  { title: 'Key', dataIndex: 'key', width: '40%' },
                  { title: 'Value', dataIndex: 'value', width: '40%' },
                  { title: '操作', dataIndex: 'operation', width: '20%' }
                ]"
                :data-source="params"
                :pagination="false"
                bordered
              >
                <template #bodyCell="{ column, record }">
                  <template v-if="column.dataIndex === 'key'">
                    <a-input
                      :value="record.key"
                      @blur="(e) => record.key = e.target.value"
                    />
                  </template>
                  <template v-if="column.dataIndex === 'value'">
                    <a-input
                      :value="record.value"
                      @blur="(e) => record.value = e.target.value"
                    />
                  </template>
                  <template v-if="column.dataIndex === 'operation'">
                    <a-button type="link" danger @click="removeRow('params', record.id)">删除</a-button>
                  </template>
                </template>
              </a-table>
              <a-button type="dashed" block @click="addRow('params')">+ 添加参数</a-button>
            </a-tab-pane>

            <a-tab-pane key="headers" tab="Headers">
              <a-table
                :columns="[
                  { title: 'Key', dataIndex: 'key', width: '40%' },
                  { title: 'Value', dataIndex: 'value', width: '40%' },
                  { title: '操作', dataIndex: 'operation', width: '20%' }
                ]"
                :data-source="headers"
                :pagination="false"
                bordered
              >
                <template #bodyCell="{ column, record }">
                  <template v-if="column.dataIndex === 'key'">
                    <a-input
                      :value="record.key"
                      @blur="(e) => record.key = e.target.value"
                    />
                  </template>
                  <template v-if="column.dataIndex === 'value'">
                    <a-input
                      :value="record.value"
                      @blur="(e) => record.value = e.target.value"
                    />
                  </template>
                  <template v-if="column.dataIndex === 'operation'">
                    <a-button type="link" danger @click="removeRow('headers', record.id)">删除</a-button>
                  </template>
                </template>
              </a-table>
              <a-button type="dashed" block @click="addRow('headers')">+ 添加Header</a-button>
            </a-tab-pane>

            <a-tab-pane key="body" tab="Body">
              <a-radio-group v-model:value="bodyType" style="margin-bottom: 16px">
                <a-radio-button value="none">none</a-radio-button>
                <a-radio-button value="form-data">form-data</a-radio-button>
                <a-radio-button value="x-www-form-urlencoded">x-www-form-urlencoded</a-radio-button>
                <a-radio-button value="json">JSON</a-radio-button>
                <a-radio-button value="raw">raw</a-radio-button>
              </a-radio-group>
              <a-textarea
                v-model:value="bodyContent"
                :placeholder="getBodyPlaceholder()"
                :rows="8"
                :disabled="bodyType === 'none'"
              />
            </a-tab-pane>
          </a-tabs>
        </div>

        <div class="response-section">
          <a-typography-title :level="4">响应结果</a-typography-title>
          <a-card>
            <a-descriptions title="响应信息">
              <a-descriptions-item label="状态码">{{ response.status }}</a-descriptions-item>
              <a-descriptions-item label="响应时间">{{ response.time }}</a-descriptions-item>
            </a-descriptions>
            <a-tabs>
              <a-tab-pane key="body" tab="Body">
                <div style="display: flex; gap: 8px;">
                  <a-textarea
                    v-model:value="response.data"
                    readonly
                    :rows="7"
                    placeholder="响应内容将在这里显示"
                    style="flex: 1;"
                  />
                  <a-button type="primary" @click="copyResponse">
                    <template #icon><copy-outlined /></template>
                    复制
                  </a-button>
                </div>
              </a-tab-pane>
              <a-tab-pane key="headers" tab="Headers">
                <a-descriptions bordered>
                  <a-descriptions-item v-for="(value, key) in response.headers" :key="key" :label="key">
                    {{ value }}
                  </a-descriptions-item>
                </a-descriptions>
              </a-tab-pane>
            </a-tabs>
          </a-card>
        </div>
      </div>
    </a-layout-content>

    <!-- 服务器配置对话框 -->
    <a-modal
      v-model:open="serverConfigVisible"
      title="服务器配置"
      @ok="saveServerConfig"
      okText="保存"
      cancelText="取消"
    >
      <a-form layout="vertical">
        <a-form-item label="服务器地址">
          <a-input
            v-model:value="serverConfig.baseUrl"
            placeholder="请输入服务器地址，例如：http://000.000.00.000:8080"
          >
            <template #prefix>
              <span style="color: rgba(0, 0, 0, 0.45)">BaseURL:</span>
            </template>
          </a-input>
        </a-form-item>
      </a-form>
    </a-modal>

    <!-- 历史记录对话框 -->
    <a-modal
      v-model:open="historyVisible"
      title="历史记录"
      width="800px"
      @ok="historyVisible = false"
      okText="关闭"
      :cancelButtonProps="{ style: { display: 'none' } }"
    >
      <template #extra>
        <a-button danger @click="clearHistory">清空历史记录</a-button>
      </template>
      <a-list
        :data-source="historyUrls"
        :pagination="{ pageSize: 10 }"
        bordered
      >
        <template #renderItem="{ item }">
          <a-list-item>
            <a-list-item-meta>
              <template #title>
                <a-space>
                  <a-tag :color="item.method === 'GET' ? 'blue' : item.method === 'POST' ? 'green' : item.method === 'PUT' ? 'orange' : 'red'">
                    {{ item.method }}
                  </a-tag>
                  <a @click="selectHistory(item)" class="history-url">{{ item.url }}</a>
                </a-space>
              </template>
              <template #description>
                <div>{{ new Date(item.timestamp).toLocaleString() }}</div>
                <div v-if="item.template" style="color: #666; font-size: 12px;">
                  包含请求模板：{{ 
                    [
                      item.template.params.length > 0 && 'Params',
                      item.template.headers.length > 0 && 'Headers',
                      item.template.bodyType !== 'none' && 'Body'
                    ].filter(Boolean).join('、') || '无'
                  }}
                </div>
              </template>
            </a-list-item-meta>
            <template #actions>
              <a-button type="link" danger @click="removeFromHistory(item.url)">删除</a-button>
            </template>
          </a-list-item>
        </template>
      </a-list>
    </a-modal>
  </a-layout>
</template>

<style scoped>
.layout {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.header {
  background: #001529;
  padding: 0;
  height: 64px;
  line-height: 64px;
}

.header-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 100%;
}

.title {
  display: flex;
  flex-direction: column;
  line-height: 1.2;
}

.title h2 {
  color: white;
  margin: 0;
  font-size: 20px;
  font-weight: 600;
}

.subtitle {
  color: rgba(255, 255, 255, 0.65);
  font-size: 12px;
  margin-top: 2px;
}

.content {
  flex: 1;
  padding: 24px;
  background: #fff;
  overflow: auto;
}

.request-section {
  max-width: 1200px;
  margin: 0 auto;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.params-section {
  margin-top: 24px;
  border: 1px solid #f0f0f0;
  border-radius: 2px;
  padding: 16px;
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

.response-section {
  margin-top: 24px;
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

:deep(.ant-table-cell) {
  padding: 8px !important;
}

:deep(.ant-tabs-content-holder) {
  flex: 1;
  overflow: auto;
}

:deep(.ant-tabs) {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.history-url {
  color: #1890ff;
  cursor: pointer;
}

.history-url:hover {
  text-decoration: underline;
}
</style>

