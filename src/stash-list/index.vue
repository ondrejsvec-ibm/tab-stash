<template>
  <main
    :class="{'stash-list': true, 'selection-active': selection_active}"
    tabindex="0"
    @click="deselectAll"
    @keydown.esc.prevent.stop="onEscape"
  >
    <transition-group
      tag="aside"
      class="notification-overlay"
      appear
      name="notification"
    >
      <OopsNotification key="oops" v-if="showCrashReport" />
      <Notification
        key="new-features"
        v-if="recently_updated === 'features'"
        @activate="go('whats-new.html')"
        @dismiss="hideWhatsNew"
      >
        Tab Stash {{ my_version }} includes small quality-of-life improvements.
        See what's new!
      </Notification>
      <Notification
        key="new-fixes"
        v-if="recently_updated === 'fixes'"
        @activate="go('whats-new.html')"
        @dismiss="hideWhatsNew"
      >
        Tab Stash {{ my_version }} includes small quality-of-life improvements.
        See what's new!
      </Notification>
      <Notification
        key="stash-root-warning"
        v-if="stash_root_warning"
        @activate="stash_root_warning!.help"
      >
        {{ stash_root_warning.text }}
      </Notification>
      <Notification
        key="recently-deleted"
        v-if="recently_deleted !== 0"
        @activate="onDeleteNotifActivated"
      >
        <span v-if="typeof recently_deleted === 'object'">
          Deleted "{{ recentlyDeletedTitle }}". Undo?
        </span>
        <span v-else>
          Deleted {{ recently_deleted }} items. Show what was deleted?
        </span>
      </Notification>
    </transition-group>

    <header class="page action-container" @click.stop="">
      <Menu
        v-if="view !== 'popup'"
        class="menu main-menu"
        summaryClass="action mainmenu"
      >
        <button @click.prevent="showOptions"><span>Options...</span></button>
        <hr />
        <button @click.prevent="dialog = {class: 'ImportDialog'}">
          <span>Import...</span>
        </button>
        <button @click.prevent="showExportDialog">
          <span>Export...</span>
        </button>
        <hr />
        <a tabindex="0" :href="pageref('deleted-items.html')"
          ><span>Deleted Items...</span></a
        >
        <button @click.prevent="fetchMissingFavicons">
          <span>Fetch Missing Icons</span>
        </button>
        <hr />
        <a tabindex="0" href="https://josh-berry.github.io/tab-stash/tips.html"
          ><span>Tips and Tricks</span></a
        >
        <a tabindex="0" href="https://github.com/josh-berry/tab-stash/wiki"
          ><span>Wiki</span></a
        >
        <a
          tabindex="0"
          href="https://josh-berry.github.io/tab-stash/support.html"
          ><span>Help and Support</span></a
        >
        <a tabindex="0" :href="pageref('whats-new.html')"
          ><span>What's New?</span></a
        >
      </Menu>

      <a
        v-else
        class="action icon icon-pop-out"
        title="Show stashed tabs in a tab"
        @click.prevent="showTabUI"
      />

      <SelectionMenu v-if="selection_active" />

      <search-input
        ref="search"
        :tooltip="searchTooltip"
        :placeholder="search_placeholder"
        v-model="searchText"
      />
      <a
        :class="{action: true, collapse: !collapsed, expand: collapsed}"
        title="Hide all tabs so only group names are showing"
        @click.prevent.stop="collapseAll"
      />
    </header>

    <ul v-if="targetWindow" class="forest one-column">
      <li>
        <window :target-window="targetWindow" :metadata="curWindowMetadata" />
      </li>
    </ul>

    <folder-list ref="stashed" v-if="stashRoot" :parentFolder="stashRoot" />

    <footer class="page status-text">
      Tab Stash {{ my_version }} &mdash;
      <a :href="pageref('whats-new.html')">What's New</a>
    </footer>
  </main>

  <transition appear name="dialog">
    <component
      v-if="dialog"
      :is="dialog.class"
      v-bind="dialog.props"
      @close="dialog = undefined"
    />
  </transition>
