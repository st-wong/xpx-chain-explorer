<template>
  <div class="searchResult">
    <search-bar/>
    <div style="text-align: center; padding: 10px 0px">
      <h1 style="font-size: 20px">RESULT FOR SEARCH...</h1>
    </div>
    <public-key/>
    <block-info/>
    <transaction/>
    <recent-trans/>
  </div>
</template>

<script>
import SearchBar from '@/components/global/SearchBar.vue'
import PublicKey from '@/components/searchResult/PublicKey.vue'
import BlockInfo from '@/components/searchResult/BlockInfo.vue'
import Transaction from '@/components/searchResult/Transaction.vue'
import RecentTrans from '@/components/searchResult/RecentTrans.vue'

export default {
  name: 'SearchResult',
  components: {
    SearchBar,
    PublicKey,
    BlockInfo,
    Transaction,
    RecentTrans
  }
}
</script>

<style lang="sass" scoped>

</style>