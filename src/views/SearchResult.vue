<template>
  <!-- Search Result View -->
  <div class="searchResult">
    <!-- Search Bar Component -->
    <node-admin/>
    <search-bar/>
    <!-- End Search Bar Component -->

    <!-- Search Result Main Message Container -->
    <div class="search-cont animated fast fadeIn">
      <div class="search-type">{{ type }}</div>
      <div class="search-value">{{ value }}</div>
    </div>
    <!-- End Search Result Main Message Container -->

    <error/>

    <!-- Public Key Component -->
    <public-key v-if="type === 'Public Key' || type === 'Address'" :detail="param"/>
    <multisig v-if="multisigActive" :info="multisigData" :cosignatories="cosignList" :relatedAccount="multisigRelatedAccount"/>
    <!-- End Public Key Component -->

    <!-- Block Info Component -->
    <block-info v-if="type === 'Block Height'" :detail="param"/>
    <!-- Block Info Component -->

    <!-- Transaction Component -->
    <transaction v-if="type === 'Transaction Hash'" :detail="param" @runOpen="openModal" @runPush="pushInfo"/>
    <!-- End Transaction Component -->

    <namespace-info v-if="type === 'Namespace'" :detail="param"/>

    <!-- Assets Component -->
    <asset-info v-if="type === 'Asset ID' || type === 'Asset Name'" :detail="param" :arrayTransactions="tableData"/>

    <div class="address-list" v-if="type === 'Public Key' || type === 'Address'">
      <div class="bititle">
        <h1 class="supertitle" :class="{ 'activeList': activeList === 'nam', 'inactiveList': activeList !== 'nam' }" @click="changeList('nam')">
          Namespaces
        </h1>
        <h1 class="supertitle" :class="{ 'activeList': activeList === 'mos', 'inactiveList': activeList !== 'mos' }" @click="changeList('mos')">
          Other Assets
        </h1>
      </div>
      <div v-if="assetLoader === true" style="padding: 10px 0px">
        <mdb-progress bgColor="cyan darken-3" indeterminate />
      </div>

      <account-namespace v-show="activeList === 'nam'" v-if="type === 'Public Key' || type === 'Address'" :namespacesList="linkNamespaces"/>

      <assets v-show="activeList === 'mos'" v-if="showRecentAsset && blockAssets !== null && blockAssets.length > 0" :arrayTransactions="blockAssets" :nameLabel="'Others Assets'" @viewAsset ="openModal" @pushInfo="pushInfo"/>

      <div class="emptyMosNam animated fast fadeIn" v-show="activeList === 'mos'" v-if="assetLoader === false && blockAssets === null">
        No assets yet
      </div>

      <div class="emptyMosNam animated fast fadeIn" v-show="activeList === 'nam'" v-if="linkNamespaces === undefined || linkNamespaces.length === 0">
        No namespaces yet
      </div>
    </div>

    <!-- End Assets Component -->

    <div v-if="$route.params.type !== 'assetInfo'">
      <!-- Transactions Component -->
      <recent-trans v-if="tableData.length > 0" :arrayTransactions="tableData"/>
      <!-- End Transactions Component -->
    </div>

    <div class="special-bottom animated faster fadeInDown" v-if="tableNextData.length !== 0" @click="loadMore">
      <div>
        <span>Load More</span>
        <span class="value" v-if="buttonLoaderActive"><mdb-spinner small color="yellow"/></span>
      </div>
    </div>

    <!-- Modal Component -->
    <modal :param="modalInfo" :active="modalActive" :run="closeModal" :title="modalTitle"/>
    <!-- End Modal Component -->

  </div>
  <!-- End Search Result View -->
</template>

<script>
import NodeAdmin from '@/components/home/NodeAdmin.vue'
import SearchBar from '@/components/global/SearchBar.vue'
import Error from '@/components/global/Error.vue'
import PublicKey from '@/components/searchResult/PublicKey.vue'
import AccountNamespace from '@/components/searchResult/AccountNamespace.vue'
import Multisig from '@/components/searchResult/MultisigInfo.vue'
import BlockInfo from '@/components/searchResult/BlockInfo.vue'
import Transaction from '@/components/searchResult/Transaction.vue'
import NamespaceInfo from '@/components/searchResult/NamespaceInfo.vue'
import AssetInfo from '@/components/searchResult/AssetInfo.vue'
import Modal from '@/components/global/Modal.vue'
import RecentTrans from '@/components/searchResult/RecentTrans.vue'
import Assets from '@/components/searchResult/Assets.vue'
import { Address, Deadline, NetworkType , Id, NamespaceId, NamespaceName, MosaicId, QueryParams, PageQueryParams, PublicAccount } from 'tsjs-xpx-chain-sdk'
import proximaxProvider from '@/services/proximaxProviders.js'
import { mdbProgress, mdbSpinner } from 'mdbvue'
import axios from 'axios'

