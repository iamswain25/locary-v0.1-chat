<template>
  <v-layout column fill-height>
    <v-alert
      v-model="alert"
      type="info"
      style="position:fixed;width:100%;"
      transition="fade-transition"
    >
      {{alertMessage}}
    </v-alert>
    <v-flex class="chat-container" style="flex-grow: 100" ref="chatContainer">
      <div v-show="empty">no more data to fetch.</div>
      <div v-show="loading">loading...</div>
      <message :messages="locaries"></message>
    </v-flex>
    <v-flex style="flex-grow: 1">
      <div class="text-xs-center" v-if="userRef === null">
        <firebaseui/>
        <!-- <v-dialog v-model="dialog">
          <v-btn slot="activator" color="yellow lighten-2" block large>Sign In &amp; Contribute to Locary</v-btn>
          <firebaseui/>
        </v-dialog>-->
      </div>
      <v-layout style="position: relative;" v-else>
        <div style="width: calc(100% - 100px);">
          <v-textarea
            v-on:keyup.ctrl.enter="sendMessage"
            placeholder="Type here..."
            v-model="content"
            auto-grow
            rows="2"
            hide-details
            background-color="transparent"
            style="padding-left: 10px;"
            :maxlength="maxlength"
          />
        </div>
        <button type="button" class="stamp" @click="sendMessage">stamp</button>
      </v-layout>
    </v-flex>
  </v-layout>
</template>

<style>
.scrollable {
  overflow-y: auto;
  height: 90vh;
}

.chat-container {
  box-sizing: border-box;
  overflow-y: auto;
  padding: 10px;
  background-color: #f2f2f2;
  max-height: calc(100vh - 9.2rem);
}

/* @media (max-width: 480px) {
  .chat-container .content {
    max-width: 80%;
  }
} */
.stamp {
  font-size: 16px;
  align-self: center;
  position: absolute;
  height: 100%;
  margin: 0;
  right: 0;
  padding: 0;
  background-color: #1976d2;
  width: 88px;
  color: white;
  box-sizing: border-box;
  /* border-radius: 10px; */
  -webkit-box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.2),
    0 1px 1px 0 rgba(0, 0, 0, 0.14), 0 2px 1px -1px rgba(0, 0, 0, 0.12);
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.2), 0 1px 1px 0 rgba(0, 0, 0, 0.14),
    0 2px 1px -1px rgba(0, 0, 0, 0.12);
}
</style>

<script>
import message from './Message.vue'
import firebaseui from '@/components/Auth/firebaseui'
import { geoLocary, GeoPoint, Timestamp } from '../../store/firestore'
import uuidV4 from 'uuid/v4'

