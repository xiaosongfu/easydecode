<script setup lang="ts">
import {Ref, ref} from "vue";
import {FormatTypes, Interface} from "@ethersproject/abi";
import {keccak256} from "@ethersproject/keccak256";
import {toUtf8Bytes} from "@ethersproject/strings";
import logo from "./assets/logo.svg";

// ------ ------ ------ ------ ------ ------ ------ ------ ------

const abi: Ref<string> = ref('');

const abiError: Ref<string> = ref('');

function parseABI(): Interface {
  try{
    abiError.value = '';
    return new Interface(abi.value);
  } catch (e) {
    abiError.value = 'Invalid ABI';
  }
  return new Interface([]);
}

// ------ ------ ------ ------ ------ ------ ------ ------ ------

const eventTopics: Ref<string> = ref('');
const eventData: Ref<string> = ref('');

const eventResult: Ref<string> = ref('');

async function decodeEvent() {
  const i = parseABI();
  const items = i.format(FormatTypes.full);
  for (const item of items) {
    // console.log(item);
    if (item.startsWith("event ")) {
      const eventSignature = item.replace("event ", ""); // 'event Borrow(uint256 indexed,address indexed,address indexed,uint256,uint256,uint256,uint8,uint256,uint256)'
      const eventHash = i.getEventTopic(i.getEvent(eventSignature));
      // console.log(eventSignature, " |---| ", eventHash);

      const topics = eventTopics.value.split(",");
      if (topics[0] == eventHash) {
        const result = i.decodeEventLog(
            i.getEvent(eventSignature)!!,
            eventData.value,
            topics,
        );

        console.log(eventSignature);
        for (const r of result.values()) {
          console.log("  >>> ", r.toString());
        }
        eventResult.value = `Event: ${eventSignature} \n Params: (${result.join(', ')})`

        break;
      }
    }
  }
}

// ------ ------ ------ ------ ------ ------ ------ ------ ------

const error: Ref<string> = ref('');

const errorResult: Ref<string> = ref('');

async function decodeError() {
  const selector = error.value.slice(0, 10);

  const i = parseABI();
  const items = i.format(FormatTypes.minimal);
  for (const item of items) {
    // console.log(item);
    if (item.startsWith("error ")) {
      const errorSignature = item.replace("error ", ""); // 'error InsufficientBalance(account,uint256)'
      const errorSelector = keccak256(toUtf8Bytes(errorSignature)).slice(
          0,
          10,
      ); // '0x' + 4 bytes(8 hex chars)
      // console.log(errorSignature, ' |---| ', errorSelector);

      if (selector == errorSelector) {
        const result = i.decodeErrorResult(
            i.getError(errorSignature)!!,
            error.value,
        );

        console.log(errorSignature);
        for (const r of result.values()) {
          console.log("  >>> ", r.toString());
        }
        errorResult.value = `Error: ${errorSignature} \n Params: (${result.join(', ')})`

        break;
      }
    }
  }}

// ------ ------ ------ ------ ------ ------ ------ ------ ------

const functionData: Ref<string> = ref('');
const functionResult: Ref<string> = ref('');

const functionDataResult: Ref<string> = ref('');
const functionResultResult: Ref<string> = ref('');

async function decodeFunctionData() {
  const selector = functionData.value.slice(0, 10);

  const i = parseABI();
  const items = i.format(FormatTypes.minimal);
  for (const item of items) {
    // console.log(item);
    if (item.startsWith("function ")) {
      const functionSignature = item.replace("function ", ""); // 'function borrow(uint256)'
      const functionSelector = keccak256(
          toUtf8Bytes(functionSignature),
      ).slice(0, 10); // '0x' + 4 bytes(8 hex chars)
      // console.log(functionSignature, ' |---| ', functionSelector);

      if (selector == functionSelector) {
        const result = i.decodeFunctionData(
            i.getFunction(functionSignature)!!,
            functionData.value,
        );

        console.log(functionSignature);
        for (const r of result.values()) {
          console.log("  >>> ", r.toString());
        }
        functionDataResult.value = `Function: ${functionSignature} \n Params: (${result.join(', ')})`;

        break;
      }
    }
  }
}

