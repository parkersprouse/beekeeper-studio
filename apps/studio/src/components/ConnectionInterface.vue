<template>
  <div class="interface connection-interface">
    <div class="interface-wrap row">
      <sidebar
        class="connection-sidebar"
        ref="sidebar"
        v-show="sidebarShown"
      >
        <connection-sidebar
          :selected-config="config"
          @remove="remove"
          @duplicate="duplicate"
          @edit="edit"
          @connect="handleConnect"
          @create="create"
        />
      </sidebar>
      <div
        ref="content"
        class="connection-main page-content flex-col"
        id="page-content"
      >
        <div class="small-wrap expand">
          <div
            v-if="!show_new_connection"
            id="placeholder-card"
          >
            <div>Select Connection</div>
            <div class="separator">or</div>
            <a
              href=""
              class="btn btn-flat btn-icon btn-block"
              @click.prevent="create"
            >
              <i class="material-icons">add</i>
              Add Connection
            </a>
          </div>

          <div
            v-else
            id="create-connection-card"
            class="card-flat padding"
            :class="determineLabelColor"
          >
            <a
              href=""
              @click.prevent="show_new_connection = false"
              id="create-connection-card__close"
            >
              <i class="material-icons close">close</i>
            </a>

            <div class="flex flex-between">
              <h3
                class="card-title"
                v-if="!pageTitle"
              >
                New Connection
              </h3>
              <h3
                class="card-title"
                v-if="pageTitle"
              >
                {{ pageTitle }}
              </h3>
              <ImportButton :config="config">
                Import from URL
              </ImportButton>
            </div>
            <error-alert
              :error="errors"
              title="Please fix the following errors"
            />
            <form
              @action="submit"
              v-if="config"
            >
              <div class="form-group">
                <label for="connectionType">Connection Type</label>
                <select
                  name="connectionType"
                  class="form-control custom-select"
                  v-model="config.connectionType"
                  id="connection-select"
                >
                  <option
                    disabled
                    hidden
                    value="null"
                  >
                    Select a connection type...
                  </option>
                  <option
                    :key="`${t.value}-${t.name}`"
                    v-for="t in connectionTypes"
                    :value="t.value"
                  >
                    {{ t.name }}
                  </option>
                </select>
              </div>
              <div v-if="config.connectionType">
                <!-- INDIVIDUAL DB CONFIGS -->
                <postgres-form
                  v-if="config.connectionType === 'cockroachdb'"
                  :config="config"
                  :testing="testing"
                />
                <mysql-form
                  v-if="['mysql', 'mariadb'].includes(config.connectionType)"
                  :config="config"
                  :testing="testing"
                  @save="save"
                  @test="testConnection"
                  @connect="submit"
                />
                <postgres-form
                  v-if="config.connectionType === 'postgresql'"
                  :config="config"
                  :testing="testing"
                />
                <redshift-form
                  v-if="config.connectionType === 'redshift'"
                  :config="config"
                  :testing="testing"
                />
                <sqlite-form
                  v-if="config.connectionType === 'sqlite'"
                  :config="config"
                  :testing="testing"
                />
                <sql-server-form
                  v-if="config.connectionType === 'sqlserver'"
                  :config="config"
                  :testing="testing"
                />
                <big-query-form
                  v-if="config.connectionType === 'bigquery'"
                  :config="config"
                  :testing="testing"
                />
                <firebird-form
                  v-if="config.connectionType === 'firebird'"
                  :config="config"
                  :testing="testing"
                />
                <other-database-notice v-if="config.connectionType === 'other'" />

                <!-- TEST AND CONNECT -->
                <div
                  v-if="config.connectionType !== 'other'"
                  class="test-connect row flex-middle"
                >
                  <span class="expand" />
                  <div class="btn-group">
                    <button
                      :disabled="testing"
                      class="btn btn-flat"
                      type="button"
                      @click.prevent="testConnection"
                    >
                      Test
                    </button>
                    <button
                      :disabled="testing"
                      class="btn btn-primary"
                      type="submit"
                      @click.prevent="submit"
                    >
                      Connect
                    </button>
                  </div>
                </div>
                <div
                  class="row"
                  v-if="connectionError"
                >
                  <div class="col">
                    <error-alert
                      :error="connectionError"
                      :help-text="errorHelp"
                      @close="connectionError = null"
                      :closable="true"
                    />
                  </div>
                </div>
                <SaveConnectionForm
                  v-if="config.connectionType !== 'other'"
                  :config="config"
                  @save="save"
                />
              </div>
            </form>
          </div>
          <div
            class="pitch"
            v-if="!config.connectionType"
            style='display: none !important;'
          >
            🌟 <strong>Upgrade to premium</strong> for data import, multi-table export, backup & restore, Oracle support, and more.
            <a
              href="https://docs.beekeeperstudio.io/docs/upgrading-from-the-community-edition"
              class=""
            >Upgrade Now</a>.
          </div>
        </div>

        <small class="app-version"><a href="https://www.beekeeperstudio.io/releases/latest">Beekeeper Studio {{ version
        }}</a></small>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import os from 'os'