</template>

<script lang="ts">
import {defineComponent} from "vue";
import browser from "webextension-polyfill";

import the from "@/globals-ui";
import {pageref} from "../launch-vue";
import {
  CUR_WINDOW_MD_ID,
  type BookmarkMetadataEntry,
} from "../model/bookmark-metadata";
import {
  friendlyFolderName,
  type Folder,
  type FolderStats,
} from "../model/bookmarks";
import type {Tab} from "../model/tabs";
import {fetchInfoForSites} from "../tasks/siteinfo";
import {TaskMonitor, parseVersion} from "../util";

import Menu from "../components/menu.vue";
import Notification from "../components/notification.vue";
import OopsNotification from "../components/oops-notification.vue";
import ProgressDialog from "../components/progress-dialog.vue";
import SearchInput from "../components/search-input.vue";
import ExportDialog from "../tasks/export.vue";
import ImportDialog from "../tasks/import.vue";
import FolderList from "./folder-list.vue";
import FolderVue from "./folder.vue";
import SelectionMenu from "./selection-menu.vue";
import WindowVue from "./window.vue";

export default defineComponent({
  components: {
    ExportDialog,
    Folder: FolderVue,
    FolderList,
    ImportDialog,
    Menu,
    Notification,
    OopsNotification,
    ProgressDialog,
    SearchInput,
    SelectionMenu,
    Window: WindowVue,
  },

  data: () => ({
    collapsed: false,
    searchText: "",
    dialog: undefined as undefined | {class: string; props?: any},
  }),

  computed: {
    my_version(): string {
      return the.version;
    },

    view(): string {
      return document.documentElement.dataset!.view ?? "tab";
    },

    stash_root_warning(): {text: string; help: () => void} | undefined {
      return the.model.bookmarks.stash_root_warning.value;
    },
    targetWindow() {
      return the.model.tabs.targetWindow.value;
    },
    tabs(): readonly Tab[] {
      const m = the.model.tabs;
      if (m.targetWindow.value === undefined) return [];
      return m.targetWindow.value.children;
    },
    stashRoot(): Folder | undefined {
      return the.model.bookmarks.stash_root.value;
    },

    recently_updated(): undefined | "features" | "fixes" {
      const last_notified = the.model.options.local.state.last_notified_version;
      if (last_notified === this.my_version) return undefined;

      const my = parseVersion(this.my_version);
      const last = last_notified ? parseVersion(last_notified) : [];

      if (my[0] == last[0] && my[1] == last[1]) return "fixes";
      return "features";
    },

    recently_deleted() {
      return the.model.deleted_items.state.recentlyDeleted;
    },
    recentlyDeletedTitle() {
      if (typeof this.recently_deleted === "object") {
        return friendlyFolderName(this.recently_deleted.item.title);
      }
      return undefined;
    },

    selection_active(): boolean {
      return the.model.selection.selectedCount.value > 0;
    },

    counts(): FolderStats {
      const stats = this.stashRoot?.$recursiveStats;
      if (!stats) return {bookmarkCount: 0, folderCount: 0};
      return stats;
    },

    search_placeholder(): string {
      const counts = this.counts;
      const groups = counts.folderCount == 1 ? "group" : "groups";
      const tabs = counts.bookmarkCount == 1 ? "tab" : "tabs";
      return `Search ${counts.folderCount} ${groups}, ${counts.bookmarkCount} ${tabs}`;
    },

    tabStats(): {open: number; discarded: number; hidden: number} {
      let open = 0,
        discarded = 0,
        hidden = 0;
      for (const tab of this.tabs) {
        if (tab.position?.parent !== this.targetWindow) continue;
        if (tab.hidden) {
          hidden += 1;
        } else if (tab.discarded) {
          discarded += 1;
        } else {
          open += 1;
        }
      }
      return {open, discarded, hidden};
    },

    searchTooltip(): string {
      const st = this.tabStats;
      const tabs_sum = st.open + st.discarded + st.hidden;
      return `${this.counts.folderCount} group${this.plural(
        this.counts.folderCount,
      )}, ${this.counts.bookmarkCount} stashed tab${this.plural(
        this.counts.bookmarkCount,
      )}\n${tabs_sum} tab${this.plural(tabs_sum)} in this window (${
        st.open
      } open, ${st.discarded} unloaded, ${st.hidden} hidden)`;
    },

    curWindowMetadata(): BookmarkMetadataEntry {
      return the.model.bookmark_metadata.get(CUR_WINDOW_MD_ID);
    },

    showCrashReport(): boolean {
      return the.model.options.showCrashReport.value;
    },
  },

  mounted() {
    if (document.documentElement.dataset.view === "popup") {
      (<any>this.$refs.search).focus();
    }

    // The following block of code is just to help me test out progress
    // dialogs (since they only appear for a limited time when things are
    // happening):
    /*
    const self = this;
    this.dialog = {
      class: "ProgressDialog",
      props: {
        cancel() {
          self.dialog = undefined;
        },
        progress: {
          status: "Top",
          max: 100,
          value: 50,
          children: [
            {
              status: "Child 1",
              max: 10,
              value: 5,
              children: [
                {status: "Grandchild A", max: 5, value: 1, children: []},
                {status: "Grandchild B", max: 5, value: 2, children: []},
                {status: "Grandchild C", max: 5, value: 5, children: []},
              ],
            },
            {status: "Child 2", max: 10, value: 2, children: []},
            {status: "Child 3", max: 10, value: 0, children: []},
          ],
        },
      },
    };
    */
  },

  watch: {
    searchText() {
      the.model.searchText.value = this.searchText;
    },
  },

  methods: {
    pageref,

    collapseAll() {
      this.collapsed = !this.collapsed;
      const metadata = the.model.bookmark_metadata;
      metadata.setCollapsed(CUR_WINDOW_MD_ID, this.collapsed);
      for (const f of this.stashRoot?.children || []) {
        metadata.setCollapsed(f.id, this.collapsed);
      }
    },

    onDeleteNotifActivated() {
      if (typeof this.recently_deleted === "object") {
        the.model.undelete(this.recently_deleted);
      } else {
        this.go("deleted-items.html");
      }
    },

    onEscape() {
      if (this.searchText) {
        this.searchText = "";
        return;
      }
      this.deselectAll();
    },

    deselectAll() {
      the.model.selection.clearSelection();
    },

    showOptions() {
      browser.runtime.openOptionsPage().catch(console.log);
    },

    showTabUI() {
      the.model.attempt(async () => {
        await the.model.restoreTabs(
          [
            {
              title: "Tab Stash",
              url: browser.runtime.getURL("stash-list.html"),
            },
          ],
          {},
        );
        window.close();
      });
    },

    go(page: string) {
      window.location.href = pageref(page);
    },
    hideWhatsNew() {
      the.model.options.local.set({last_notified_version: this.my_version});
    },

    showExportDialog() {
      this.dialog = {class: "ExportDialog", props: {}};
    },

    plural(n: number): string {
      return n == 1 ? "" : "s";
    },

    async fetchMissingFavicons() {
      the.model.attempt(async () => {
        const favicons = the.model.favicons;
        const urls = the.model.bookmarks.urlsInStash();

        // This is just an async filter :/
        for (const url of urls) {
          const favicon = favicons.get(url);
          if (favicon?.value?.favIconUrl) urls.delete(url);
        }

        const iter = TaskMonitor.run_iter(tm => fetchInfoForSites(urls, tm));
        this.dialog = {
          class: "ProgressDialog",
          props: {progress: iter.progress, cancel: () => iter.cancel()},
        };

        try {
          for await (const info of iter) {
            favicons.set(info.originalUrl, {
              favIconUrl: info.favIconUrl ?? null,
              title: info.title,
            });
          }
        } finally {
          this.dialog = undefined;
        }
      });
    },
  },
});
</script>