export default {
  name: 'SearchResult',
  components: {
    NodeAdmin,
    SearchBar,
    Error,
    PublicKey,
    Multisig,
    AccountNamespace,
    BlockInfo,
    Transaction,
    NamespaceInfo,
    AssetInfo,
    RecentTrans,
    Assets,
    Modal,
    mdbProgress,
    mdbSpinner,
  },
  data () {
    return {
      param: {},
      type: '',
      value: '',
      recent: [],
      tableData: [],
      tableNextData: [],
      page: 0,
      pageSize: 25,
      buttonLoaderActive: false,
      showRecentAsset: false,
      blockAssets: null,
      assetLoader: false,
      // Public Key
      errorPublicKey: false,
      linkNamespaces: undefined,
      activeList: 'nam',

      // Multisig
      multisigActive: false,
      multisigData: undefined,
      multisigRelatedAccount: [],
      errorMultisig: false,

      // modalConfig
      modalInfo: [],
      modalActive: false,
      modalTitle: 'Info',

      invalidPublicKey: '0000000000000000000000000000000000000000000000000000000000000000'
    }
  },
  mounted () {
    // $proxProvider and proximaxProvider is Proximax Service (Proximax Provider Service)
    // included in the main instance of vue (No need import)
    /**
     * Mounted - Lifecycle Hook of Vue
     * Search Result View
     *
     * In the search results, the mounted method, performs an analysis
     * of what type of data is being received and depending on this shows
     * the appropriate component to represent the data, in this case it
     * alternates on the components: Public Key, Block Info, and Transactions
     *
     * This component is widely connected to the Search Bar component
     */
    if (this.$route.params.type === 'publicKey' || this.$route.params.type === 'address') {
      let tmp
      if (this.$route.params.id.length === 64) {
        if (this.$store.state.netType === undefined) {
          let response = axios.get('./config/config.json')
          tmp = this.$proxProvider.createPublicAccount(this.$route.params.id, response.data.NetworkType.number)
        } else {
          tmp = this.$proxProvider.createPublicAccount(this.$route.params.id, this.$store.state.netType.number)
        }
        this.getInfoAccountAndViewTransactions(tmp.address.address)
      } else if (this.$route.params.id.length === 40 || this.$route.params.id.length === 46) {
        this.getInfoAccountAndViewTransactions(this.$route.params.id)
      }
    } else if (this.$route.params.type === 'blockHeight') {
      this.getBlockByHeight(this.$route.params.id)
    } else if (this.$route.params.type === 'hash') {
      this.getInfoTransaction(this.$route.params.id)
    } else if (this.$route.params.type === 'namespaceInfo') {
      if (this.isHex(this.$route.params.id) === false) {
        let tmp = this.$route.params.id
        let tmp2 = new NamespaceId(tmp)
        this.getNamespaceInfo(tmp2.id.toHex())
      } else {
        this.getNamespaceInfo(this.$route.params.id)
      }
    } else if (this.$route.params.type === 'assetInfo') {
      if (this.isHex(this.$route.params.id) === false) {
        let tmp = this.$route.params.id
        let tmp2 = new NamespaceId(tmp)
        this.$proxProvider.getNamespacesInfo(tmp2.id).subscribe(
          response => {
            this.getAssetInfo(response.alias.mosaicId.toHex())
          },
          error => {
            this.$store.dispatch('updateErrorInfo', {
              active: true,
              message: 'Asset not found',
              submessage: 'Check the information provided and try again'
            })
          }
        )
      } else {
        this.getAssetInfo(this.$route.params.id)
      }
    }
    this.value = this.$route.params.id
  },
  methods: {
    // For Public Key and Hash Transactions Data
    /**
     * Get Info Account And View Transactions
     *
     * This method performs the request and receipt of the information of an account,
     * and also returns your recent transactions shown in a component dedicated to it.
     *
     * @param { String } Account
     */
    async getInfoAccountAndViewTransactions (account) {
      const addr = Address.createFromRawAddress(account)
      const xpx = this.$store.state.xpx
      let errorActive1 = false
      let errorActive2 = false
      this.assetLoader = true

      this.$proxProvider.getAccountInfo(addr).subscribe(
        resp => {
          // Assign the response to accountInfo and show the account information
          if ([null, undefined].includes(resp.publicKey)) {
            let tmpObj = {
              address: resp.address,
              publicKey: this.invalidPublicKey
            }
            this.param = tmpObj
          } else {
            this.param = resp
          }

          this.showComponent()
          // If your account information has tiles, look up your information and name to display them in the tile table
          if (resp.mosaics.length > 0) {
            let filteredTrans = resp.mosaics.filter(el => el.id.toHex() !== xpx)
            let tmpArr = []

            if (filteredTrans.length === 0) {
              this.assetLoader = false
            }


            filteredTrans.forEach((el, index) => {
              this.$proxProvider.getAsset(el.id).subscribe(
                assetResponse => {
                  let amountCompact = el.amount.compact()
                  let mosHeight = assetResponse.height.compact()
                  let mosDurat = (assetResponse.duration === undefined) ? 0 : assetResponse.duration.compact()
                  this.$proxProvider.getAssetsName([assetResponse.mosaicId]).subscribe(
                    responseName => {
                      let tmpObj = {
                        name: (responseName[0].names.length > 0) ? responseName[0].names[0].name : '',
                        id: el.id.toHex(),
                        owner: (resp.publicKey === assetResponse.owner.publicKey) ? 'true' : 'false',
                        quantity: (assetResponse.divisibility === 0) ? amountCompact : this.$utils.fmtDivisibility(el.amount.compact(), assetResponse.divisibility),
                        expired: (this.$store.state.currentBlock.height >= (mosHeight + mosDurat)) ? false : true
                      }
                      tmpArr.push(tmpObj)
                      if (index + 1 === filteredTrans.length) {
                        this.blockAssets = tmpArr
                        this.showRecentAsset = !this.showRecentAsset
                        this.assetLoader = false
                      }
                    },
                    error => {
                      console.warn(error)
                      this.blockAssets = null
                      this.showRecentAsset = !this.showRecentAsset
                      this.assetLoader = false
                    }
                  )
                }, err => {
                  console.warn(err)
                }
              )
            })
          } else {
            this.assetLoader = false
          }

          this.viewTransactionsFromPublicAccount()
        },
        error => {
          this.errorPublicKey = true
        }
      )

      axios.get(`${this.$store.getters.getCurrentNodeProtocol}/account/${addr.address}/namespaces`)
        .then(response => {
          let tmpArr = []
          let revisionArray = []
          if (response.data.length !== 0) {
            response.data.forEach(el => {
              let tmpObj = {
                status: (el.meta.active) ? 'Active' : 'Inactive'
              }

              let requestArr = []
              let currentLevel = 0
              if (el.namespace.level2 !== undefined) {
                currentLevel = 2
                requestArr.push(new Id(el.namespace.level2))
                requestArr.push(new Id(el.namespace.level1))
                requestArr.push(new Id(el.namespace.level0))
                tmpObj.id = new Id(el.namespace.level2).toHex()
              } else if (el.namespace.level1 !== undefined) {
                currentLevel = 1
                requestArr.push(new Id(el.namespace.level1))
                requestArr.push(new Id(el.namespace.level0))
                tmpObj.id = new Id(el.namespace.level1).toHex()
              } else if (el.namespace.level0 !== undefined) {
                currentLevel = 0
                requestArr.push(new Id(el.namespace.level0))
                tmpObj.id = new Id(el.namespace.level0).toHex()
              }


              this.$proxProvider.getNamespacesName(requestArr).subscribe(
                responseName => {
                  let verifyLength = (arr) => {
                    let arrResult
                    let biggerNumber = 0
                    let index = undefined

                    arrResult = arr.map(el => el.length)

                    arrResult.forEach(el => {
                      if (el > biggerNumber) {
                        biggerNumber = el
                      }
                    })

                    return arr[arrResult.indexOf(biggerNumber)]
                  }

                  if (currentLevel === 0) {
                    tmpObj.name = responseName[0].name
                  } else if (currentLevel === 1) {
                    tmpObj.name = verifyLength([responseName[0].name, responseName[1].name])
                  } else if (currentLevel === 2) {
                    tmpObj.name = verifyLength([responseName[0].name, responseName[1].name, responseName[2].name])
                  }

                  tmpArr.push(tmpObj)
                }
              )
            })
            this.linkNamespaces = tmpArr
          } else {
            this.linkNamespaces = []
          }
        })
        .catch(err => {
          this.linkNamespaces = []
        })

      axios.get(`${this.$store.getters.getCurrentNodeProtocol}/account/${addr.address}/multisig`)
        .then(response => {
          let objTmp = {
            account: response.data.multisig.account,
            accountAddress: response.data.multisig.accountAddress,
            minApproval: response.data.multisig.minApproval,
            minRemoval: response.data.multisig.minRemoval
          }


          this.cosignList = Array.from(response.data.multisig.cosignatories)
          this.multisigRelatedAccount = Array.from(response.data.multisig.multisigAccounts),
          this.multisigConsignatories = Array.from(response.data.multisig.cosignatories)
          this.multisigData = objTmp
          this.multisigActive = true

          if (this.type === '') {
            this.type = 'Multisig Account'
          }
        })
        .catch(error => {
          this.errorMultisig = true
        })
    },

    /**
     * Get Block By Height
     *
     * This method performs the request and receipt of the information of an block,
     * and also returns your recent transactions shown in a component dedicated to it.
     *
     * @param { String } block
     */
    async getBlockByHeight (block) {
      this.$proxProvider.blockHttp.getBlockByHeight(parseInt(block)).subscribe(
        async next => {
          next.date = this.$utils.fmtTime(new Date(next.timestamp.compact() + Deadline.timestampNemesisBlock * 1000))
          next.difficulty = (next.difficulty.compact()/Math.pow(10, 14)*100).toFixed(2) + "%"
          next.totalFee = this.$utils.fmtAmountValue(next.totalFee.compact())
          this.param = {
            height: block,
            timestamp: next.date,
            publicKey: next.signer.publicKey,
            hash: next.hash,
            difficulty: next.difficulty,
            txes: next.numTransactions,
            fee: next.totalFee
          }
          this.showComponent()

          if (this.param.txes > 0) {
            this.getBlockTransactions()
          }
        },
        error => {
          this.$store.dispatch('updateErrorInfo', {
            active: true,
            message: 'Block Height not found',
            submessage: 'Check the information provided and try again'
          })
        }
      )
    },

    /**
     * Get Info Transaction
     *
     * This method performs the request and receipt of information about a transaction
     *
     * @param { String } hast
     */
    getInfoTransaction: function (hast) {
      this.$proxProvider.getTransactionInformation(hast).subscribe(
        resp => {
          this.param = resp
          this.showComponent()
        },
        error => {
          this.$store.dispatch('updateErrorInfo', {
            active: true,
            message: 'Transaction not found',
            submessage: 'Check the information provided and try again'
          })
        }
      )
    },

    getNamespaceInfo (namespaceHex) {
      let namespaceId = Id.fromHex(namespaceHex)
      this.$proxProvider.getNamespacesInfo(namespaceId).subscribe(
        response => {
          this.$proxProvider.getNamespacesName([namespaceId]).subscribe(
            nameResponse => {
              this.param = response
              nameResponse = nameResponse.reverse()
              if (nameResponse.length > 1) {
                this.param.name = `${nameResponse[0].name}.${nameResponse[1].name}`
              } else {
                this.param.name = nameResponse[0].name
              }
              this.showComponent()
            }
          )
        },
        error => {
          this.$store.dispatch('updateErrorInfo', {
            active: true,
            message: 'Namespace not found',
            submessage: 'Check the information provided and try again'
          })
        }
      )
    },

    getAssetInfo (assetHex) {
      let assetId = Id.fromHex(assetHex)
      this.$proxProvider.getAsset(assetId).subscribe(
        response => {
          this.param = response
          this.$proxProvider.getAssetsName([assetId]).subscribe(
            nameResponse => {
              this.param = response
              this.param.name = (nameResponse[0].names && nameResponse[0].names.length !== 0) ? nameResponse[0].names[0].name : undefined
              this.showComponent()
            }
          )

          this.getRichlist()
        },
        error => {
          this.$store.dispatch('updateErrorInfo', {
            active: true,
            message: 'Asset not found',
            submessage: 'Check the information provided and try again'
          })
        }
      )
    },

    showComponent () {
      if (this.$route.params.type === 'publicKey') {
        this.type = 'Public Key'
      } else if (this.$route.params.type === 'blockHeight') {
        this.type = 'Block Height'
      } else if (this.$route.params.type === 'hash') {
        this.type = 'Transaction Hash'
      } else if (this.$route.params.type === 'address') {
        this.type = 'Address'
      } else if (this.$route.params.type === 'namespaceInfo') {
        this.type = 'Namespace'
      } else if (this.$route.params.type === 'assetInfo') {
        this.type = (isNaN(parseInt(this.$route.params.id, 16))) ? 'Asset Name' : 'Asset ID'
      }
      this.value = this.$route.params.id
    },

    /**
     * View Transactions From Public Account / Address
     *
     */
    async viewTransactionsFromPublicAccount() {
      try {
        if (this.tableNextData.length > 0) {
          // Get prefetch data
          this.tableNextData.forEach(element => {
            element.fee = this.$utils.fmtAmountValue(element.maxFee.compact())
            element.deadline = this.$utils.fmtTime(new Date(element.deadline.value.toString()))
            this.tableData.push(element)
          })
          if (this.tableNextData.length < this.pageSize) {
            this.tableNextData = []
            return
          }
        } else if (this.tableData.length > 0) {
          // Do nothing since there is no more data to fetch
          return
        } else {
          // Get data during initial loading
          if (this.param.publicKey == this.invalidPublicKey) {
            this.tableData = await this.$proxProvider.accountHttp.incomingTransactions(this.param.address, new QueryParams(this.pageSize)).toPromise()
          } else {
            this.tableData = await this.$proxProvider.accountHttp.transactions(this.param.publicAccount, new QueryParams(this.pageSize)).toPromise()
          }
          for (const index in this.tableData) {
            this.tableData[index].fee = this.$utils.fmtAmountValue(this.tableData[index].maxFee.compact())
            this.tableData[index].deadline = this.$utils.fmtTime(new Date(this.tableData[index].deadline.value.toString()))
          }
        }

        if (this.tableData.length == (this.page + 1) * this.pageSize) {
          // Prefetch data
          this.page += 1
          if (this.param.publicKey == this.invalidPublicKey) {
            this.tableNextData = await this.$proxProvider.accountHttp.incomingTransactions(this.param.address, new QueryParams(this.pageSize, this.tableData[this.tableData.length - 1].transactionInfo.id)).toPromise()
          } else {
            this.tableNextData = await this.$proxProvider.accountHttp.transactions(this.param.publicAccount, new QueryParams(this.pageSize, this.tableData[this.tableData.length - 1].transactionInfo.id)).toPromise()
          }
          if (this.tableData[this.tableData.length - 1].transactionInfo.id == this.tableNextData[this.tableNextData.length - 1].transactionInfo.id) {
            this.tableNextData = []
          }
        }
      } catch (error) {
        console.log('ACCOUNT TRANSACTIONS ERROR',error)
      }
    },

    /**
     * Get Block Transactions
     *
     */
    async getBlockTransactions() {
      try {
        if (this.tableData.length != this.param.txes) {
          if (this.tableData.length > 0) {
            this.tableNextData = await this.$proxProvider.blockHttp.getBlockTransactions(parseInt(this.param.height), new QueryParams(this.pageSize, this.tableData[this.tableData.length - 1].transactionInfo.id)).toPromise()
          } else {
            this.tableNextData = await this.$proxProvider.blockHttp.getBlockTransactions(parseInt(this.param.height), new QueryParams(this.pageSize)).toPromise()
          }
          this.tableNextData.forEach(element => {
            element.fee = this.$utils.fmtAmountValue(element.maxFee.compact())
            element.deadline = this.$utils.fmtTime(new Date(element.deadline.value.toString()))
            this.tableData.push(element)
          })
        }

        // Deliberate not else case because tableData has new elements since new transactions has been added
        if (this.tableData.length == this.param.txes) {
          this.tableNextData = []
        }
      } catch (error) {
        console.log('BLOCK TRANSACTIONS ERROR',error)
      }
    },

    /**
     * Get Rich List
     *
     */
    async getRichlist() {
      try {
        if (this.tableNextData.length > 0) {
          // Get prefetch data
          this.tableData = this.tableData.concat(this.tableNextData)
        } else if (this.tableData.length > 0) {
          // Do nothing since there is no more data to fetch
          return
        } else {
          // Get data during initial loading
          this.tableData = await this.$proxProvider.assetHttp.getMosaicRichlist(this.param.mosaicId, {pageSize: this.pageSize, page: this.page}).toPromise()
        }

        if (this.tableData.length == (this.page + 1) * this.pageSize) {
          // Prefetch data
          this.page += 1
          this.tableNextData = await this.$proxProvider.assetHttp.getMosaicRichlist(this.param.mosaicId, {pageSize: this.pageSize, page: this.page}).toPromise()
        } else {
          this.tableNextData = []
        }
      } catch (error) {
        console.log('ASSET RICH LIST ERROR',error)
      }
    },

    /**
     * Open Modal
     *
     * Se inicia el componente modal, de modo que aparece en la pantalla
     * y al mismo tiempo se configura el título.
     */
    openModal (title) {
      this.modalTitle = title
      this.modalActive = true
    },

    /**
     * Push Info
     *
     * This function performs the configuration of the information
     * that you want to show in the modal
     */
    pushInfo (info) {
      this.modalInfo = info
    },

    /**
     * Close Modal
     *
     * This method works in reverse of 'openModal' method,
     * but emptying the information, preparing the modal for a next use
     */
    closeModal () {
      this.modalActive = false
      this.modalInfo = []
    },

    showError () {
      this.$store.dispatch('updateErrorInfo', {
        active: true,
        message: 'Address or public key not found',
        submessage: 'Check the information provided and try again'
      })
    },

    changeList (list) {
      this.activeList = list
    },

    isHex (value) {
      let regex =  /^[0-9A-Fa-f]+$/
      return regex.test(value)
    },

    loadMore () {
      this.buttonLoaderActive = true
      if (this.$route.params.type === 'publicKey' || this.$route.params.type === 'address') {
        this.viewTransactionsFromPublicAccount()
      } else if (this.$route.params.type === 'blockHeight') {
        this.getBlockTransactions()
      } else if (this.$route.params.type === 'assetInfo') {
        this.getRichlist()
      }
      this.buttonLoaderActive = false
    }
  },
  watch: {
    errorMultisig (n, o) {
      if (this.errorMultisig && this.errorPublicKey) {
        this.showError()
      }
    },

    errorPublicKey (n, o) {
      if (this.errorMultisig && this.errorPublicKey) {
        this.showError()
      }
    }
  }
}
</script>