import { SavedConnection } from '../common/appdb/models/saved_connection'
import ConnectionSidebar from './sidebar/ConnectionSidebar.vue'
import MysqlForm from './connection/MysqlForm.vue'
import PostgresForm from './connection/PostgresForm.vue'
import RedshiftForm from './connection/RedshiftForm.vue'
import Sidebar from './common/Sidebar.vue'
import SqliteForm from './connection/SqliteForm.vue'
import SqlServerForm from './connection/SqlServerForm.vue'
import SaveConnectionForm from './connection/SaveConnectionForm.vue'
import BigQueryForm from './connection/BigQueryForm.vue'
import FirebirdForm from './connection/FirebirdForm.vue'
import Split from 'split.js'
import ImportButton from './connection/ImportButton.vue'
import _ from 'lodash'
import platformInfo from '@/common/platform_info'
import ErrorAlert from './common/ErrorAlert.vue'
import rawLog from 'electron-log'
import { mapState } from 'vuex'
import { dialectFor } from '@shared/lib/dialects/models'
import { findClient } from '@/lib/db/clients'
import OtherDatabaseNotice from './connection/OtherDatabaseNotice.vue'
import Vue from 'vue'
import { AppEvent } from '@/common/AppEvent'

const log = rawLog.scope('ConnectionInterface')
// import ImportUrlForm from './connection/ImportUrlForm';

