<template>
  <div class="text-white text-center f">
    <h1 class="text-xl">Beluga Dashboard</h1>
    <span class="text-base">A simple dashboard for Beluga's stats.</span>
    <h2 class="text-lg">Beluga Treasury</h2>
    <span class="text-base" v-if="treasuryBalanceInUsd == 0">Loading total value...</span>
    <span class="text-base" v-else>Total Value: ${{ treasuryBalanceInUsd }}</span>
    <ul>
      <li class="text-base" v-for="balance in treasuryBalanceSheet">{{ balance.symbol }}: ${{ balance.balance }}</li>
    </ul>
    <span>Last updated at block: {{ currentBlock == 0 ? "fetching..." : currentBlock }}</span>
  </div>
</template>

<style lang="css">
  @font-face {
    font-family: 'Lato';
    font-style: normal;
    font-weight: 400;
    font-display: swap;
    src: url('/Lato-Regular.ttf') format('truetype');
  }

  body {
    background: #222432; /*linear-gradient(116.51deg, #0b0516 70.51%, #9F7FD8 99.55%) fixed;*/
    background-image: url("https://www.transparenttextures.com/patterns/black-twill.png"); /* carbon-fibre-v2, 60-lines, black-twill, az-subtle */
    font-family: 'Lato';
  }
</style>

<script lang="ts">
import Vue from 'vue'
import constants from '@/constants'
import ERC20ABI from '@/constants/ERC20.json'
import OracleABI from '@/constants/Oracle.json'
import { JsonRpcProvider } from '@ethersproject/providers'
import { Provider, Contract } from 'ethcall'

export default Vue.extend({
  name: 'IndexPage',
  data() {
    return {
      provider: new JsonRpcProvider(constants.targetRpc),
      currentBlock: 0,
      treasuryBalanceInUsd: 0,
      treasuryBalanceSheet: []
    }
  },
  methods: {
    async fetchBalanceSheet() {
      // Load providers.
      let multicallProvider = new Provider()
      await multicallProvider.init(this.provider)

      // Load contracts for multicall fetching.
      let tokenContracts: Contract[] = []
      for(let i = 0; i < constants.protocolTokens.length; i++) {
        tokenContracts.push(new Contract(constants.protocolTokens[i].address, ERC20ABI))
      }
      let oracle: Contract = new Contract(constants.priceOracle, OracleABI)
      
      // Fetch all balances and prices
      let balanceCalls = []
      let priceCalls = []
      for(let i = 0; i < tokenContracts.length; i++) {
        balanceCalls.push(tokenContracts[i].balanceOf(constants.governance))
        priceCalls.push(oracle.calculateAssetPrice(tokenContracts[i].address))
      }
      let balances: any[] = await multicallProvider.all(balanceCalls)
      let prices: any[] = await multicallProvider.all(priceCalls)

      // Calculate balance sheet.
      let balInUsd = 0
      let balSheet = []

      let ethBalance: any = await multicallProvider.all([multicallProvider.getEthBalance(constants.governance)])
      balInUsd += (Number(ethBalance[0]) / 1e18) * (prices[0] / 1e18)
      // @ts-ignore
      balSheet.push({ symbol: "FTM", balance: (ethBalance[0] / 1e18) })
      for(let i = 0; i < balances.length; i++) {
        let balance = (balances[i] / 1e18) * (prices[i] / 1e18)
        balInUsd += balance
        // @ts-ignore
        balSheet.push({symbol: constants.protocolTokens[i].symbol, balance: balance})
      }

      // Reload balance sheet
      this.treasuryBalanceInUsd = balInUsd
      // @ts-ignore
      this.treasuryBalanceSheet = balSheet
    },
    async updateCurrentBlock() {
      setInterval(async () => this.currentBlock = await this.provider.getBlockNumber(), 5000)
    }
  },
  mounted: function () {
    this.fetchBalanceSheet()
    this.updateCurrentBlock()
  },
  watch: {
    currentBlock() {
      this.fetchBalanceSheet()
    }
  }
})
</script>