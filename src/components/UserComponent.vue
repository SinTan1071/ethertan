<template>
  <div class="user-component">
    <h3>{{ nickname }} ({{ ethAddress }})</h3>
    <div v-for="(contract, index) in processedContracts" :key="index" class="contract-section">
        <h4>Contract: {{ contract.name }} <span v-if="contract.balance !== null"> (Balance: {{ contract.balance }})</span></h4>
      <div v-for="(func, funcIndex) in contract.functions" :key="funcIndex" class="function-section">
        <h5>{{ func.name }}</h5>
        <div class="function-inputs">
          <div v-for="(input, inputIndex) in func.inputs" :key="inputIndex" class="input-group">
            <label :for="`${contract.address}-${func.name}-${input.name}`">{{ input.name }} ({{ input.type }}):</label>
            <input :id="`${contract.address}-${func.name}-${input.name}`" v-model="func.inputValues[input.name]" :placeholder="input.type" />
          </div>
        </div>
        <div class="function-buttons">
          <button v-if="func.stateMutability === 'view' || func.stateMutability === 'pure'" @click="handleQuery(contract, func)">Query</button>
          <button v-else @click="handleTransact(contract, func)">Transact</button>
        </div>
        <p v-if="func.result">Result: {{ func.result }}</p>
        <p v-if="func.error" class="error">Error: {{ func.error }}</p>
      </div>
    </div>
  </div>
</template>

<script>
import Web3 from 'web3';

