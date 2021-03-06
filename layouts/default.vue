<template>
  <v-app ref="app">
    <backdrop />
    <v-navigation-drawer
      v-if="$store.state.page.showNavDrawer"
      v-model="drawer"
      :temporary="$vuetify.breakpoint.mobile"
      :permanent="!$vuetify.breakpoint.mobile"
      app
    >
      <template #prepend>
        <user-button />
        <v-divider />
      </template>
      <v-list>
        <v-list-item
          v-for="item in items"
          :key="item.Id"
          :to="item.to"
          router
          exact
        >
          <v-list-item-action>
            <v-icon>{{ item.icon }}</v-icon>
          </v-list-item-action>
          <v-list-item-content>
            <v-list-item-title v-text="item.title" />
          </v-list-item-content>
        </v-list-item>
        <v-subheader>{{ $t('libraries') }}</v-subheader>
        <v-list-item
          v-for="library in libraryItems"
          :key="library.Id"
          :to="library.to"
          router
          exact
        >
          <v-list-item-action>
            <v-icon>{{ library.icon }}</v-icon>
          </v-list-item-action>
          <v-list-item-content>
            <v-list-item-title v-text="library.title" />
          </v-list-item-content>
        </v-list-item>
      </v-list>
      <template #append>
        <connection-monitor class="ml-auto" />
      </template>
    </v-navigation-drawer>
    <v-app-bar
      :clipped-left="$vuetify.breakpoint.mobile"
      class="pl-2 pr-2"
      flat
      app
      :class="{ opaque: opaqueAppBar || $vuetify.breakpoint.xsOnly }"
    >
      <v-app-bar-nav-icon
        v-if="$vuetify.breakpoint.mobile && $store.state.page.showNavDrawer"
        @click.stop="drawer = !drawer"
      />
      <v-btn
        :icon="opaqueAppBar || $vuetify.breakpoint.xsOnly || isScrolled"
        :fab="!(opaqueAppBar || $vuetify.breakpoint.xsOnly) && !isScrolled"
        :small="!(opaqueAppBar || $vuetify.breakpoint.xsOnly) && !isScrolled"
        :class="{
          'ml-n1': opaqueAppBar || $vuetify.breakpoint.xsOnly || isScrolled,
          'mr-2': !(opaqueAppBar || $vuetify.breakpoint.xsOnly) && !isScrolled,
          'mr-1': opaqueAppBar || $vuetify.breakpoint.xsOnly || isScrolled,
          hidden: $route.name === 'index'
        }"
        @click="$router.back()"
      >
        <v-icon>mdi-arrow-left</v-icon>
      </v-btn>
      <v-text-field
        class="search-input"
        :class="$vuetify.breakpoint.mdAndUp ? 'expandable' : null"
        prepend-inner-icon="mdi-magnify"
        :placeholder="$t('search')"
        max-width="15em"
        dense
        outlined
        filled
        flat
        hide-details
        single-line
      />
      <v-spacer />
      <dark-mode-toggle
        :fab="!(opaqueAppBar || $vuetify.breakpoint.xsOnly) && !isScrolled"
      />
      <locale-switcher
        :fab="!(opaqueAppBar || $vuetify.breakpoint.xsOnly) && !isScrolled"
        bottom
      />
    </v-app-bar>
    <v-main>
      <nuxt />
    </v-main>
    <audio-controls />
    <!-- Utilities and global systems -->
    <snackbar />
    <player-manager />
  </v-app>
</template>

<script lang="ts">
import { BaseItemDto } from '@jellyfin/client-axios';
import { stringify } from 'qs';
import Vue from 'vue';
import { mapActions, mapState, MutationPayload } from 'vuex';
import { AppState } from '~/store';
import { getLibraryIcon } from '~/utils/items';

interface WebSocketMessage {
  MessageType: string;
  Data?: Record<string, never>;
}

interface LayoutButton {
  icon: string;
  title: string;
  to: string;
}

export default Vue.extend({
  data() {
    return {
      isScrolled: false,
      drawer: false,
      opacity: 0,
      keepAliveInterval: undefined as number | undefined
    };
  },
  computed: {
    ...mapState<AppState>({
      opaqueAppBar: (state: AppState) => state.page.opaqueAppBar,
      libraryItems: (state: AppState) =>
        state.userViews.views.map((view: BaseItemDto) => {
          return {
            icon: getLibraryIcon(view.CollectionType),
            title: view.Name,
            to: `/library/${view.Id}`
          };
        })
    }),
    items(): LayoutButton[] {
      return [
        {
          icon: 'mdi-home',
          title: this.$t('home'),
          to: '/'
        }
      ];
    }
  },
  watch: {
    $route(to): void {
      if (to.fullPath.includes('fullscreen')) {
        this.showNavDrawer({ showNavDrawer: false });
      } else if (!this.$store.state.page.showNavDrawer) {
        this.showNavDrawer({ showNavDrawer: true });
      }
    }
  },
  beforeMount() {
    this.callAllCallbacks();
    this.refreshUserViews();

    const socketParams = stringify({
      api_key: this.$store.state.user.accessToken,
      deviceId: this.$store.state.deviceProfile.deviceId
    });
    let socketUrl = `${this.$axios.defaults.baseURL}/socket?${socketParams}`;
    socketUrl = socketUrl.replace('https:', 'wss:');
    socketUrl = socketUrl.replace('http:', 'ws:');

    this.$connect(socketUrl);
    this.handleKeepAlive();
  },
  mounted() {
    window.addEventListener('scroll', this.setIsScrolled, { passive: true });
  },
  beforeDestroy() {
    if (this.keepAliveInterval) {
      clearInterval(this.keepAliveInterval);
    }
  },
  destroyed() {
    window.removeEventListener('scroll', this.setIsScrolled);
  },
  methods: {
    ...mapActions('userViews', ['refreshUserViews']),
    ...mapActions('displayPreferences', ['callAllCallbacks']),
    ...mapActions('page', ['showNavDrawer']),
    handleKeepAlive(): void {
      this.$store.subscribe(
        (mutation: MutationPayload, state: AppState): void => {
          if (
            mutation.type === 'SOCKET_ONMESSAGE' &&
            state.socket.message.MessageType === 'ForceKeepAlive'
          ) {
            this.sendWebSocketMessage('KeepAlive');
            this.keepAliveInterval = window.setInterval(() => {
              this.sendWebSocketMessage('KeepAlive');
            }, state.socket.message.Data * 1000 * 0.5);
          }
        }
      );
    },
    setIsScrolled(): void {
      // Set it slightly higher than needed, so the transition of the app bar syncs with the button transition
      this.isScrolled = window.scrollY > 10;
    },
    sendWebSocketMessage(name: string, data?: Record<string, never>): void {
      const msg: WebSocketMessage = {
        MessageType: name,
        ...(data ? { Data: data } : {})
      };

      this.$store.state.socket.instance.send(JSON.stringify(msg));
    }
  }
});
</script>

<style lang="scss" scoped>
.v-app-bar:not(.v-app-bar--is-scrolled):not(.opaque) {
  background-color: transparent !important;
}

.v-app-bar .v-app-bar--is-scrolled:not(.opaque) {
  padding-top: 0;
  padding-bottom: 0;
  background-color: rgba(32, 32, 32, 1) !important;
}

.search-input {
  max-width: 15em;
  transition: max-width 0.25s;
}

.search-input.expandable.primary--text {
  max-width: 40em;
  transition: max-width 0.25s;
}

.hidden {
  visibility: hidden;
}
</style>
