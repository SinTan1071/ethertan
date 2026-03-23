<template>
  <div id="app">
    <div v-if="loading">Loading configuration...</div>
    <div v-if="error">Error loading configuration: {{ error }}</div>
    <div v-if="!loading && !error && config && config.accounts">
      <div class="tabs">
        <button
          v-for="(account, index) in config.accounts" 
          :key="`user-${index}`" 
          @click="activeTab = `user-${index}`"
          :class="{ active: activeTab === `user-${index}` }"
        >
          {{ account.nickname || `User ${index + 1}`  }}
        </button>
        <button 
          key="explorer"
          @click="activeTab = 'explorer'"
          :class="{ active: activeTab === 'explorer' }"
        >
          Block Explorer
        </button>
      </div>
      <div class="tab-content">
        <div v-if="activeTab && activeTab.startsWith('user-')">
          <UserComponent
            v-if="getUserForTab(activeTab)"
            :key="activeTab" 
            :nickname="getUserForTab(activeTab).nickname"
            :private-key="getUserForTab(activeTab).private_key" 
            :contracts="getUserContractsForActiveTab(activeTab)" 
            :rpc-url="config.rpc_url"
            :global-contracts="config.contracts" 
          />
        </div>
        <div v-if="activeTab === 'explorer'">
          <BlockExplorer 
            :rpc-url="config.rpc_url"
            :global-contracts="config.contracts"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import UserComponent from './components/UserComponent.vue';
import BlockExplorer from './components/BlockExplorer.vue'; // 导入新组件

export default {
  name: 'App',
  components: {
    UserComponent,
    BlockExplorer, // 注册新组件
  },
  data() {
    return {
      config: null,
      loading: true,
      error: null,
      activeTab: null,
    };
  },
  async created() {
    try {
      const response = await fetch('/config.json');
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      this.config = await response.json();
      if (this.config && this.config.accounts && this.config.accounts.length > 0) {
        this.activeTab = 'user-0'; // 默认选中第一个用户
      }
    } catch (e) {
      console.error("Failed to load config.json:", e);
      this.error = e.message;
    } finally {
      this.loading = false;
    }
  },
  methods: {
    getUserForTab(tab) {
      const index = parseInt(tab.split('-')[1], 10);
      return this.config.accounts[index];
    },
    getUserContractsForActiveTab(tab) {
      if (!this.config || !this.config.accounts || tab === null || !tab.startsWith('user-')) {
        return [];
      }
      const index = parseInt(tab.split('-')[1], 10);
      const currentAccountConfig = this.config.accounts[index];

      if (!currentAccountConfig || !currentAccountConfig.contract_functions || !Array.isArray(currentAccountConfig.contract_functions)) {
        console.warn(`User ${index + 1} has no contract_functions defined or it's not an array.`);
        return [];
      }
      // The contract_functions array already has the desired structure for the UserComponent's 'contracts' prop
      return currentAccountConfig.contract_functions.filter(cf => cf && typeof cf.name === 'string' && Array.isArray(cf.functions));
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.tabs button {
  padding: 10px 15px;
  cursor: pointer;
  border: 1px solid #ccc;
  background-color: #f0f0f0;
  margin-right: 5px;
  color: #2c3e50; /* 确保字体颜色不是白色 */
}

.tabs button.active {
  background-color: #fff;
  border-bottom: 1px solid #fff;
  color: #2c3e50; /* 确保选中状态下字体颜色也不是白色 */
}

.tab-content {
  border: 1px solid #ccc;
  padding: 20px;
  margin-top: -1px; /* Align with tab border */
}
</style>
