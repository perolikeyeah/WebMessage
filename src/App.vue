<template>
  <div id="app" :class="{ nostyle: !($store.state.macstyle || process.platform === 'darwin'), maximized: (maximized || !$store.state.acceleration) && process.platform !== 'darwin' }">
    <settings ref="settingsModal" @saved="connectWS"></settings>
    <div id="nav" :class="{ notrans: !$store.state.acceleration }">
      <div class="titlebar">
        <div class="buttons" v-if="$store.state.macstyle || process.platform === 'darwin'">
          <div class="close" @click="closeWindow">
            <span class="closebutton"><span>x</span></span>
          </div>
          <div class="minimize" @click="minimizeWindow">
            <span class="minimizebutton"><span>&ndash;</span></span>
          </div>
          <div class="zoom" @click="maximizeWindow">
            <span class="zoombutton"><span>+</span></span>
          </div>
        </div>
        <div class="menuBtn">
          <feather type="settings" stroke="rgba(152,152,152,0.5)" size="20" @click="$refs.settingsModal.openModal()"></feather>
        </div>
        <div class="menuBtn">
          <feather type="edit" stroke="rgba(36,132,255,0.65)" size="20" @click="composeMessage"></feather>
        </div>
        <div class="menuBtn" v-if="updateAvailable">
          <feather type="download" stroke="rgba(152,255,152,0.65)" size="20" @click="restart"></feather>
        </div>
      </div>
      <div class="searchContainer">
        <input type="search" placeholder="Search" class="textinput" v-model="search" />
      </div>
      <simplebar class="chats" ref="chats" data-simplebar-auto-hide="false">
        <chat v-for="chat in filteredChats" :key="chat.id"
          :chatid="chat.address"
          :author="chat.author"
          :text="chat.text"
          :date="chat.date"
          :read="chat.read"
          :docid="chat.docid"
          @navigated="setAsRead(chat)">
        </chat>
      </simplebar>
    </div>
    <div id="content">
      <transition name="fade" mode="out-in">
        <router-view></router-view>
      </transition>
    </div>
  </div>
</template>

<script>
import Chat from './components/Chat.vue'
import Settings from "./components/Settings.vue"
import simplebar from 'simplebar-vue'
import 'simplebar/dist/simplebar.css'

