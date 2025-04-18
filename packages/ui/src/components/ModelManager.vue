<template>
  <div
    class="fixed inset-0 theme-mask z-[60] flex items-center justify-center overflow-y-auto"
    @click="$emit('close')"
  >
    <div
      class="relative theme-manager-container w-full max-w-3xl m-4"
      @click.stop
    >
      <div class="p-6 space-y-6">
        <!-- 标题和关闭按钮 -->
        <div class="flex items-center justify-between">
          <h2 class="text-xl font-semibold theme-manager-text">模型设置</h2>
          <button
            @click="$emit('close')"
            class="theme-manager-text-secondary hover:theme-manager-text transition-colors text-xl"
          >
            ×
          </button>
        </div>

        <!-- 已启用模型列表 -->
        <div class="space-y-3">
          <h3 class="text-lg font-semibold theme-manager-text">模型列表</h3>
          <div class="space-y-3">
            <div v-for="model in models" :key="model.key" 
                 :class="['p-4 rounded-xl border transition-colors',
                         model.enabled 
                           ? 'theme-manager-card' 
                           : 'theme-manager-card opacity-50 shadow-none hover:shadow-none']">
              <div class="flex items-center justify-between">
                <div>
                  <div class="flex items-center gap-2">
                    <h4 class="font-medium" :class="model.enabled ? 'theme-manager-text' : 'theme-manager-text-disabled'">
                      {{ model.name }}
                    </h4>
                    <span v-if="!model.enabled" 
                          class="theme-manager-tag">
                      已禁用
                    </span>
                  </div>
                  <p class="text-sm" :class="model.enabled ? 'theme-manager-text-secondary' : 'theme-manager-text-disabled'">
                    {{ model.model }}
                  </p>
                </div>
                <div class="flex items-center space-x-2">
                  <button @click="testConnection(model.key)"
                          class="theme-manager-button-test">
                    测试连接
                  </button>
                  <button @click="editModel(model.key)"
                          class="theme-manager-button-edit">
                    编辑
                  </button>
                  <button @click="model.enabled ? disableModel(model.key) : enableModel(model.key)"
                          :class="[
                            model.enabled 
                              ? 'theme-manager-button-warning' 
                              : 'theme-manager-button-success'
                          ]">
                    {{ model.enabled ? '禁用' : '启用' }}
                  </button>
                  <!-- 只对自定义模型显示删除按钮 -->
                  <button v-if="!isDefaultModel(model.key)" 
                          @click="handleDelete(model.key)"
                          class="theme-manager-button-danger">
                    删除
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 使用 Teleport 将模态框传送到 body -->
        <Teleport to="body">
          <!-- 编辑模型弹窗 -->
          <div v-if="isEditing" 
               class="fixed inset-0 z-[60] flex items-center justify-center overflow-y-auto"
               @click="cancelEdit">
            <div class="fixed inset-0 bg-black/60 backdrop-blur-sm"></div>
            
            <div class="relative theme-manager-container w-full max-w-2xl m-4 z-10"
                 @click.stop>
              <div class="p-6 space-y-6">
                <div class="flex items-center justify-between">
                  <h3 class="text-xl font-semibold theme-manager-text">编辑模型</h3>
                  <button
                    @click="cancelEdit"
                    class="theme-manager-text-secondary hover:theme-manager-text transition-colors text-xl"
                  >
                    ×
                  </button>
                </div>
                
                <form @submit.prevent="saveEdit" class="space-y-4">
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">显示名称</label>
                    <input v-model="editingModel.name" type="text" required
                           class="theme-manager-input"
                           placeholder="例如: 自定义模型"/>
                  </div>
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">API 地址</label>
                    <input v-model="editingModel.baseURL" type="url" required
                           class="theme-manager-input"
                           placeholder="https://api.example.com/v1"/>
                  </div>
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">默认模型名称</label>
                    <input v-model="editingModel.defaultModel" type="text" required
                           class="theme-manager-input"
                           placeholder="例如: gpt-3.5-turbo"/>
                  </div>
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">API 密钥</label>
                    <input v-model="editingModel.apiKey" type="password"
                           class="theme-manager-input"
                           placeholder="输入新的 API 密钥（留空则保持不变）"/>
                  </div>
                  <div v-if="vercelProxyAvailable" class="flex items-center space-x-2">
                    <input 
                      :id="`vercel-proxy-${editingModel.key}`" 
                      v-model="editingModel.useVercelProxy" 
                      type="checkbox"
                      class="w-4 h-4 text-purple-600 bg-black/20 border-purple-600/50 rounded focus:ring-purple-500/50"
                    />
                    <label :for="`vercel-proxy-${editingModel.key}`" class="text-sm font-medium theme-manager-text">
                      使用Vercel代理 (解决跨域问题，有一定风险，请谨慎使用)
                    </label>
                  </div>
                  <div class="flex justify-end space-x-3 pt-4">
                    <button
                      type="button"
                      @click="cancelEdit"
                      class="theme-manager-button-secondary"
                    >
                      取消
                    </button>
                    <button
                      type="submit"
                      class="theme-manager-button-primary"
                    >
                      保存修改
                    </button>
                  </div>
                </form>
              </div>
            </div>
          </div>

          <!-- 添加模型弹窗 -->
          <div v-if="showAddForm" 
               class="fixed inset-0 z-[60] flex items-center justify-center overflow-y-auto"
               @click="showAddForm = false">
            <div class="fixed inset-0 bg-black/60 backdrop-blur-sm"></div>
            
            <div class="relative theme-manager-container w-full max-w-2xl m-4 z-10"
                 @click.stop>
              <div class="p-6 space-y-6">
                <div class="flex items-center justify-between">
                  <h3 class="text-xl font-semibold theme-manager-text">添加自定义模型</h3>
                  <button
                    @click="showAddForm = false"
                    class="theme-manager-text-secondary hover:theme-manager-text transition-colors text-xl"
                  >
                    ×
                  </button>
                </div>
                
                <form @submit.prevent="addCustomModel" class="space-y-4">
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">模型标识</label>
                    <input v-model="newModel.key" type="text" required
                           class="theme-manager-input"
                           placeholder="例如: custom-model"/>
                  </div>
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">显示名称</label>
                    <input v-model="newModel.name" type="text" required
                           class="theme-manager-input"
                           placeholder="例如: 自定义模型"/>
                  </div>
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">API 地址</label>
                    <input v-model="newModel.baseURL" type="url" required
                           class="theme-manager-input"
                           placeholder="https://api.example.com/v1"/>
                  </div>
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">默认模型名称</label>
                    <input v-model="newModel.defaultModel" type="text" required
                           class="theme-manager-input"
                           placeholder="例如: gpt-3.5-turbo"/>
                  </div>
                  <div>
                    <label class="block text-sm font-medium theme-manager-text mb-1.5">API 密钥</label>
                    <input v-model="newModel.apiKey" type="password" required
                           class="theme-manager-input"
                           placeholder="输入 API 密钥"/>
                  </div>
                  <div v-if="vercelProxyAvailable" class="flex items-center space-x-2">
                    <input 
                      id="new-model-vercel-proxy" 
                      v-model="newModel.useVercelProxy" 
                      type="checkbox"
                      class="w-4 h-4 text-purple-600 bg-black/20 border-purple-600/50 rounded focus:ring-purple-500/50"
                    />
                    <label for="new-model-vercel-proxy" class="text-sm font-medium theme-manager-text">
                      使用Vercel代理 (解决跨域问题，有一定风险，请谨慎使用)
                    </label>
                  </div>
                  <div class="flex justify-end space-x-3 pt-4">
                    <button
                      type="button"
                      @click="showAddForm = false"
                      class="theme-manager-button-secondary"
                    >
                      取消
                    </button>
                    <button
                      type="submit"
                      class="theme-manager-button-primary"
                    >
                      添加模型
                    </button>
                  </div>
                </form>
              </div>
            </div>
          </div>
        </Teleport>

        <!-- 添加模型按钮 -->
        <div class="mt-2 theme-manager-divider pt-2">
          <button
            @click="showAddForm = true"
            class="w-full theme-manager-button-secondary py-3 flex items-center justify-center space-x-2"
          >
            <span>+</span>
            <span>添加自定义模型</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, defineEmits } from 'vue';