export default {
  name: 'UserComponent',
  props: {
    nickname: {
      type: String,
      required: true,
    },
    privateKey: {
      type: String,
      required: true,
    },
    contracts: { // This prop now directly receives the filtered contract_functions for the user
      type: Array,
      required: true, // Expected: [{ address: '0x...', functions: ['funcA', 'funcB'] }]
    },
    rpcUrl: {
      type: String,
      required: true,
    },
    globalContracts: { // This contains all contract ABIs from config.json
      type: Array,
      required: true, // Expected: [{ address: '0x...', abi: [...] }]
    }
  },
  data() {
    return {
      web3: new Web3(this.rpcUrl),
      ethAddress: '',
      processedContracts: [],
    };
  },
  watch: {
    // Watch all relevant props that affect processedContracts
    // Using a single watcher for multiple props by returning an array of them
    '$props': {
      deep: true,
      immediate: true, // Process initially when component is created
      handler(newProps) {
        // Specifically call processContracts if 'contracts' or 'globalContracts' props change.
        // The 'contracts' prop is the one derived from user's contract_functions.
        this.processContracts(newProps.contracts); 
      }
    }
  },
  created() {
    try {
    const account = this.web3.eth.accounts.privateKeyToAccount(this.privateKey);
        this.ethAddress = account.address;
      this.web3.eth.accounts.wallet.add(account); // Add account to wallet for transactions
    } catch (e) {
      console.error("Invalid private key:", e);
      this.ethAddress = 'Invalid Private Key';
    }
    this.processContracts(this.contracts);
  },
  methods: {
    async processContracts(userContractsFromProps) { // 注意这里改为 async
      if (!userContractsFromProps || !Array.isArray(userContractsFromProps) || !this.globalContracts || !Array.isArray(this.globalContracts)) {
        this.processedContracts = [];
        console.warn('UserComponent: Invalid props (userContractsFromProps or globalContracts).');
        return;
      }

      const processed = await Promise.all(userContractsFromProps // 使用 Promise.all 来处理异步操作
        .filter(userContractDef => userContractDef && typeof userContractDef.name === 'string' && Array.isArray(userContractDef.functions))
        .map(async userContractDef => { // map 回调也改为 async
          const globalContractAbiInfo = this.globalContracts.find(gc => gc && typeof gc.name === 'string' && gc.name.toLowerCase() === userContractDef.name.toLowerCase());
          
          if (!globalContractAbiInfo || !globalContractAbiInfo.abi) {
            console.warn(`UserComponent: ABI not found in globalContracts for address: ${userContractDef.address}`);
            return { address: userContractDef.address, functions: [], name: 'Unknown Contract', type: null, balance: null, error: `ABI not found for ${userContractDef.address}` };
          }
        
          userContractDef.address = globalContractAbiInfo.address;

          // Instantiate the contract using the global ABI inf
          let contractInstance;
          try {
            contractInstance = new this.web3.eth.Contract(globalContractAbiInfo.abi, userContractDef.address);
          } catch (e) {
            console.error(`UserComponent: Failed to instantiate contract ${userContractDef.address}:`, e);
            return { address: userContractDef.address, functions: [], name: globalContractAbiInfo.name || userContractDef.address, type: globalContractAbiInfo.type, balance: null, error: `Failed to instantiate contract: ${e.message}` };
          }

          const functions = userContractDef.functions
            .filter(funcName => typeof funcName === 'string')
            .map(funcName => {
              const abiEntry = globalContractAbiInfo.abi.find(entry => entry && entry.type === 'function' && entry.name === funcName);
              if (!abiEntry) {
                console.warn(`UserComponent: Function ${funcName} not found in ABI for contract ${userContractDef.address}`);
                return null;
              }
              return {
                name: funcName,
                inputs: abiEntry.inputs || [],
                inputValues: (abiEntry.inputs || []).reduce((acc, input) => {
                  if (input && typeof input.name === 'string') {
                    acc[input.name] = '';
                  }
                  return acc;
                }, {}),
                stateMutability: abiEntry.stateMutability,
                abi: abiEntry,
                result: null,
                error: null,
                contractInstance: contractInstance
              };
            }).filter(f => f !== null);
          
          let balance = null;
          if (globalContractAbiInfo.type === 'ERC20' && this.ethAddress) {
            const balanceOfAbi = globalContractAbiInfo.abi.find(entry => entry.name === 'balanceOf' && entry.type === 'function');
            if (balanceOfAbi && contractInstance) {
              try {
                const rawBalance = await contractInstance.methods.balanceOf(this.ethAddress).call();
                // 假设 balanceOf 返回的是一个数字或可以被 toString 的值，对于 ERC20 通常是 BigInt
                balance = this.stringifyBigInts(rawBalance); // 使用已有的辅助函数处理 BigInt
              } catch (e) {
                console.error(`UserComponent: Error fetching balance for ${globalContractAbiInfo.name || userContractDef.address}:`, e);
                balance = 'Error';
              }
            }
          }
          
          return { address: userContractDef.address, functions, name: globalContractAbiInfo.name || userContractDef.address, type: globalContractAbiInfo.type, balance: balance };
      }));
      this.processedContracts = processed.filter(p => p !== null); // 过滤掉处理失败的合约（如果有的话）
    },
    stringifyBigInts(obj) { // Helper function to convert BigInts in an object/array to strings
      if (typeof obj === 'bigint') {
        return obj.toString();
      }
      if (Array.isArray(obj)) {
        return obj.map(this.stringifyBigInts); // Use this.stringifyBigInts for recursive calls if it's a method
      }
      if (typeof obj === 'object' && obj !== null) {
        const newObj = {};
        for (const key in obj) {
          if (Object.prototype.hasOwnProperty.call(obj, key)) {
            newObj[key] = this.stringifyBigInts(obj[key]); // Use this.stringifyBigInts
          }
        }
        return newObj;
      }
      return obj;
    },
    async updateBalance(contractObject) {
      if (contractObject && contractObject.type === 'ERC20' && this.ethAddress) {
        // Find the ABI for balanceOf and the contract instance
        // We can assume contractObject.functions[0].contractInstance exists if functions are populated
        // Or, more robustly, find the contract instance associated with this contractObject
        // For simplicity, let's assume the instance is readily available or can be derived.
        // If contractInstance was stored directly on contractObject during processContracts, that would be easier.
        // Let's try to get it from the first function, assuming it's consistent for the contract.
        const globalContractAbiInfo = this.globalContracts.find(gc => gc && typeof gc.address === 'string' && gc.address.toLowerCase() === contractObject.address.toLowerCase());

        if (globalContractAbiInfo && globalContractAbiInfo.abi) {
          const contractInstance = new this.web3.eth.Contract(globalContractAbiInfo.abi, contractObject.address);
          const balanceOfAbi = globalContractAbiInfo.abi.find(entry => entry.name === 'balanceOf' && entry.type === 'function');
          if (balanceOfAbi) {
            try {
              const rawBalance = await contractInstance.methods.balanceOf(this.ethAddress).call();
              contractObject.balance = this.stringifyBigInts(rawBalance);
              console.log(`Balance updated for ${contractObject.name}: ${contractObject.balance}`);
            } catch (e) {
              console.error(`Error updating balance for ${contractObject.name}:`, e);
              contractObject.balance = 'Error fetching balance';
            }
          }
        } else {
          console.warn(`Could not update balance for ${contractObject.name}: instance or ABI info missing.`);
        }
      }
    },
    async handleQuery(contract, func) {
      func.error = null;
      func.result = null;
      try {
        const params = func.inputs.map(input => func.inputValues[input.name]);
        const result = await func.contractInstance.methods[func.name](...params).call({ from: this.ethAddress });
        // Convert BigInts to strings before stringifying
        const resultStringifiable = this.stringifyBigInts(result);
        func.result = JSON.stringify(resultStringifiable);
      } catch (e) {
        console.error(`Error querying ${func.name}:`, e);
        func.error = e.message;
      }
    },
    async handleTransact(contract, func) { // 'contract' here is the specific contract the transaction was for
      func.error = null;
      func.result = null;
      try {
        const params = func.inputs.map(input => func.inputValues[input.name]);
        const gasPrice = await this.web3.eth.getGasPrice();
        const gasEstimate = await func.contractInstance.methods[func.name](...params).estimateGas({ from: this.ethAddress });

        const tx = {
          from: this.ethAddress,
          to: contract.address, // Use the address of the contract the transaction is for
          gas: gasEstimate,
          gasPrice: gasPrice,
          data: func.contractInstance.methods[func.name](...params).encodeABI(),
        };

        const signedTx = await this.web3.eth.accounts.signTransaction(tx, this.privateKey);
        const receipt = await this.web3.eth.sendSignedTransaction(signedTx.rawTransaction);
        func.result = `Transaction successful: ${receipt.transactionHash}`;

        // After successful transaction, update balances for ALL ERC20 contracts for this user
        for (const contractToUpdate of this.processedContracts) {
          if (contractToUpdate.type === 'ERC20') {
            await this.updateBalance(contractToUpdate);
          }
        }

      } catch (e) {
        console.error(`Error transacting ${func.name}:`, e);
        func.error = e.message;
      }
    },
  },
};
</script>

