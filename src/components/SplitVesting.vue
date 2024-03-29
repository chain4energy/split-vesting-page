<script setup lang="ts">
import {ref} from 'vue'
import {AminoTypes, SigningStargateClient} from "@cosmjs/stargate";
import {coins, DirectSecp256k1HdWallet} from "@cosmjs/proto-signing";
import {createKeplrConfig, createVestingAminoConverters, getFees} from "./helpers";
import {MsgSplitVesting} from "../../ts-client/chain4energy.c4echain.cfevesting/types/c4echain/cfevesting/tx";
import {registry} from "../../ts-client";
import {blockchainConfig} from "../blockchainConfig";

const mnemonic = ref("")
const userAddress = ref("")
const amount = ref("")
const confirmed = ref(false)
const toAddress = ref("")

const denomination = 1000000;

let client:SigningStargateClient

const createEncodedMsg = (objectToEncode: any) => {
    return {
        typeUrl: "/chain4energy.c4echain.cfevesting.MsgSplitVesting",
        value: objectToEncode,
    };
}
declare global { interface Window {keplr:any} }

const connectWithKeplr = async () => {
    const chainInfo = createKeplrConfig();
    await window.keplr.experimentalSuggestChain(chainInfo);
    await window.keplr.enable(chainInfo.chainId);
    const offlineSigner = window.keplr.getOfflineSignerOnlyAmino(chainInfo.chainId);
    const accounts = await offlineSigner.getAccounts();
    userAddress.value = accounts[0].address
    const aminoTypes = new AminoTypes(createVestingAminoConverters())
    client = await SigningStargateClient.connectWithSigner(
        blockchainConfig.rpcUrl,
        offlineSigner,
        {registry, aminoTypes}
    );
}

const splitVesting = async() => {
  if (confirm("Confirm transaction")) {
    confirmed.value = true
    try {
      const coinsToSend = new coins(amount.value, "uc4e");
      const msgSplitVestsing: MsgSplitVesting = {
          amount: coinsToSend,
          fromAddress: userAddress.value,
          toAddress: toAddress.value,
      }
      const encoded = createEncodedMsg(msgSplitVestsing)
      const res = await client.signAndBroadcast(userAddress.value, [encoded], getFees(), "ABC=")
      confirmed.value = false
      if (res.code === 0) {
        alert("Transaction successful! Splitted "+ coinsToSend[0].amount / denomination +"C4E tokens.")
      } else {
        alert("Transaction error! Raw error log:" + res.rawLog)
      }
    } catch (e) {
      confirmed.value = false
      alert("Error: " + e.toString())
    }

  }
}

</script>

<template>
  <div id="wrapper">
    <div v-if="!confirmed">
      <div class="wrapper">
          <div v-if="userAddress">
              <h3>User address: {{userAddress}}</h3>
          </div>
        <div v-if="!userAddress" >
            <h2>To split vesting connect with keplr first</h2>
            <button @click="connectWithKeplr()">Connect with keplr</button>
<!--            TODO: add if needed-->
<!--            <hr style="border: 1px solid grey; width: 100%"/>-->
<!--            <h3>Enter mnemonic:</h3>-->
<!--            <textarea v-model="mnemonic"></textarea>-->
<!--            <button @click="retrieveKey(); retrieved = true">Connect with mnemonic</button>-->
        </div>

        <div v-if="userAddress">
          <h3>Split vesting to new vesting acount: </h3>
          <input v-model="toAddress" type="text" />
          <h3>Amount to split (uc4e): </h3>
          <input v-model="amount" type="number" />
          <button @click="splitVesting()">Split vesting</button>
        </div>
      </div>
    </div>

    <div v-if="confirmed">
      <h1 style="font-size:45px; text-align: center; padding: 300px 0; margin: 0">Loading...</h1>
    </div>
  </div>
</template>

<style scoped lang="scss">


#wrapper {
  height: 100vh;
  padding-top: 40px;
  background: orange;
}
textarea {
  min-height: 150px;
  border-radius: 10px;
  border: 1px solid;
  padding: 7px;
  font-size: 15px;
  box-shadow: 3px 4px 4px grey;
}
input {
  border-radius: 10px;
  border: 1px solid;
  padding: 7px;
  font-size: 15px;
  box-shadow: 3px 4px 4px grey;
}
.wrapper {
  display: flex;
  flex-direction: column;
  height: 100%;
  justify-content: center;
  align-content: center;
  align-items: center;

  div {
    box-shadow: 2px 5px 24px 0px rgba(66, 68, 90, 1);
    margin-top: 10px;
    padding: 10px;
    border-radius: 10px;
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 400px;
    background: white;
  }
}

</style>
