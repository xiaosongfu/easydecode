<script setup lang="ts">
import {Ref, ref} from "vue";
import {FormatTypes, Interface} from "@ethersproject/abi";
import {keccak256} from "@ethersproject/keccak256";
import {toUtf8Bytes} from "@ethersproject/strings";

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
  <div>
    <div class="abi">
      <h2>Paste your ABI here</h2>
      <textarea v-model="abi" cols="80" rows="12" placeholder='[ { "inputs": [], "name": "Transfer", "type": "event" }, ... ]' :class="{'abi-error': abiError != ''}" @blur="parseABI" />&nbsp;
      <span style="color: red">{{abiError}}</span>
    </div>
    <div class="event">
      <h2>Event</h2>
      Topics: <input v-model="eventTopics" placeholder='0x...Topic1,0x...Topic2,0x...Topic3' />&nbsp;
      Data: <input v-model="eventData" placeholder='0x...Data'/>&nbsp;
      <button @click="decodeEvent" :disabled="abi == '' || abiError != '' || eventTopics == '' || eventData == ''">Decode Event</button>
      <br />
      <pre v-if="eventResult != ''">{{eventResult}}</pre>
    </div>
    <div class="error">
      <h2>Error</h2>
      Hex: <input v-model="error" placeholder="0x..." />&nbsp;
      <button @click="decodeError" :disabled="abi == '' || abiError != '' || error == ''">Decode Error</button>
      <br />
      <pre v-if="errorResult != ''">{{errorResult}}</pre>
    </div>
    <div class="function-data">
      <h2>Function-Data</h2>
      Hex: <input v-model="functionData" placeholder="0x..." />&nbsp;
      <button @click="decodeFunctionData" :disabled="abi == '' || abiError != '' || functionData == ''">Decode Function Data</button>
      <br />
      <pre v-if="functionDataResult != ''">{{functionDataResult}}</pre>
    </div>
    <div class="function-result">
      <h2>Function-Result</h2>
      Hex: <input v-model="functionResult" placeholder="0x..." />&nbsp;
      <button @click="decodeFunctionResult" :disabled="abi == '' || abiError != '' || functionResult == ''">Decode Function Result</button>
      <br />
      <pre v-if="functionResultResult != ''">{{functionResultResult}}</pre>
    </div>
    <div class="star">
      <div><a href="https://github.com/xiaosongfu/easydecode" target="_blank"><img src="https://github.githubassets.com/favicons/favicon-dark.svg" alt="github" width="24xp" height="24px" />&nbsp;Give me a star🌟</a></div>
    </div>
  </div>
</template>

<style scoped>
.abi-error {
  border: 1px solid red;
}
input {
  width: 220px;
  height: 24px;
}
button {
  margin-top: 8px;
}
pre {
  border-radius: 4px;
  border: 1px solid green;
  white-space: pre-wrap;
  word-wrap: break-word;
}
.star {
  margin-top: 18px;
}
</style>