export default {
  name: 'App',
  components: {
    Chat,
    simplebar,
    Settings
  },
  data: function () {
    return {
      chats: [],
      limit: 50,
      offset: 0,
      loading: false,
      notifSound: null,
      updateAvailable: false,
      search: '',
      process: window.process,
      maximized: false,
      maximizing: false
    }
  },
  computed: {
    filteredChats() {
      return this.chats.filter((chat) => {
        return chat.author.toLowerCase().includes(this.search.toLowerCase()) || chat.text.toLowerCase().includes(this.search.toLowerCase())
      })
    }
  },
  methods: {
    getWindow () {
      return window.remote.BrowserWindow.getAllWindows()[0]
    },
    closeWindow () {
      if (this.$store.state.minimize) {
        ipcRenderer.send('minimizeToTray')
      } else {
        ipcRenderer.send('quitApp')
      }
    },
    minimizeWindow () {
      this.getWindow().minimize()
    },
    maximizeWindow () {
      this.maximizing = true
      const win = this.getWindow()
      if (this.maximized) {
        win.restore()
        win.setSize(700,600)
        win.center()
        if (process.platform !== 'darwin') document.body.style.borderRadius = null
      } else {
        win.maximize()
        if (process.platform !== 'darwin') document.body.style.borderRadius = '0'
      }

      this.maximized = !this.maximized
      setTimeout(() => {
        this.maximizing = false
      }, 50)
    },
    restart () {
      ipcRenderer.send('restart_app')
    },
    requestChats (clear) {
      if (this.$socket && this.$socket.readyState == 1) {
        if (this.loading) return
        this.loading = true

        this.sendSocket({ action: 'fetchChats', data: {offset: `${this.offset}`, limit: `${this.limit}`} })
      } else {
        setTimeout(this.requestChats, 100)
      }
    },
    connectWS () {
      this.chats = []
      this.offset = 0
      this.loading = false

      const baseURI = this.$store.getters.baseURI
      this.$disconnect()
      this.$connect(baseURI, {
        format: 'json',
        reconnection: true,
      })
    },
    setAsRead (chat) {
      chat.read = true
    },
    composeMessage () {
      this.$router.push('/message/new')
    }
  },
  mounted () {
    this.connectWS()

    let container = this.$refs.chats.SimpleBar.getScrollElement()
    container.addEventListener('scroll', (e) => {
      if (container.scrollTop + container.offsetHeight == container.scrollHeight && !this.loading) {
        this.requestChats()
      }
    })

    ipcRenderer.send('loaded')

    ipcRenderer.on('update_available', () => {
      ipcRenderer.removeAllListeners('update_available')
    })

    ipcRenderer.on('update_downloaded', () => {
      ipcRenderer.removeAllListeners('update_downloaded')
      this.updateAvailable = true
    })

    this.notifSound = new Audio('wm-audio://receivedText.mp3')

    if (!(this.$store.state.macstyle || process.platform === 'darwin')) {
      document.body.style.border = 'none'
      document.body.style.borderRadius = '0'
    }

    if (!this.$store.state.acceleration) {
      document.documentElement.style.backgroundColor = "black"
    }

    const win = this.getWindow()

    window.addEventListener('resize', (e) => {
      if (this.maximizing) return
      if (this.maximized) {
        win.restore()
        if (process.platform !== 'darwin') document.body.style.borderRadius = null
        this.maximized = false
      }
    })

    win.on('move', (e) => {
      e.preventDefault()
      if (this.maximizing) return
      if (this.maximized) {
        win.restore()
        if (process.platform !== 'darwin') document.body.style.borderRadius = null
        this.maximized = false
      }
    })
  },
  socket: {
    fetchChats (data) {
      if (this.offset == 0) this.chats = data
      else this.chats.push(...data)
      this.offset += this.limit
      this.loading = false
    },
    newMessage (data) {
      var chatData = data.chat[0]

      if (chatData && chatData.id) {
        var chatIndex = this.chats.findIndex(obj => obj.id == chatData.id)

        if (chatIndex > -1) {
          this.chats.splice(chatIndex, 1)
        }

        this.chats.unshift(chatData)
      }

      var messageData = data.message[0]

      if (messageData && messageData.sender != 1 && remote.Notification.isSupported()) {
        const notification = {
          title: messageData.name,
          body: messageData.text,
          silent: this.$store.state.playsound
        }

        if (this.$store.state.playsound) this.notifSound.play()
        let notif = new remote.Notification(notification)
        notif.on('click', (event, arg) => {
          if (chatData && chatData.id) {
            ipcRenderer.send('show_win')
            this.$router.push('/message/'+messageData.chatId)
          }
        })
        notif.show()

      } else if (!remote.Notification.isSupported()) {
        console.log('Notifications are not supported on this system.')
      }
    },
    onopen () {
      this.requestChats()
    },
    onerror () {
      this.loading = false
    },
    onclose () {
      this.loading = false
    }
  }
}
</script>

<style lang="scss">
.emoji {
  position: relative;
  top: 0.15em;
  width: 13px;
}

.searchContainer {
  padding-right: 6px;
}

.menuBtn {
  -webkit-app-region: no-drag;
  width: auto;
  float: right;
  margin-right: 10px;

  .feather {
    cursor: pointer;

    &:hover {
      filter: brightness(80%);
    }
  }
}

.textinput {
  background-color: rgba(200,200,200,0.1);
  width: calc(100% - 4px);
  margin: 0 !important;
  margin-left: -2px !important;
  border: 0px none;
  border-color: rgba(87,87,87,0.7) !important;
  color: #EBECEC !important;
  text-align: left !important;
}

.chats {
  margin-top: 12px;
  max-height: calc(100% - 80px);

  .simplebar-scrollbar:before {
    background: #575757;
  }
}

@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@200;300;500;700&display=swap');

html {
  height: 100%;
  max-height: 100%;
  width: 100%;
  background-color: rgba(29,29,29,0);
}

body {
  margin: 0;
  height: calc(100% - 2px);
  max-height: 100%;
  width: calc(100% - 2px);
  background-color: rgba(29,29,29, 0);
  overflow: hidden;
  border: 1px solid rgb(0,0,0);
  border-radius: 10px;
}

#app {
  font-family: 'Roboto', -apple-system, BlinkMacSystemFont, Avenir, Helvetica, Arial, sans-serif;
  font-weight: 300;
  text-align: center;
  color: #EBECEC;
  position: absolute;
  top: 1px; left: 1px; right: 1px; bottom: 1px;
  background-color: rgba(29,29,29, 0);
  border: 1px solid #4A4A4A;
  border-radius: 10px;

  &.maximized {
    border-radius: 0;

    #nav {
      border-radius: 0;
    }

    #content {
      border-radius: 0;
    }
  }

  &.nostyle {
    background-color: rgba(40,40,40,1);
    border: none;
    border-radius: 0;
    height: 100%;
    width: 100%;

    #nav {
      border-radius: 0 !important;
      margin-left: -1px;
      margin-top: -1px;
      width: 312px;
    }

    #content {
      bottom: 0; right: 0; top: 0;
      border-radius: 0;
    }
  }
}

