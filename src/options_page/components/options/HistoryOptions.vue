<template>
  <div class="option-section">
    <span class="option-card-title">{{ tl('History') }}</span>

    <v-card>
      <v-list two-line>
        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('Enable_save_visit_history') }}</v-list-tile-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-switch v-model="enableSaveVisitHistory"></v-switch>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('_enable_save_download_history') }}</v-list-tile-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-select
              :items="[{
                text: this.tl('_enable'),
                value: 1
              }, {
                text: this.tl('_show_in_work_only'),
                value: 2
              }, {
                text: this.tl('_disable'),
                value: 0
              }]"
              v-model="enableSaveDownloadHistory"
              style="width:150px;"
            ></v-select>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('Do_not_save_NSFW_work_to_history') }}</v-list-tile-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-switch v-model="notSaveNSFWWorkInHistory" :disabled="!enableSaveVisitHistory"></v-switch>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('_max_history_items') }}</v-list-tile-title>
            <v-list-tile-sub-title>{{ tl('_reload_to_apply_change') }}</v-list-tile-sub-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-text-field
              v-model="maxHistoryItems"
              reverse
              type="number"
              style="width:100px;"
            ></v-text-field>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('_max_download_records') }}</v-list-tile-title>
            <v-list-tile-sub-title>{{ tl('_reload_to_apply_change') }}</v-list-tile-sub-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-text-field
              v-model="maxDownloadRecords"
              reverse
              type="number"
              style="width:100px;"
            ></v-text-field>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('Export_visit_history') }}</v-list-tile-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-btn
              depressed
              @click="exportVisitHistory"
            >{{ tl('_export') }}</v-btn>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('Import_visit_history') }}</v-list-tile-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-btn
              depressed
              @click="importVisitHistory"
            >{{ tl('_import') }}</v-btn>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('_recovery_history') }} ({{ tl('_total_backup') }}: {{ historyBackupCount }})</v-list-tile-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-btn
              depressed
              @click="recoveryHistory"
            >{{ tl('_recovery') }}</v-btn>
          </v-list-tile-action>
        </v-list-tile>

        <v-list-tile>
          <v-list-tile-content>
            <v-list-tile-title>{{ tl('clear_history_data') }}</v-list-tile-title>
            <v-list-tile-sub-title>{{ tl('clear_history_data_cannot_be_reversed') }}</v-list-tile-sub-title>
          </v-list-tile-content>
          <v-list-tile-action>
            <v-btn
              depressed
              @click="confirmDialog = true"
            >{{ tl('Clear') }}</v-btn>
          </v-list-tile-action>
        </v-list-tile>
      </v-list>
    </v-card>

    <v-dialog v-model="confirmDialog" width="500">
      <v-card>
        <v-card-title>{{ tl('clear_history_data') }}</v-card-title>
        <v-card-text>
          <p style="font-size:14px;">{{ tl('_this_operation_cannot_be_reversed_are_you_sure') }}</p>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn @click="clearHistory" color="error">{{ tl('Clear') }}</v-btn>
          <v-btn @click="confirmDialog = false">{{ tl('Cancel') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog
      v-model="importing"
      width="250"
      persistent="true"
    >
      <v-card
        color="primary"
        dark
      >
        <v-card-text
          style="font-size:14px"
        >
          Import progress: {{ importCount - importItems.length }} / {{ importCount }}
          <v-progress-linear
            v-model="importProgress"
            color="white"
          ></v-progress-linear>
        </v-card-text>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import VisitHistoryPort from '@/modules/Ports/IllustHistoryPort/RendererPort';
import IllustHistory from "@/repositories/IllustHistory";

export default {
  data() {
    return {
      enableSaveVisitHistory: true,
      enableSaveDownloadHistory: true,
      notSaveNSFWWorkInHistory: false,
      confirmDialog: false,
      importing: false,
      exporting: false,
      importItems: [],
      importCount: 0,
      maxHistoryItems: 10000,
      maxDownloadRecords: 10000
    };
  },

  created() {
    this.illustHistory = new IllustHistory();
    this.visitHistoryPort = VisitHistoryPort.getInstance();
    this.visitHistoryPort.port.onMessage.addListener(this.handleVisitHistoryPortResponse);
  },

  beforeMount() {
    this.enableSaveVisitHistory = this.browserItems.enableSaveVisitHistory;
    this.enableSaveDownloadHistory = this.browserItems.enableSaveDownloadHistory;
    this.notSaveNSFWWorkInHistory = this.browserItems.notSaveNSFWWorkInHistory;
    this.maxHistoryItems = this.browserItems.maxHistoryItems;
    this.maxDownloadRecords = this.browserItems.maxDownloadRecords;
  },

  computed: {
    historyBackupCount() {
      return this.browserItems.historyBackup ? this.browserItems.historyBackup.length : 0;
    },

    importProgress() {
      return Math.floor((this.importCount - this.importItems.length) / this.importCount * 100);
    }
  },

  watch: {
    enableSaveVisitHistory(val) {
      browser.storage.local.set({
        enableSaveVisitHistory: !!val
      });
    },

    enableSaveDownloadHistory(val) {
      console.log(val);
      browser.storage.local.set({
        enableSaveDownloadHistory: val
      });
    },

    notSaveNSFWWorkInHistory(val) {
      browser.storage.local.set({
        notSaveNSFWWorkInHistory: !!val
      });
    },

    maxHistoryItems(val) {
      if (/^\d+$/.test(val) && val > 0) {
        browser.storage.local.set({
          maxHistoryItems: val
        });
      } else {
        this.maxHistoryItems = 10000;
      }
    },

    maxDownloadRecords(val) {
      if (/^\d+$/.test(val) && val > 0) {
        browser.storage.local.set({
          maxDownloadRecords: val
        });
      } else {
        this.maxDownloadRecords = 10000;
      }
    }
  },

  methods: {
    handleVisitHistoryPortResponse(message, port) {
      if (this.visitHistoryPort.isChannel(message.channel, 'save-batch-histories')) {
        if (this.importItems.length > 0) {
          this.importVisitHistoryData();
        } else {
          this.importing = false;
        }
      }
    },

    clearHistory() {
      this.visitHistoryPort.clearHistory();
      this.confirmDialog = false;
    },

    exportVisitHistory() {
      if (this.exporting || this.importing) {
        return
      }

      this.exporting = true

      this.illustHistory.getIllusts({
        limit: null
      }).then(docs => {
        let blob = new Blob([JSON.stringify(docs)], {type: 'application/json'})

        let a = document.createElement('a')
        a.href = URL.createObjectURL(blob)
        a.download = 'illust_histories-' + Date.now() + '.json'
        document.body.appendChild(a);
        a.click()
        a.remove();
        this.exporting = false
      })
    },

    importVisitHistoryData() {
      let items = [];

      while (items.length < 100) {
        if (!!this.importItems[0]) {
          items.push(this.importItems[0]);
          this.importItems.splice(0, 1);
        } else {
          break;
        }
      }

      if (items.length > 0) {
        this.visitHistoryPort.saveBatchHistories({
          items
        });
      }
    },

    importVisitHistory() {
      if (this.exporting || this.importing) {
        return
      }

      let input = document.createElement('input')
      input.type = 'file'
      input.addEventListener('change', (e) => {
        const files = e.target.files
        let fileReader = new FileReader()

        fileReader.addEventListener('load', () => {
          try {
            this.importItems = JSON.parse(fileReader.result);
            this.importCount = this.importItems.length;
            this.importing = true;
            this.importVisitHistoryData();
          } catch (e) {
            this.importing = false;
            alert('Invalid import file');
          }
        });

        fileReader.addEventListener('abort', () => {
          this.importing = false;
        });

        fileReader.readAsText(files[0])
      })

      if (window.confirm(this.tl('_import_history_will_take_some_time_determine_by_how_many_history_need_to_be_imported_are_you_sure'))) {
        input.click()
      }
    },

    recoveryHistory() {
      if (!window.confirm(this.tl('_recovery_history_will_take_some_time_determine_by_how_many_history_need_to_be_recovery_are_you_sure'))) {
        return;
      }

      this.importItems = this.browserItems.historyBackup;
      this.importCount = this.importItems.length;
      this.importing = true;
      this.importVisitHistoryData();
    }
  }
};
</script>