export default {
  data () {
    return {
      dialog: false,
      content: '',
      empty: true,
      locaries: [],
      totalChatHeight: 0,
      geoQuery: null,
      queryLimit: 5,
      alert: false,
      alertMessage: 'this is alert',
      maxlength: 300
    }
  },
  components: {
    message,
    firebaseui
  },
  mounted () {
    this.$store.commit('setLoading', true)
    const store = this.$store
    const authUser = this.authUser
    const uid = (authUser) ? authUser.uid : uuidV4()
    console.time('getCurrentPosition')
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(position => {
        console.timeEnd('getCurrentPosition', position)
        store.commit('setPosition', position.coords)
        store.dispatch('rtdb_presence', { uid })
        store.dispatch('getListeningRadius')
          .then(this.loadChat)
        // this.loadChat()
      })
    }
  },
  computed: {
    userRef () {
      return this.$store.getters.userRef
    },
    displayName () {
      return this.$store.getters.userInfo.displayName
    },
    radius () {
      return this.$store.getters.listeningRadius
    },
    loading () {
      return this.$store.getters.loading
    }
  },
  methods: {
    loadChat () {
      const that = this
      // this.$store.commit('setLoading', true)
      const coords = this.$store.getters.position
      const radius = this.radius
      const center = new GeoPoint(coords.latitude, coords.longitude)
      // const query = ref => ref.limit(this.queryLimit) // limit(8) .orderBy('d.createdAt', 'desc')
      const geoQuery = geoLocary.query({ center, radius })
      this.geoQuery = geoQuery
      geoQuery.on('ready', function () {
        that.locaries.sort((a, b) => (a.createdAt.toMillis() >= b.createdAt.toMillis()) ? 1 : -1)
        if (that.loading) {
          that.scrollToBottom()
        }
      })
      geoQuery.on('key_entered', function (key, doc, distance) {
        if (doc.userRef === that.userRef) {
          doc.isRead = true
        }
        Object.assign(doc, { key, distance })
        that.locaries.push(doc)
      })
      geoQuery.on('key_modified', function (key, doc, distance) {
        // console.log(`key_modified ${key}`, doc)
        const modifiedIndex = that.locaries.findIndex(doc => doc.key === key)
        Object.assign(that.locaries[modifiedIndex], doc)
      })
      geoQuery.on('key_exited', function (key, doc, distance) {
        // console.log(`key_exited ${key}, doc)
        const modifiedIndex = that.locaries.findIndex(doc => doc.key === key)
        that.locaries.splice(modifiedIndex, 1)
      })
      this.watchLocation()
    },
    updateLocaries () {
      const coords = this.$store.getters.position
      const radius = this.radius
      const center = new GeoPoint(coords.latitude, coords.longitude)
      this.geoQuery.updateCriteria({ center, radius })
    },
    watchLocation () {
      const that = this
      const store = this.$store
      if (navigator.geolocation) {
        navigator.geolocation.watchPosition(position => {
          const oldCoords = this.$store.getters.position
          const newCoords = position.coords
          if (JSON.stringify(oldCoords) !== JSON.stringify(newCoords)) {
            store.commit('setPosition', position.coords)
            store.dispatch('updateLocation')
            that.updateLocaries()
          }
        })
      }
    },
    sendMessage () {
      if (this.content.trim() === '') {
        return false
      }
      const store = this.$store
      const coords = this.$store.getters.position
      const message = {
        userRef: store.getters.userRef,
        displayName: store.getters.userInfo.displayName,
        photoURL: store.getters.userInfo.photoURL,
        coordinates: new GeoPoint(coords.latitude, coords.longitude),
        body: this.content,
        createdAt: Timestamp.fromDate(new Date()),
        accuracy: coords.accuracy,
        readCount: 0,
        clickCount: 0,
        threadCount: 0
      }
      geoLocary
        .add(message)
        .then((docRef) => {
          console.log(docRef)
          this.content = ''
          this.scrollToBottom()
        })
    },
    onScroll (ev) {
      const that = this
      if (ev.target.scrollTop < 1 && !this.empty) {
        that.$store.commit('setLoading', true)
        this.queryLimit += 5
        const query = ref => ref.limit(this.queryLimit) // limit(8) .orderBy('d.createdAt', 'desc')
        this.geoQuery.updateCriteria({ query })
        this.geoQuery.on('ready', function () {
          that.locaries.sort((a, b) => a.createdAt.toMillis() > b.createdAt.toMillis())
          that.$store.commit('setLoading', false)
          that.scrollTo()
        })
      }
    },
    scrollTo () {
      this.$nextTick(() => {
        let currentHeight = this.$refs.chatContainer.scrollHeight
        let difference = currentHeight - this.totalChatHeight
        const container = this.$el.querySelector('.chat-container')
        container.scrollTop = difference
        this.totalChatHeight = currentHeight
      })
    },
    scrollToBottom () {
      this.$nextTick(() => {
        const container = this.$refs.chatContainer
        container.scrollTop = container.scrollHeight
        console.log(`.chat-container.scrollHeight: ${container.scrollHeight}`)
        this.totalChatHeight = container.scrollHeight
        this.$store.commit('setLoading', false)
      })
    },
    triggerAlertMessage (msg) {
      this.alertMessage = msg
      this.alert = true
      setTimeout(() => {
        this.alert = false
      }, 1000)
    }
  },
  destroyed () {
    // window.removeEventListener('scroll', this.onScroll)
  },
  watch: {
    radius () {
      if (this.geoQuery !== null) {
        this.updateLocaries()
      }
    },
    content (newVal) {
      const maxlength = this.maxlength
      if (newVal.length >= maxlength) {
        this.triggerAlertMessage(`cannot exceed ${maxlength} charactors`)
      }
    }
  }
}
</script>