.fade-enter {
  opacity: 0;
}

.fade-enter-active {
  transition: opacity 0.25s ease;
}

.fade-leave-active {
  transition: opacity 0.25s ease;
  opacity: 0;
}

#nav {
  background-color: rgba(40,40,40,0.93);
  width: 311px;
  padding: 9px;
  padding-right: 0;
  float: left;
  position: absolute;
  top: 0; left: 0; bottom: 0;
  border-top-left-radius: 10px;
  border-bottom-left-radius: 10px;

  &.notrans {
    background-color: rgba(40,40,40,1);
  }

  a {
    font-weight: bold;
    color: #2c3e50;

    &.router-link-exact-active {
      color: #42b983;
    }
  }
}

.titlebar {
  -webkit-app-region: drag;
  width: calc(100% - 10px);
  height: 30px;
  padding: 5px;
  padding-top: 7px;
}

.buttons {
  -webkit-app-region: no-drag;
  padding-left: 5px;
  padding-top: 3px;
  float: left;
  line-height: 0px;

  div:hover {
    filter: brightness(75%);
  }
}

.close {
  background: #ff5c5c;
  font-size: 9pt;
  line-height: 12px;
  width: 12px;
  height: 12px;
  border: 0px solid #e33e41;
  border-radius: 50%;
  display: inline-block;
}

.close:active {
  background: #c14645;
}

.closebutton {
  color: #820005;
  visibility: hidden;
  cursor: default;
}

.minimize {
  background: #ffbd4c;
  font-size: 9pt;
  line-height: 12px;
  margin-left: 8px;
  width: 12px;
  height: 12px;
  border: 0px solid #e09e3e;
  border-radius: 50%;
  display: inline-block;
}

.minimize:active {
  background: #c08e38;
}

.minimizebutton {
  color: #9a5518;
  visibility: hidden;
  cursor: default;
}

.zoom {
  background: #00ca56;
  font-size: 9pt;
  line-height: 12px;
  margin-left: 8px;
  width: 12px;
  height: 12px;
  border: 0px solid #14ae46;
  border-radius: 50%;
  display: inline-block;
}

.zoom:active {
  background: #029740;
}

.zoombutton {
  color: #006519;
  visibility: hidden;
  cursor: default;
}

#content {
  float: left;
  background-color: rgba(29,29,29, 1);
  position:fixed;
  top: 2px; left: 321px; right: 2px; bottom: 2px;
  border-top-right-radius: 10px;
  border-bottom-right-radius: 10px;
  border-left: 1px solid #000;
  overflow: hidden;
}

input:not([type="range"]):not([type="color"]):not(.message-input) {
  height: auto;
  height: inherit;
  font-size: 13px;
  height: 6px;
  padding: 10px 6px 10px 6px;
  outline: none;
  border: 1px solid rgb(213, 213, 213);
  margin: 5px;
  cursor: text;
  -webkit-app-region: no-drag;
}

input:not([type="range"]):not([type="color"]):not(.message-input):focus {
  border-radius: 1px;
  box-shadow: 0px 0px 0px 3.5px rgba(23, 101, 144, 1);
  animation: showFocus .3s;
  border-color: rgb(122, 167, 221) !important;
}

@keyframes showFocus {
  0% {
    border-color: #dbdbdb;
    border-radius: -1px;
    box-shadow: 0px 0px 0px 14px rgba(23, 101, 144, 0);
    /*outline: 14px solid rgba(159, 204, 250, 0);
    outline-offset: 0px;*/
  }
  100% {
    border-color: rgb(122, 167, 221) !important;
    border-radius: 1px;
    box-shadow: 0px 0px 0px 3.5px rgba(23, 101, 144, 1);
    /*outline: 4px solid rgba(159, 204, 250, 1);
    outline-offset: -1px;*/
  }
}

input[type="search"] {
  text-indent: 0px;
  text-align: center;
  background-image: url('assets/search.svg');
  background-size: auto 50%, 100% 100%;
  background-position: 5px 6px, center;
  background-repeat: no-repeat;
  text-align: center;
  text-indent: 18px;
  height: 28px !important;
  border-radius: 5px !important;
}

input[type="search"]:focus {
  text-align: left;
  box-shadow: 0px 0px 0px 4.5px rgb(115, 166, 233);
  border-bottom-color: #f00 !important;
  border-radius: 4px !important;
}

input[type="search"]::-webkit-input-placeholder {
  /*text-align: center;*/
  color: rgb(152,152,152);
  font-weight: 400;
  letter-spacing: 0.2px;
}
</style>
