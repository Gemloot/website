<script setup>
import {onMounted, ref, computed} from "@vue/runtime-core";
import {ethers} from "ethers";
import { useMessage } from 'naive-ui'
import VueCountdown from '@chenfengyuan/vue-countdown';
import {range} from "lodash";
import {CONTRACT_CONFIG} from "../constants";
import { LogoTwitter, LogoDiscord } from "@vicons/ionicons5";
import GEM_ABI from "../abis/gem.abi.json";
import {addressFilter} from "../utils/address";
import Gem from '../components/gem/index.vue'

const address = ref('')
const error = ref(false)
const claimed = ref(false)
const loading = ref({
  account: false,
  gem: false,
  claim: false,
  counting: false,
})
const nftContract = ref()
const loots = ref([])
const counting = ref(0)
const message = useMessage()

const provider = window.web3 ? new ethers.providers.Web3Provider(window.web3.currentProvider) :  new ethers.providers.JsonRpcProvider('https://mainnet.infura.io/v3/9aa3d95b3bc440fa88ea12eaa4456161');

const setAddress = async () => {
  loading.value.account = true
  loading.value.gem = true
  try {
    const signer = await provider.getSigner()
    const value = await signer.getAddress()
    nftContract.value = new ethers.Contract(CONTRACT_CONFIG.GLOOT, GEM_ABI, signer)
    address.value = value
    loading.value.account = false
    await countDown()
    claimed.value = (await nftContract.value._claimTimes(value)).toNumber() >= 1
    loading.value.gem = false
  } catch (e) {
    loading.value.account = false
    loading.value.gem = false
  }
}

const countDown = async () => {
  const now = Date.now()
  loading.value.counting = true
  const nftContract = new ethers.Contract(CONTRACT_CONFIG.GLOOT, GEM_ABI, provider)
  const time = (await nftContract.startTime()).toNumber() * 1e3
  const startDate = time
  if(now < time) {
    counting.value = startDate - now
  } else {
    counting.value = 0
  }
  loading.value.false = true
}

onMounted(() => {
  const load = async () => {
    await setAddress()
    if(address.value) {
      await loadLoot()
    }
  }

  if(window.web3) {
    const web3Provider = window.web3.currentProvider

    web3Provider.on('connect', () => {
      load()
    })

    web3Provider.on('disconnect', () => {
      address.value = ''
      claimed.value = false
      loots.value = []
    })

    web3Provider.on('accountsChanged', () => {
      load()
    })

    web3Provider.on("networkChanged", async (id) => {
      checkNetwork(id)
    });
  }

  checkNetwork()
  load()
})

const connect = async () => {
  await provider.send("eth_requestAccounts", []);
}

const claim = async () => {
  const signer = await provider.getSigner()
  loading.value.claim = true
  try {
    const tx = await nftContract.value.claim();
    const recept = await tx.wait()
    claimed.value = true
    loading.value.claim = false
    message.success('Claim Success!')
    await loadLoot()
  }catch (e) {
    loading.value.claim = false
    message.warning(e.message || e.toString())
  }
}

const checkNetwork = async (id) => {
  console.log(id);
  const chainId = id || (await provider.getNetwork()).chainId;
  console.log("chainId", chainId);
  if (chainId != 1) {
    error.value = true
    message.error('Please switch to mainnet')
  } else {
    error.value = false
  }
}

const loadLoot = async  () => {
  loading.value.gem = true
  const value = address.value;
  const nftContract = new ethers.Contract(CONTRACT_CONFIG.GLOOT, GEM_ABI, provider)
  const count = (await nftContract.balanceOf(value)).toNumber()
  const list = (await Promise.all(range(count).map(async (index) => {
    console.log(index, address);
    const id = (await nftContract.tokenOfOwnerByIndex(value, index)).toNumber()
    const uri = await nftContract.tokenURI(id)
    const json = JSON.parse(window.atob(uri.replace('data:application/json;base64,', '')))
    return {
      id,
      json
    }
  })));
  loots.value = list
  loading.value.gem = false
}

const onCountdownEnd = () => {
  counting.value = 0
}

const displayAddress = computed(() => {
  return addressFilter(address.value)
})

</script>

<template>
  <div class="container">
    <div class="header">
      <div class="logo">
        <div class="img"></div>
        Gemloot
      </div>

      <div class="wallet">
        <n-button v-if="error" type="error" size="large">
          Error Network
        </n-button>
        <template v-else>
          <n-button v-if="!address" ghost size="large" color="#fff" @click="connect">
            connect
          </n-button>
          <n-button v-else ghost color="#fff" size="large" :loading="loading.account">
            {{displayAddress}}
          </n-button>
        </template>
      </div>
    </div>

    <div v-if="loading.gem" class="loading">
      <n-spin stroke="#fff"></n-spin>
    </div>

    <template v-else>


      <div class="info">
        <div class="details">
          <p>
            Nasty monsters and devils ravaging, they non-stopped swarming from the dungeons and the situation has only become worse.
          </p>
          <p>
            Heroes, stand out and fight for our people, retake our lands. To start your adventure for salvation, you can claim this loot of gemstones with magic power from our ancient heroes legacy.
          </p>
          <p>You may manufacture them into jewellery, trade them or set them on your equipment for enhancement. These precious gemstones are rare and valuable, so use them wisely. Adventurers, claim your loot and get ready for your journey.</p>
        </div>
        <img class="chest" v-if="!claimed" src="../assets/chest-c.svg"/>
        <img class="chest" v-if="claimed" src="../assets/chest.svg"/>
      </div>

      <div v-if="!!address">

      <vue-countdown :time="counting" v-if="counting" @end="onCountdownEnd" v-slot="{ days, hours, minutes, seconds }">
        <div class="ts-count-down">
          <div><span class="ts-cc-number">{{days}}</span><span class="ts-cc-description">Days</span></div>
          <div><span class="ts-cc-number">{{hours}}</span><span class="ts-cc-description">Hours</span></div>
          <div><span class="ts-cc-number">{{minutes}}</span><span class="ts-cc-description">Minutes</span></div>
          <div><span class="ts-cc-number">{{seconds}}</span><span class="ts-cc-description">Seconds</span></div>
        </div>
      </vue-countdown>

        <div class="block-claim">
          <n-button class="btn-claim" size="large" color="#9b8e3c"
                    @click="claim"
                    :loading="loading.claim"
                    :disabled="claimed || !!counting"
          >
            {{ claimed ? 'Claimed' : 'Claim' }}
          </n-button>
        </div>
        <h3></h3>
        <div :class="['block-gems', {'block-gems-multi': loots.length >= 2} ]">
          <gem v-for="item in loots" :item="item" key="item.id" class="item-gem">
            <img :src="item.json.image" class="gem">
            <p>{{item.json.name}}</p>
          </gem>
        </div>
      </div>

      <div v-else class="block-claim">
        <n-button class="btn-claim" size="large" color="#9b8e3c" @click="connect">
          connect
        </n-button>
      </div>

    </template>

    <div class="footer">
      <div class="actions">
        <a href="">
          <n-icon size="20">
            <logo-twitter></logo-twitter>
          </n-icon>
        </a>
      </div>
      <p>Gemloot is randomized gemstones generated and stored on chain for adventure</p>
    </div>
  </div>
</template>

<style lang="less" scoped src="./home.less">
</style>
