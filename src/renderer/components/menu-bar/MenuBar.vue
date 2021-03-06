<template>
  <v-container
    fluid
    class="py-0"
  >
    <v-row
      class="px-3"
      :class="hasUpdate ? 'update-available' : ''"
    >
      <v-col cols="auto">
        <SettingsMenu />
      </v-col>
      <v-spacer />
      <v-col cols="auto">
        <span
          v-if="!hasUpdate"
          class="d-inline-flex align-center fill-height mx-1"
        >No updates right now</span>
        <v-btn
          v-else-if="filteredModUpdates.length === 0 || !availableSMMUpdate"
          class="my-2 mx-1"
          @click="filteredModUpdates.length > 0 ? openModUpdatesDialog() : openSMMUpdateDialog()"
        >
          <span class="mx-1">
            {{ filteredModUpdates.length > 0 ? 'Mod updates are available' : 'SMM updates are available' }}
          </span>
        </v-btn>
        <v-menu v-else>
          <template v-slot:activator="{ on, attrs }">
            <v-btn
              class="my-2 mx-1"
              v-bind="attrs"
              v-on="on"
            >
              <span class="mx-1">
                SMM and mod updates are avaialble
              </span>
            </v-btn>
          </template>
          <v-card>
            <v-list class="menu">
              <v-list-item @click="openModUpdatesDialog">
                <v-list-item-action />
                <v-list-item-content>
                  <v-list-item-title>Mod updates ({{ filteredModUpdates.length }})</v-list-item-title>
                </v-list-item-content>
              </v-list-item>

              <v-divider
                inset
                class="custom"
              />

              <v-list-item @click="openSMMUpdateDialog">
                <v-list-item-action />
                <v-list-item-content>
                  <v-list-item-title>SMM updates ({{ smmUpdateCount }})</v-list-item-title>
                </v-list-item-content>
              </v-list-item>

              <v-divider
                inset
                class="custom"
              />
            </v-list>
          </v-card>
        </v-menu>
      </v-col>
      <v-col cols="auto">
        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-btn
              style="min-width: 36px"
              class="my-2 mx-1"
              v-bind="attrs"
              v-on="on"
              @click="manualCheckForUpdates"
            >
              <v-icon style="font-size: 25px !important">
                mdi-update
              </v-icon>
            </v-btn>
          </template>
          <span>Check for updates</span>
        </v-tooltip>
      </v-col>
      <v-col cols="auto">
        <UpdatesMenu
          :available-s-m-m-update="availableSMMUpdate"
          :filtered-mod-updates="filteredModUpdates"
          :show-ignored-updates.sync="showIgnoredUpdates"
          :update-check-mode.sync="updateCheckMode"
          @addUpdateListener="addUpdateListener"
          @openSMMUpdateDialog="openSMMUpdateDialog"
          @openModUpdatesDialog="openModUpdatesDialog"
        />
      </v-col>
    </v-row>
    <ModUpdatesDialog
      ref="modUpdatesDialog"
      :filtered-mod-updates="filteredModUpdates"
      :ignored-updates="ignoredUpdates"
      :is-ignored="isIgnored"
      @ignoreUpdate="ignoreUpdate"
      @unignoreUpdate="unignoreUpdate"
      @updateItem="updateItem"
      @updateAll="updateAll"
      @viewChangelog="viewChangelog"
    />
    <ChangelogDialog
      ref="changelogDialog"
      :view-changelog-update="viewChangelogUpdate"
    />
    <SMMUpdateDialog
      ref="smmUpdateDialog"
      :available-s-m-m-update="availableSMMUpdate"
      :smm-update-notes="smmUpdateNotes"
      @updateSMMNow="updateSMMNow"
    />
    <ProfileExportProgressDialog />
    <ProfileImportProgressDialog />
  </v-container>
</template>

<script>
import { mapState } from 'vuex';
import {
  ignoreUpdate, unignoreUpdate,
} from '@/utils';
import SettingsMenu from './SettingsMenu';
import UpdatesMenu from './UpdatesMenu';
import ProfileImportProgressDialog from './dialogs/ProfileImportProgressDialog';
import ProfileExportProgressDialog from './dialogs/ProfileExportProgressDialog';
import SMMUpdateDialog from './dialogs/SMMUpdateDialog';
import ChangelogDialog from './dialogs/ChangelogDialog';
import ModUpdatesDialog from './dialogs/ModUpdatesDialog';
import { getSetting, saveSetting } from '~/settings';

const UPDATE_CHECK_INTERVAL = 5 * 60 * 1000;

