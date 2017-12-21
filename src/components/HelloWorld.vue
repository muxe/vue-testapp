<template>
  <div class="hello">
    <h2>id-provider stuff</h2>
    <div>
      <button @click="startPolling()">Start Polling</button>
      <button @click="stopPolling()">Stop Polling</button>
    </div>

    <h2>Metadata Test</h2>
    <div v-if="hasIdManager">
      <div>
        <input type="text" v-model="namespace" placeholder="namespace"></input>
        <input type="text" v-model="inputKey" placeholder="key"></input>
        <input type="text" v-model="inputValue" placeholder="value"></input>
        <button @click="storeMetadata(inputKey, inputValue, requestedNamespace)">Store Metadata</button>
        <button @click="readMetadata(inputKey, requestedNamespace)">Read Metadata</button>
      </div>
      <div>
        <button @click="requestPermissions(examplePermissionJson)">Request Example Permissions</button>
      </div>
    </div>
    <div v-else>
      No Id Manager found
    </div>
  </div>
</template>

<script>
import IdManagerProvider from '@aeternity/id-manager-provider'
import Web3 from 'web3'

let idWeb3 = null
let provider = null
let idManager = null

export default {
  name: 'HelloWorld',
  data () {
    return {
      credentials: null,
      hasIdManager: false,
      namespace: 'identity.aepps.com',
      inputKey: '',
      inputValue: '',
      examplePermissionJson: {
        'http://identity.aepps.com': {
          'userName': {
            read: true,
            write: true
          },
          'userPic': {
            read: true,
            write: false
          }
        },
        'wall.aepps.com': ['entries', 'likes']
      }
    }
  },
  computed: {
    requestedNamespace () {
      return this.namespace && this.namespace !== '' ? this.namespace : null
    }
  },
  methods: {
    startPolling: function () {
      idManager.startPolling()
    },
    stopPolling: function () {
      idManager.stopPolling()
    },

    storeMetadata: async function (key, value, namespace) {
      if (idWeb3) {
        let response = await idManager.storeMetadata(key, value, namespace)
        console.log('storeMetadata', response)
        if (!response.success && response.error && response.error.code === 210) {
          let request = {}
          request[namespace] = {}
          request[namespace][key] = {write: true, read: true}
          if (await this.requestPermissions(request)) {
            // repeat request
            this.storeMetadata(key, value, namespace)
          }
        }
      }
    },
    readMetadata: async function (key, namespace) {
      if (idWeb3) {
        let response = await idManager.readMetadata(key, namespace)
        console.log('readMetadata', response)
        if (response && response.success) {
          this.inputValue = response.value
        } else {
          if (response.error && response.error.code === 210) {
            let request = {}
            request[namespace] = {}
            request[namespace][key] = {read: true}
            if (await this.requestPermissions(request)) {
              // repeat request
              this.readMetadata(key, namespace)
            }
          }
        }
      }
    },
    requestPermissions: async function (permissionsJson) {
      if (idWeb3) {
        let response = await idManager.requestPermissions(permissionsJson)
        console.log('requestPermissions', response)
        return response && response.success
      }
      return false
    }
  },
  mounted: function() {
    // let idWeb3 = null
    idManager = new IdManagerProvider({
      skipSecurity: process.env.NODE_ENV === 'development'
    })
    idManager.checkIdManager().then( (idManagerPresent) => {
      console.log('idManagerPresent', idManagerPresent)
      provider = idManager.provider
      if (idManagerPresent) {
        idWeb3 = new Web3(idManager.provider)
        console.log('idWeb3', idWeb3)
        idManager.startPolling()
        this.hasIdManager = true
      }
    });
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