<style lang="sass" scoped>
.activeList
  background: #2BA1B9
  color: white !important

.inactiveList
  background: #f4f4f4
  color: black
  cursor: pointer
  &:hover
    background: silver
    color: white

.bititle
  width: 100%
  display: flex
  flex-flow: row
  padding: 0px 10px
  justify-content: center

.supertitle
  display: flex
  flex-flow: row nowrap
  justify-content: center
  color: #2BA1B9
  margin: 0px
  font-size: 17px
  padding: 5px 20px
  border-radius: 20px
  &:first-child
    border-radius: 20px 0px 0px 20px
  &:last-child
    border-radius: 0px 20px 20px 0px

.search-cont
  text-align: center
  padding: 15px 0px
  border-bottom: 1px solid silver

.search-type
  color: #2BA1B9
  font-weight: bold
  font-size: 25px

.search-value
  padding: 0px 10px 5px
  color: black
  font-size: 20px
  text-transform: uppercase
  font-weight: bold
  word-wrap: break-word

.emptyMosNam
  width: 100%
  color: grey
  padding: 5px
  text-align: center
  background: #f4f4f4
  margin: 10px 0px

.special-bottom
  padding-bottom: 10px
  margin-top: 5px
  display: flex
  flex-flow: row nowrap
  justify-content: space-around
  color: white
  border-radius: 20px
  background: transparent
  & > div
    display: flex
    flex-flow: column
    align-items: center
    padding: 5px
    background: #2d819b
    border-radius: 20px
    padding: 5px 40px
    & > span:first-child
      font-size: 15px !important
      text-transform: uppercase
      font-weight: bold
      color: white
    & > span:last-child
      text-align: center
      font-size: 10px
      text-transform: uppercase
      font-weight: bold
      color: white

@media screen and (max-width: 550px)
  .search-type
    font-size: 20px

  .search-value
    font-size: 15px
</style>