<style>
.user-component {
  text-align: left; /* 确保 UserComponent 内部内容左对齐 */
}

.contract-section {
  border: 1px solid #eee;
  margin-bottom: 20px;
  padding: 10px; /* 调整或移除 padding-left */
  border-left: none; /* 移除左侧边框 */
}

.contract-section h4 {
  margin-left: 0; /* 确保 h4 左对齐 */
}

.function-section {
  border: 1px solid #f9f9f9;
  margin-bottom: 10px;
  padding: 10px; /* 调整或移除 padding-left */
  border-left: none; /* 移除左侧边框 */
}

.function-section h5 {
  margin-left: 0; /* 确保 h5 左对齐 */
}

.input-group {
  margin-bottom: 10px;
  display: flex; /* 使用 flexbox */
  align-items: center; /* 垂直居中对齐 */
}

.input-group label {
  margin-right: 5px; /* 在 label 和 input 之间添加一些间距 */
  margin-bottom: 0; /* 移除底部边距 */
}

.input-group input[type="text"] {
  flex-grow: 1; /* input 占据剩余空间 */
  padding: 8px;
  box-sizing: border-box;
}

.function-buttons {
  text-align: right; /* 按钮右对齐 */
  margin-top: 10px;
}

button {
  padding: 8px 15px;
  background-color: #007bff;
  color: white;
  border: none;
  cursor: pointer;
  margin-left: 10px; /* 按钮之间留有间距 */
  margin-top: 5px;
  /* margin-left: 0; */ /* 移除此行，由父容器控制对齐 */
}

button:hover {
  background-color: #0056b3;
}

p {
  margin-top: 10px;
}

.error {
  color: red;
  margin-top: 10px;
  margin-left: 0; /* 确保错误信息左对齐 */
}
</style>