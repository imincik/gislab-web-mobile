<template>
  <div id="app" class="app f-col">
    <intro-page v-if="!projectName"/>
    <map-app v-if="projectStatus === 200" :key="projectKey"/>
    <login-dialog
      :value="showLogin"
      :login-required="projectStatus !== 200"
      :permission-denied="projectStatus === 403"
      :password-reset-url="app.reset_password_url"
      :project="project"
      @login="onLogin"
      @close="$store.commit('showLogin', false)"
    />
    <project-not-found v-if="projectStatus === 400 || projectStatus === 404"/>
    <server-error v-if="projectStatus === 500"/>
    <app-notifications v-if="notifications.length" :notifications="notifications"/>
    <transition name="modal">
      <div v-if="updateExists" class="notification f-col light">
        <div class="content p-2 f-col-ac shadow-2">
          <div class="msg my-2">
            <img src="@/assets/image_logo.svg">
            <translate class="text">A new version of the map application is available</translate>
          </div>
          <div class="actions f-row-ac mx-auto">
            <div class="f-grow"/>
            <v-btn class="small outlined round mx-4" color="#aaa" @click="updateExists = false">
              <translate>Not now</translate>
            </v-btn>
            <v-btn class="small round mx-4" color="green" @click="refreshApp">
              <translate translate-context="verb">Update</translate>
            </v-btn>
          </div>
        </div>
      </div>
    </transition>
    <transition name="modal">
      <div v-if="showInstallPrompt" class="notification f-col light">
        <div class="content p-2 f-col-ac shadow-2">
          <div class="msg my-2">
            <img src="@/assets/image_logo.svg">
            <translate class="text">Add to Home screen?</translate>
          </div>
          <translate class="desc m-2">
            It will give you the experience of a native app, with easy access and a standalone window without the browser's address bar and other controls.
          </translate>
          <div class="actions f-row-ac mx-auto">
            <v-btn class="small outlined round" color="#ddd" @click="dismissInstallPrompt">
              <translate>No</translate>
            </v-btn>
            <v-btn class="small round" color="green" @click="installApp">
              <translate>Yes</translate>
            </v-btn>
          </div>
        </div>
      </div>
    </transition>
    <popup-layer class="light"/>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import { set, get } from 'idb-keyval'

import IntroPage from '@/IntroPage.vue'
import ProjectNotFound from '@/ProjectNotFound.vue'
import DesktopMap from '@/components/Map.vue'
import MobileMap from '@/components/MobileMap.vue'
import LoginDialog from '@/components/LoginDialog.vue'
import PopupLayer from '@/ui/PopupLayer.vue'
import ServerError from './ServerError.vue'
import AppNotifications from './AppNotifications.vue'
import projectsHistory from '@/projects-history'
import Update from '@/mixins/update'
import { parseColor } from '@/ui/utils/colors'

export default {
  mixins: [ Update ],
  components: {
    PopupLayer,
    ProjectNotFound,
    LoginDialog,
    IntroPage,
    ServerError,
    AppNotifications,
    MapApp: async () => window.env.mobile ? MobileMap : DesktopMap
  },
  data () {
    return {
      projectKey: 0,
      showInstallPrompt: false
    }
  },
  computed: {
    ...mapState(['app', 'user', 'project', 'showLogin']),
    projectName () {
      return new URLSearchParams(location.search).get('PROJECT') || this.app.landing_project
    },
    projectStatus () {
      return this.project && this.project.config.status
    },
    isStandaloneApp () {
      return window.matchMedia('(display-mode: standalone)').matches
    },
    notifications () {
      return this.project?.config?.notifications ?? []
    }
  },
  created () {
    this.loadProject()
  },
  mounted () {
    if (window.env.mobile) {
      const setHeightStyle = () => {
        const vh = window.innerHeight / 100
        document.documentElement.style.setProperty('--vh', `${vh}px`)
      }
      window.addEventListener('resize', setHeightStyle)
      setHeightStyle()
      this.$once('hook:beforeDestroy', () => window.removeEventListener('resize', setHeightStyle))
      if (!this.isStandaloneApp) {
        const installHandler = async (e) => {
          e.preventDefault()
          let promptTimes = await get('pwa-install-prompts') || []
          if (promptTimes.length >= 3) {
            return
          }
          const now = parseInt(new Date().getTime() / 1000)
          const lastPrompt = promptTimes?.[0]
          const minInterval = 10 * 60 * 60 * 24 // 10 days
          if (lastPrompt && now - lastPrompt < minInterval) {
            // not enought time elapsed since last prompt
            return
          }
          this.deferredPrompt = e
          this.showInstallPrompt = true
          set('pwa-install-prompts', [now, ...promptTimes])
        }
        window.addEventListener('beforeinstallprompt', installHandler)
        this.$once('hook:beforeDestroy', () => window.removeEventListener('beforeinstallprompt', installHandler))
      }
    }
  },
  watch: {
    projectStatus: {
      immediate: true,
      handler (status) {
        if (status === 401 || status === 403) {
          this.$store.commit('showLogin', true)
        }
      }
    },
    project () {
      this.projectKey++
    }
  },
  methods: {
    async loadProject () {
      const data = await this.$http.project(this.projectName).catch(data => data)

      if (data.lang) {
        this.$language.current = data.lang.replace('-', '_')
        document.documentElement.setAttribute('lang', data.lang.split('-')[0])
      }

      this.$store.commit('project', data)
      if (data.status === 200) {
        projectsHistory.push(this.user, this.projectName)
      }
      const title = data.title || data.root_title
      if (title) {
        document.title = title
      }
      const themeColor = data.app?.theme_color
      if (themeColor) {
        document.documentElement.style.setProperty('--color-primary', themeColor)
        try {
          const rgba = parseColor(themeColor)
          document.documentElement.style.setProperty('--color-primary-rgb', rgba.slice(0, 3).join(','))
        } catch (err) {
          console.error(`Invalid theme color: ${themeColor}`)
        }
      }

    },
    onLogin (user) {
      this.$store.commit('user', user)
      this.$store.commit('showLogin', false)
      this.loadProject()
    },
    installApp () {
      this.deferredPrompt.prompt()
      this.showInstallPrompt = false
    },
    dismissInstallPrompt () {
      this.showInstallPrompt = false
    }
  }
}
</script>

<style lang="scss">
@import './theme.scss';
@import './transitions.scss';

html {
  overflow: auto;
  font-size: 16px;
  font-family: Roboto,sans-serif;
  line-height: 1.5;
  word-spacing: 1px;
  -ms-text-size-adjust: 100%;
  -webkit-text-size-adjust: 100%;
  -moz-osx-font-smoothing: grayscale;
  -webkit-font-smoothing: antialiased;
  -webkit-tap-highlight-color: transparent;
}
body {
  overscroll-behavior-x: none;
  touch-action: none;
}
</style>

<style lang="scss" scoped>
.notification {
  position: fixed;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.3);
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 6px;
  overflow: hidden;

  .content {
    background-color: var(--color-dark);
    color: #fff;
    min-width: 300px;
    max-width: 100%;
    overflow: hidden;
    border-radius: 6px;
    .msg {
      display: grid;
      grid-template-columns: auto 1fr 20px;
      line-height: 1.3;
      align-items: center;
      font-weight: 500;
      img {
        height: 40px;
      }
      .text {
        display: flex;
        text-align: center;
        padding-inline: 8px;
      }
    }
    .desc {
      font-size: 14px;
      max-width: 400px;
      text-align: center;
      opacity: 0.8;
    }
    .actions {
      .btn {
        min-width: 110px;
      }
    }
  }
}
</style>