import { modelManager, createLLMService, checkVercelApiAvailability, resetVercelStatusCache } from '@prompt-optimizer/core';
import { useToast } from '../composables/useToast';

const toast = useToast();
const emit = defineEmits(['modelsUpdated', 'close', 'select']);

const models = ref([]);
const isEditing = ref(false);
const showAddForm = ref(false);
const editingModel = ref(null);
const newModel = ref({
  key: '',
  name: '',
  baseURL: '',
  defaultModel: '',
  apiKey: '',
  useVercelProxy: false
});
// 是否支持Vercel代理
const vercelProxyAvailable = ref(false);

// 检测Vercel代理是否可用
const checkVercelProxy = async () => {
  try {
    // 先重置缓存，确保每次都重新检测
    resetVercelStatusCache();
    // 使用core中的检测函数
    const available = await checkVercelApiAvailability();
    vercelProxyAvailable.value = available;
    console.log('Vercel代理检测结果:', vercelProxyAvailable.value);
  } catch (error) {
    console.log('Vercel代理不可用:', error);
    vercelProxyAvailable.value = false;
  }
};

// 加载所有模型
const loadModels = () => {
  try {
    // 强制刷新模型数据
    const allModels = modelManager.getAllModels();
    
    // 使用深拷贝确保响应式更新
    models.value = JSON.parse(JSON.stringify(allModels)).sort((a, b) => {
      // 启用的模型排在前面
      if (a.enabled !== b.enabled) {
        return a.enabled ? -1 : 1;
      }
      // 默认模型排在前面
      if (isDefaultModel(a.key) !== isDefaultModel(b.key)) {
        return isDefaultModel(a.key) ? -1 : 1;
      }
      return 0;
    });
    
    console.log('处理后的模型列表:', models.value.map(m => ({
      key: m.key,
      name: m.name,
      enabled: m.enabled,
      hasApiKey: !!m.apiKey
    })));
    
    emit('modelsUpdated', models.value[0]?.key);
  } catch (error) {
    console.error('加载模型列表失败:', error);
    toast.error('加载模型列表失败');
  }
};