async function decodeFunctionResult() {
  const selector = functionResult.value.slice(0, 10);

  // TODO 未测试

  const i = parseABI();
  const items = i.format(FormatTypes.minimal);
  for (const item of items) {
    // console.log(item);
    if (item.startsWith("function ")) {
      const functionSignature = item.replace("function ", ""); // 'function borrowInterestRate() view returns (uint256)'
      const functionSelector = keccak256(
          toUtf8Bytes(functionSignature),
      ).slice(0, 10); // '0x' + 4 bytes(8 hex chars)
      // console.log(functionSignature, ' |---| ', functionSelector);

      if (selector == functionSelector) {
        const result = i.decodeFunctionResult(
            i.getFunction(functionSignature)!!,
            functionResult.value,
        );

        console.log(functionSignature);
        for (const r of result.values()) {
          console.log("  >>> ", r.toString());
        }
        functionResultResult.value = `Function: ${functionSignature} \n Result: (${result.join(', ')})`;

        break;
      }
    }
  }
}
</script>

<template>
  <div class="app-container">
    <header class="app-header">
      <h1 class="app-title">Easy Decode</h1>
      <p class="app-subtitle">Easily decode errors, events, and function data from EVM-compatible smart contracts</p>
    </header>

    <main class="main-content">
      <!-- ABI Section -->
      <section class="card abi-section">
        <div class="card-header">
          <h2 class="section-title">ABI</h2>
          <p class="section-description">Paste your ABI JSON here</p>
        </div>
        <div class="card-body">
          <textarea 
            v-model="abi" 
            class="textarea-input"
            :class="{'input-error': abiError != ''}" 
            placeholder='[ { "inputs": [], "name": "Transfer", "type": "event" }, ... ]' 
            @blur="parseABI"
          />
          <div v-if="abiError != ''" class="error-message">{{abiError}}</div>
        </div>
      </section>

      <!-- Event Section -->
      <section class="card decode-section">
        <div class="card-header">
          <h2 class="section-title">Event Decoder</h2>
          <p class="section-description">Decode event logs from transaction receipts</p>
        </div>
        <div class="card-body">
          <div class="form-group">
            <label class="form-label">Topics</label>
            <input 
              v-model="eventTopics" 
              class="text-input"
              placeholder='0x...Topic1,0x...Topic2,0x...Topic3' 
            />
          </div>
          <div class="form-group">
            <label class="form-label">Data</label>
            <input 
              v-model="eventData" 
              class="text-input"
              placeholder='0x...Data'
            />
          </div>
          <button 
            class="btn btn-primary"
            @click="decodeEvent" 
            :disabled="abi == '' || abiError != '' || eventTopics == '' || eventData == ''"
          >
            Decode Event
          </button>
          <div v-if="eventResult != ''" class="result-box">
            <pre class="result-text">{{eventResult}}</pre>
          </div>
        </div>
      </section>

      <!-- Error Section -->
      <section class="card decode-section">
        <div class="card-header">
          <h2 class="section-title">Error Decoder</h2>
          <p class="section-description">Decode custom error messages</p>
        </div>
        <div class="card-body">
          <div class="form-group">
            <label class="form-label">Hex Data</label>
            <input 
              v-model="error" 
              class="text-input"
              placeholder="0x..." 
            />
          </div>
          <button 
            class="btn btn-primary"
            @click="decodeError" 
            :disabled="abi == '' || abiError != '' || error == ''"
          >
            Decode Error
          </button>
          <div v-if="errorResult != ''" class="result-box">
            <pre class="result-text">{{errorResult}}</pre>
          </div>
        </div>
      </section>

      <!-- Function Data Section -->
      <section class="card decode-section">
        <div class="card-header">
          <h2 class="section-title">Function Data Decoder</h2>
          <p class="section-description">Decode function call data</p>
        </div>
        <div class="card-body">
          <div class="form-group">
            <label class="form-label">Hex Data</label>
            <input 
              v-model="functionData" 
              class="text-input"
              placeholder="0x..." 
            />
          </div>
          <button 
            class="btn btn-primary"
            @click="decodeFunctionData" 
            :disabled="abi == '' || abiError != '' || functionData == ''"
          >
            Decode Function Data
          </button>
          <div v-if="functionDataResult != ''" class="result-box">
            <pre class="result-text">{{functionDataResult}}</pre>
          </div>
        </div>
      </section>

      <!-- Function Result Section -->
      <section class="card decode-section">
        <div class="card-header">
          <h2 class="section-title">Function Result Decoder</h2>
          <p class="section-description">Decode function return values</p>
        </div>
        <div class="card-body">
          <div class="form-group">
            <label class="form-label">Hex Data</label>
            <input 
              v-model="functionResult" 
              class="text-input"
              placeholder="0x..." 
            />
          </div>
          <button 
            class="btn btn-primary"
            @click="decodeFunctionResult" 
            :disabled="abi == '' || abiError != '' || functionResult == ''"
          >
            Decode Function Result
          </button>
          <div v-if="functionResultResult != ''" class="result-box">
            <pre class="result-text">{{functionResultResult}}</pre>
          </div>
        </div>
      </section>
    </main>

    <footer class="app-footer">
      <a 
        href="https://github.com/xiaosongfu/easydecode" 
        target="_blank"
        class="github-link"
      >
        <img 
          src="https://github.githubassets.com/favicons/favicon-dark.svg" 
          alt="github" 
          class="github-icon"
        />
        <span>Give me a star 🌟</span>
      </a>
    </footer>
  </div>