export default Vue.extend({
  components: { ConnectionSidebar, MysqlForm, PostgresForm, RedshiftForm, Sidebar, SqliteForm, SqlServerForm, SaveConnectionForm, ImportButton, ErrorAlert, OtherDatabaseNotice, BigQueryForm, FirebirdForm },

  data() {
    return {
      config: new SavedConnection(),
      connectionError: null,
      errorHelp: null,
      errors: null,
      show_new_connection: false,
      importError: null,
      sidebarShown: true,
      split: null,
      testing: false,
      url: null,
      version: platformInfo.appVersion,
    }
  },
  computed: {
    ...mapState(['workspaceId']),
    ...mapState('data/connections', { 'connections': 'items' }),
    connectionTypes() {
      return this.$config.defaults.connectionTypes
    },
    pageTitle() {
      if (_.isNull(this.config) || _.isUndefined(this.config.id)) {
        return "New Connection"
      } else {
        return this.config.name
      }
    },
    dialect() {
      return dialectFor(this.config.connectionType)
    },
    determineLabelColor() {
      return this.config.labelColor == "default" ? '' : `connection-label-color-${this.config.labelColor}`
    },
    rootBindings() {
      return [
        { event: AppEvent.dropzoneDrop, handler: this.maybeLoadSqlite },
      ]
    },
  },
  watch: {
    workspaceId() {
      this.config = new SavedConnection()
    },
    config: {
      deep: true,
      handler() {
        this.connectionError = null
      }
    },
    'config.connectionType'(newConnectionType) {
      if (!findClient(newConnectionType)?.supportsSocketPath) {
        this.config.socketPathEnabled = false
      }
    },
    connectionError() {
      console.log("error watch", this.connectionError, this.dialect)
      if (this.connectionError &&
        this.dialect == 'sqlserver' &&
        this.connectionError.message &&
        this.connectionError.message.includes('self signed certificate')
      ) {
        this.errorHelp = `You might need to check 'Trust Server Certificate'`
      } else {
        this.errorHelp = null
      }
    }
  },
  async mounted() {
    if (!this.$store.getters.workspace) {
      await this.$store.commit('workspace', this.$store.state.localWorkspace)
    }
    await this.$store.dispatch('loadUsedConfigs')
    await this.$store.dispatch('pinnedConnections/loadPins')
    await this.$store.dispatch('pinnedConnections/reorder')
    this.config.sshUsername = os.userInfo().username
    this.$nextTick(() => {
      const components = [
        this.$refs.sidebar.$refs.sidebar,
        this.$refs.content
      ]
      /* https://github.com/nathancahill/split/tree/master/packages/splitjs#options */
      this.split = Split(components, {
        elementStyle: (_dimension, size) => ({
          'flex-basis': `calc(${size}%)`,
        }),
        // sizes: [300, 500], // these are supposed to be percentages
        gutterize: 8,
        minSize: [300, 300],
        expandToMin: true,
      } as Split.Options)

      this.split.collapse(0);
    })
    this.registerHandlers(this.rootBindings)
  },
  beforeDestroy() {
    if (this.split) {
      this.split.destroy()
    }
    this.unregisterHandlers(this.rootBindings)
  },
  methods: {
    maybeLoadSqlite({ files }) {
      // cast to an array
      if (!files || !files.length) return
      if (!this.config) return;
      // we only load the first
      const file = files[0]
      const allGood = this.config.parse(file.path)
      if (!allGood) {
        this.$noty.error(`Unable to open '${file.name}'. It is not a valid SQLite file.`);
        return
      } else {
        this.submit()
      }
    },
    create() {
      this.show_new_connection = true;
      this.config = new SavedConnection();
    },
    edit(config) {
      this.config = config
      this.errors = null
      this.connectionError = null
    },
    async remove(config) {
      if (this.config === config) {
        this.config = new SavedConnection()
      }
      await this.$store.dispatch('pinnedConnections/remove', config)
      await this.$store.dispatch('data/connections/remove', config)
      this.$noty.success(`${config.name} deleted`)
    },
    async duplicate(config) {
      // Duplicates ES 6 class of the connection, without any reference to the old one.
      const duplicateConfig = await this.$store.dispatch('data/connections/clone', config)
      duplicateConfig.name = 'Copy of ' + duplicateConfig.name

      try {
        const id = await this.$store.dispatch('data/connections/save', duplicateConfig)
        this.$noty.success(`The connection was successfully duplicated!`)
        this.config = this.connections.find((c) => c.id === id) || this.config
      } catch (ex) {
        this.$noty.error(`Could not duplicate Connection: ${ex.message}`)
      }

    },
    async submit() {
      this.connectionError = null
      try {
        await this.$store.dispatch('connect', this.config)
      } catch (ex) {
        this.connectionError = ex
        this.$noty.error("Error establishing a connection")
        log.error(ex)
      }
    },
    async handleConnect(config) {
      this.config = config
      await this.submit()
    },
    async testConnection() {
      try {
        this.testing = true
        this.connectionError = null
        await this.$store.dispatch('test', this.config)
        this.$noty.success("Connection looks good!")
        return true
      } catch (ex) {
        this.connectionError = ex
        this.$noty.error("Error establishing a connection")
      } finally {
        this.testing = false
      }
    },
    async save() {
      try {
        this.errors = null
        this.connectionError = null
        if (!this.config.name) {
          throw new Error("Name is required")
        }
        await this.$store.dispatch('data/connections/save', this.config)
        this.$noty.success("Connection Saved")
      } catch (ex) {
        console.error(ex)
        this.errors = [ex.message]
        this.$noty.error("Could not save connection information")
      }
    },
    handleErrorMessage(message) {
      if (message) {
        this.errors = [message]
        this.$noty.error("Could not parse connection URL.")
      } else {
        this.errors = null
      }
    }
  },
})
</script>

<style lang='scss' scoped>
#placeholder-card {
  align-items: center;
  color: #808080;
  display: flex;
  flex-direction: column;
  flex-wrap: nowrap;
  gap: 1rem;
  justify-content: center;

  .btn {
    width: 200px;
    color: #808080;
  }
}

#create-connection-card {
  overflow: visible;

  #create-connection-card__close {
    display: flex;
    margin: 0;
    padding: 0;
    position: absolute;
    right: -16px;
    top: -16px;

    .close {
      color: #5a5a5a;
      font-size: 16px;
      font-weight: bold;
      margin: 0;
      padding: 0;

      &:hover {
        color: #808080;
      }
    }
  }
}
</style>