// 测试连接
const testConnection = async (key) => {
  try {
    const model = modelManager.getModel(key);
    if (!model) throw new Error('模型配置不存在');

    const llm = createLLMService(modelManager);
    await llm.testConnection(key);
    toast.success('连接测试成功');
  } catch (error) {
    console.error('连接测试失败:', error);
    toast.error(`连接测试失败: ${error.message}`);
  }
};

// 判断是否为默认模型
const isDefaultModel = (key) => {
  return ['openai', 'gemini', 'deepseek'].includes(key);
};

// 处理删除
const handleDelete = async (key) => {
  if (confirm(`确定要删除模型 ${key} 吗？此操作不可恢复。`)) {
    try {
      modelManager.deleteModel(key)
      loadModels()
      toast.success('模型已删除')
    } catch (error) {
      console.error('删除模型失败:', error)
      toast.error(`删除模型失败: ${error.message}`)
    }
  }
};

// 编辑模型
const editModel = (key) => {
  const model = modelManager.getModel(key);
  if (model) {
    editingModel.value = {
      key,
      name: model.name,
      baseURL: model.baseURL,
      defaultModel: model.defaultModel,
      apiKey: '',  // 不显示原有的 API 密钥
      useVercelProxy: model.useVercelProxy
    };
    isEditing.value = true;
  }
};

// 取消编辑
const cancelEdit = () => {
  isEditing.value = false;
  editingModel.value = null;
};

// 保存编辑
const saveEdit = async () => {
  console.log('开始保存编辑...');
  console.log('编辑的模型数据:', editingModel.value);
  
  try {
    const originalConfig = modelManager.getModel(editingModel.value.key)
    if (!originalConfig) {
      throw new Error('找不到原始配置')
    }
    console.log('原始配置:', originalConfig);

    const config = {
      name: editingModel.value.name,
      baseURL: editingModel.value.baseURL,
      models: [editingModel.value.defaultModel],
      defaultModel: editingModel.value.defaultModel,
      apiKey: editingModel.value.apiKey.trim() || originalConfig.apiKey,
      enabled: originalConfig.enabled,
      provider: originalConfig.provider,
      useVercelProxy: editingModel.value.useVercelProxy
    }
    console.log('新配置:', config);

    modelManager.updateModel(editingModel.value.key, config)
    
    loadModels()
    
    emit('modelsUpdated', editingModel.value.key)
    console.log('已触发 modelsUpdated 事件');
    
    isEditing.value = false
    editingModel.value = null
        
    toast.success('模型配置已更新')
  } catch (error) {
    console.error('更新模型失败:', error)
    toast.error(`更新模型失败: ${error.message}`)
  }
};

// 添加自定义模型
const addCustomModel = async () => {
  try {
    const config = {
      name: newModel.value.name,
      baseURL: newModel.value.baseURL,
      models: [newModel.value.defaultModel],
      defaultModel: newModel.value.defaultModel,
      apiKey: newModel.value.apiKey,
      enabled: true,
      provider: 'custom',
      useVercelProxy: newModel.value.useVercelProxy
    }

    modelManager.addModel(newModel.value.key, config)
    loadModels()
    showAddForm.value = false
    // 修改这里，传递新添加的模型的 key
    emit('modelsUpdated', newModel.value.key)
    newModel.value = {
      key: '',
      name: '',
      baseURL: '',
      defaultModel: '',
      apiKey: '',
      useVercelProxy: false
    }
    toast.success('模型添加成功')
  } catch (error) {
    console.error('添加模型失败:', error)
    toast.error(`添加模型失败: ${error.message}`)
  }
};

// 启用/禁用模型
const enableModel = async (key) => {
  try {
    const model = modelManager.getModel(key)
    if (!model) throw new Error('模型配置不存在')

    modelManager.enableModel(key)
    loadModels()
    // 修改这里，传递启用的模型的 key
    emit('modelsUpdated', key)
    toast.success('模型已启用')
  } catch (error) {
    console.error('启用模型失败:', error)
    toast.error(`启用模型失败: ${error.message}`)
  }
}

const disableModel = async (key) => {
  try {
    const model = modelManager.getModel(key)
    if (!model) throw new Error('模型配置不存在')

    modelManager.disableModel(key)
    loadModels()
    // 修改这里，传递禁用的模型的 key
    emit('modelsUpdated', key)
    toast.success('模型已禁用')
  } catch (error) {
    console.error('禁用模型失败:', error)
    toast.error(`禁用模型失败: ${error.message}`)
  }
}


// 初始化
onMounted(() => {
  loadModels();
  checkVercelProxy();
});
</script>

<style scoped>
/* 添加过渡动画 */
.modal-enter-active,
.modal-leave-active {
  transition: all 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
  transform: scale(0.95);
}
</style>