</template>

<style scoped>
.app-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 2rem 1rem;
}

.app-header {
  text-align: center;
  margin-bottom: 3rem;
  color: white;
}

@keyframes fadeInDown {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.app-title {
  font-size: 3rem;
  font-weight: 700;
  margin: 0 0 0.5rem 0;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.app-subtitle {
  font-size: 1.1rem;
  opacity: 0.95;
  margin: 0;
  font-weight: 300;
}

.main-content {
  max-width: 900px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.card {
  background: white;
  border-radius: 16px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 15px 50px rgba(0, 0, 0, 0.15);
}

.card-header {
  padding: 1.5rem 2rem;
  background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
  border-bottom: 1px solid #e0e0e0;
}

.section-title {
  font-size: 1.5rem;
  font-weight: 600;
  margin: 0 0 0.25rem 0;
  color: #2d3748;
}

.section-description {
  font-size: 0.9rem;
  color: #718096;
  margin: 0;
}

.card-body {
  padding: 2rem;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-label {
  display: block;
  font-size: 0.9rem;
  font-weight: 500;
  color: #4a5568;
  margin-bottom: 0.5rem;
}

.text-input {
  width: 100%;
  padding: 0.75rem 1rem;
  font-size: 0.95rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  transition: all 0.2s ease;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  background: #f7fafc;
}

.text-input:focus {
  outline: none;
  border-color: #667eea;
  background: white;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.textarea-input {
  width: 100%;
  min-height: 200px;
  padding: 1rem;
  font-size: 0.9rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  transition: all 0.2s ease;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  background: #f7fafc;
  resize: vertical;
  line-height: 1.6;
}

.textarea-input:focus {
  outline: none;
  border-color: #667eea;
  background: white;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.input-error {
  border-color: #fc8181 !important;
  background: #fff5f5 !important;
}

.error-message {
  margin-top: 0.75rem;
  color: #e53e3e;
  font-size: 0.9rem;
  font-weight: 500;
}

.btn {
  padding: 0.75rem 2rem;
  font-size: 1rem;
  font-weight: 600;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-family: inherit;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
}

.btn-primary:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.5);
}

.btn-primary:active:not(:disabled) {
  transform: translateY(0);
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.result-box {
  margin-top: 1.5rem;
  padding: 1.25rem;
  background: #f0f9ff;
  border: 2px solid #0ea5e9;
  border-radius: 8px;
}

.result-text {
  margin: 0;
  font-size: 0.9rem;
  color: #0c4a6e;
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  line-height: 1.6;
}

.app-footer {
  text-align: center;
  margin-top: 3rem;
  padding-top: 2rem;
}

.github-link {
  display: inline-flex;
  align-items: center;
  gap: 0.75rem;
  color: #2d3748;
  text-decoration: none;
  font-size: 1rem;
  font-weight: 600;
  padding: 1rem 2rem;
  background: white;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.github-link::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(102, 126, 234, 0.1), transparent);
  transition: left 0.5s ease;
}

.github-link:hover::before {
  left: 100%;
}

.github-link:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.25);
  border-color: rgba(102, 126, 234, 0.5);
  background: #f8f9ff;
}

.github-link:active {
  transform: translateY(-1px);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.github-icon {
  width: 22px;
  height: 22px;
  transition: transform 0.3s ease;
  filter: brightness(0) saturate(100%);
}

.github-link:hover .github-icon {
  transform: rotate(360deg) scale(1.1);
}

@media (max-width: 768px) {
  .app-container {
    padding: 1.5rem 1rem;
  }

  .app-logo {
    width: 80px;
    height: 80px;
  }

  .app-title {
    font-size: 2rem;
  }

  .app-subtitle {
    font-size: 0.95rem;
  }

  .card-header {
    padding: 1.25rem 1.5rem;
  }

  .card-body {
    padding: 1.5rem;
  }

  .section-title {
    font-size: 1.25rem;
  }

  .github-link {
    padding: 0.875rem 1.5rem;
    font-size: 0.95rem;
  }

  .github-icon {
    width: 20px;
    height: 20px;
  }
}
</style>
