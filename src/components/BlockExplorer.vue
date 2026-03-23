<template>
  <div class="block-explorer">
    <h3>Block Explorer</h3>
    <div class="input-group">
      <label for="block-number">Block Number:</label>
      <input id="block-number" v-model="blockNumber" placeholder="Enter block number" />
      <button @click="getBlockDetails">Query</button>
    </div>
    <div v-if="loading">Loading block details...</div>
    <div v-if="error" class="error">Error: {{ error }}</div>
    <div v-if="blockDetails" class="block-details">
      <h4>Block #{{ blockDetails.number }}</h4>
      <p><strong>Hash:</strong> {{ blockDetails.hash }}</p>
      <p><strong>Timestamp:</strong> {{ new Date(blockDetails.timestamp * 1000).toLocaleString() }}</p>
      <p><strong>Transactions:</strong> {{ blockDetails.transactions.length }}</p>
      <div v-if="decodedTransactions.length > 0">
        <h5>Decoded Transactions:</h5>
        <ul>
          <li v-for="(tx, index) in decodedTransactions" :key="index">
            <p><strong>Contract:</strong> {{ tx.contractName }}</p>
            <p><strong>Function:</strong> {{ tx.functionName }}({{ tx.params.join(', ') }})</p>
            <p><strong>From:</strong> {{ tx.from }}</p>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import Web3 from 'web3';

export default {
  name: 'BlockExplorer',
  props: {
    rpcUrl: {
      type: String,
      required: true,
    },
    globalContracts: {
      type: Array,
      required: true,
    },
  },
  data() {
    return {
      web3: new Web3(this.rpcUrl),
      blockNumber: '',
      blockDetails: null,
      decodedTransactions: [],
      loading: false,
      error: null,
      abiDecoder: null,
    };
  },
  created() {
    // Create a map of contract addresses to ABIs for quick lookup
    this.contractAbiMap = this.globalContracts.reduce((map, contract) => {
      if (contract.address && contract.abi) {
        map[contract.address.toLowerCase()] = { abi: contract.abi, name: contract.name };
      }
      return map;
    }, {});
  },
  methods: {
    async getBlockDetails() {
      if (!this.blockNumber) {
        this.error = 'Please enter a block number.';
        return;
      }
      this.loading = true;
      this.error = null;
      this.blockDetails = null;
      this.decodedTransactions = [];

      try {
        const block = await this.web3.eth.getBlock(this.blockNumber, true); // true to get transaction objects
        if (!block) {
          throw new Error(`Block ${this.blockNumber} not found.`);
        }
        this.blockDetails = block;
        this.decodeTransactions(block.transactions);
      } catch (e) {
        console.error('Error fetching block details:', e);
        this.error = e.message;
      } finally {
        this.loading = false;
      }
    },
    decodeTransactions(transactions) {
      const decoded = [];
      const web3 = new Web3(); // Local instance for decoding

      for (const tx of transactions) {
        const toAddress = tx.to ? tx.to.toLowerCase() : null;
        if (toAddress && this.contractAbiMap[toAddress]) {
          const { abi, name } = this.contractAbiMap[toAddress];
          const contract = new web3.eth.Contract(abi);
          console.log(contract)
          
          // Find the function signature from the input data
          const methodSignature = tx.input.slice(0, 10);
          const abiItem = abi.find(item => item.type === 'function' && web3.eth.abi.encodeFunctionSignature(item) === methodSignature);

          if (abiItem) {
            try {
              const decodedParams = web3.eth.abi.decodeParameters(abiItem.inputs, tx.input.slice(10));
              const params = abiItem.inputs.map(input => decodedParams[input.name]);
              decoded.push({
                contractName: name,
                functionName: abiItem.name,
                params: params.map(p => p.toString()), // Convert values to string for display
                from: tx.from,
              });
            } catch (e) {
              console.warn(`Could not decode transaction for ${name}:`, e);
            }
          }
        }
      }
      this.decodedTransactions = decoded;
    },
  },
};
</script>

<style scoped>
.block-explorer {
  text-align: left;
}
.input-group {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
}
.input-group input {
  margin: 0 10px;
  flex-grow: 1;
}
.block-details {
  margin-top: 20px;
  border-top: 1px solid #eee;
  padding-top: 20px;
}
.error {
  color: red;
}
</style>