export default {
  components: {
    SettingsMenu,
    UpdatesMenu,
    ModUpdatesDialog,
    ChangelogDialog,
    SMMUpdateDialog,
    ProfileExportProgressDialog,
    ProfileImportProgressDialog,
  },
  data() {
    return {
      availableSMMUpdate: null,
      modUpdates: [],
      nextCheckForUpdates: -1,
      currentCheckForUpdates: -1,
      viewChangelogUpdate: null,
      showIgnoredUpdates: false,
      ignoredUpdates: [],
      cachedUpdateCheckMode: 'launch',
    };
  },
  computed: {
    ...mapState([
      'inProgress',
      'selectedInstall',
      'selectedProfile',
    ]),
    ...mapState({
      allMods: 'mods',
    }),
    smmUpdateNotes() {
      if (!this.availableSMMUpdate) {
        return '';
      }
      return this.availableSMMUpdate.releaseNotes.map((release) => release.note.substr(release.note.indexOf('<h2>Changelog</h2>')).replace('<h2>Changelog</h2>', `<h2>v${release.version} changelog</h2>`)).join('\n');
    },
    hasUpdate() {
      return !!this.availableSMMUpdate || this.filteredModUpdates.length > 0;
    },
    filteredModUpdates() {
      return (this.showIgnoredUpdates ? this.modUpdates : this.modUpdates.filter((update) => !this.isIgnored(update)));
    },
    updateCheckMode: {
      get() {
        return this.cachedUpdateCheckMode;
      },
      set(value) {
        saveSetting('updateCheckMode', value);
        this.cachedUpdateCheckMode = value;
      },
    },
    smmUpdateCount() {
      if (!this.availableSMMUpdate) {
        return 0;
      }
      return this.availableSMMUpdate.releaseNotes.length;
    },
  },
  watch: {
    async selectedInstall() {
      await this.checkForUpdates();
    },
    async selectedProfile() {
      await this.checkForUpdates();
    },
  },
  mounted() {
    this.ignoredUpdates = getSetting('ignoredUpdates', []);
    this.cachedUpdateCheckMode = getSetting('updateCheckMode', 'launch');

    if (this.updateCheckMode === 'launch') {
      this.$root.$once('doneLaunchUpdateCheck', () => {
        this.addUpdateListener();
      });
    } else {
      this.addUpdateListener();
    }
    this.nextCheckForUpdates = setTimeout(() => this.checkForUpdates(), UPDATE_CHECK_INTERVAL);
  },
  methods: {
    openSMMUpdateDialog() {
      this.$refs.smmUpdateDialog.smmUpdateDialog = true;
    },
    openModUpdatesDialog() {
      this.$refs.modUpdatesDialog.modUpdatesDialog = true;
    },
    addUpdateListener() {
      this.$electron.ipcRenderer.on('updateAvailable', (e, updateInfo) => {
        this.availableSMMUpdate = updateInfo;
        if (this.updateCheckMode === 'ask' || this.updateCheckmode === 'launch') {
          this.$refs.smmUpdateDialog.smmUpdateDialog = true;
        }
      });
    },
    async manualCheckForUpdates() {
      await this.checkForUpdates();
      if (this.filteredModUpdates.length > 0) {
        this.openModUpdatesDialog();
      }
    },
    async checkForUpdates() {
      this.currentCheckForUpdates += 1;
      const thisCheckForUpdates = this.currentCheckForUpdates;
      clearTimeout(this.nextCheckForUpdates);
      // don't check for updates while something is in progress
      while (this.inProgress.length > 0) {
        // eslint-disable-next-line no-await-in-loop
        await new Promise((res) => setTimeout(() => res(), 500));
        if (thisCheckForUpdates !== this.currentCheckForUpdates) {
          return;
        }
      }
      if (thisCheckForUpdates !== this.currentCheckForUpdates) {
        return;
      }
      this.$electron.ipcRenderer.send('checkForUpdates');
      const modUpdates = (await this.$store.state.selectedInstall.checkForUpdates()).map((update) => Object.assign(update, {
        name: this.allMods.find((mod) => mod.modInfo.mod_reference === update.item)?.modInfo.name || update.item,
      }));
      if (thisCheckForUpdates === this.currentCheckForUpdates) {
        this.modUpdates = modUpdates;
        this.nextCheckForUpdates = setTimeout(() => this.checkForUpdates(), UPDATE_CHECK_INTERVAL);
      }
    },
    updateSMMNow() {
      this.$root.$emit('downloadUpdate');
      this.refs.smmUpdateDialog.smmUpdateDialog = false;
    },
    async updateAll() {
      await this.$store.dispatch('updateMulti', this.filteredModUpdates);
      const currentUpdates = this.filteredModUpdates;
      this.modUpdates.removeWhere((update) => currentUpdates.includes(update));
      if (this.filteredModUpdates.length === 0) {
        this.$refs.modUpdatesDialog.modUpdatesDialog = false;
      }
    },
    async updateItem(update) {
      await this.$store.dispatch('updateSingle', update);
      this.modUpdates.remove(update);
      if (this.filteredModUpdates.length === 0) {
        this.$refs.modUpdatesDialog.modUpdatesDialog = false;
      }
    },
    ignoreUpdate(update) {
      this.ignoredUpdates = ignoreUpdate(update.item, update.version);
    },
    unignoreUpdate(update) {
      this.ignoredUpdates = unignoreUpdate(update.item, update.version);
    },
    isIgnored(update) {
      return this.ignoredUpdates.some((ignoredUpdate) => ignoredUpdate.item === update.item && ignoredUpdate.version === update.version);
    },
    viewChangelog(update) {
      this.viewChangelogUpdate = update;
      this.$refs.changelogDialog.changelogDialog = true;
    },
  },
};
</script>

<style>
.menu.v-list {
  background-color: var(--v-backgroundMenu-base) !important;
}
.menu .custom.v-list .v-list-item__action {
  margin: 0;
}
.menu .v-icon {
  font-size: 18px !important;
}
.menu .v-list-item {
  padding-left: 10px !important;
}
.menu .v-list-item__action:first-child {
  margin-right: 0px !important;
}
.menu .custom.v-divider--inset:not(.v-divider--vertical) {
  margin-left: 30px !important;
  max-width: calc(100% - 60px) !important;
}
.menu .custom.v-divider:not(.v-divider--inset):not(.v-divider--vertical) {
  margin-left: 10px !important;
  max-width: calc(100% - 40px) !important;
}
</style>

<style scoped>
.row.update-available, .row.update-available>* {
  background-color: var(--v-primary-base) !important;
  color: black !important;
}
div {
  background-color: var(--v-backgroundMenuBar-base);
}
.col {
  padding: 0;
  padding-top: 8px;
  padding-bottom: 8px;
}
</style>
