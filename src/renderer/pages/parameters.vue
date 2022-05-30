<template>
  <v-container>
    <v-row v-if="alertMessage">
      <v-col>
        <v-alert type="error" dismissible>
          {{ alertMessage }}
        </v-alert>
      </v-col>
    </v-row>
    <v-row>
      <v-col>
        <h1>Settings</h1>
      </v-col>
    </v-row>
    <v-row>
      <v-col>
        <h3>Appearance</h3>
        <br />
        <v-switch v-model="modelSwitchDarkMode" label="Theme Switch - White/Dark" />
      </v-col>
    </v-row>
    <v-row>
      <v-col>
        <h3>Server</h3>
        <br />
        <v-text-field
          v-model="modelInputPort"
          :rules="[(v) => !!v || 'You must put a mandatory port']"
          label="Port"
          name="port"
          type="number"
        />
        <v-btn small color="primary" @click="savePortChange">Save</v-btn>
      </v-col>
    </v-row>
    <v-row>
      <v-col>
        <h3>Generation history</h3>
        <br />
        <v-switch
          v-model="modelSwitchHistoryQRCode"
          label="Enable/disable history"
        />
      </v-col>
    </v-row>
    <v-dialog v-model="dialogRestart" max-width="400" persistent>
      <v-card>
        <v-card-title class="headline">Attention</v-card-title>

        <v-card-text>
          {{ messageRestart }}
        </v-card-text>

        <v-card-actions>
          <v-spacer></v-spacer>

          <v-btn text @click="relaunch">
            Restart 3DSend
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-btn bottom color="primary" dark fab fixed right @click="restoreParams">
      <v-icon>mdi-restore</v-icon>
    </v-btn>
  </v-container>
</template>

<script>
import storage from 'electron-json-storage'
import { networkInterfaces } from 'os';
import tcpPortUsed from 'tcp-port-used'
import { remote } from 'electron'

export default {
  data: () => ({
    alertMessage: null,
    modelSwitchDarkMode: null,
    modelSwitchHistoryQRCode: null,
    modelInputPort: null,
    config: null,
    dialogRestart: false,
    messageRestart: null
  }),
  fetch() {
    const context = this
    storage.get('config', function(error, data) {
      if (error) throw error
      context.config = data
      context.modelSwitchDarkMode = data.dark ? data.dark : true
      context.modelSwitchHistoryQRCode = data.historyGenerate
        ? data.historyGenerate
        : true
      context.modelInputPort = data.port ? data.port : 9850
    })
  },
  watch: {
    modelSwitchDarkMode(value) {
      this.$vuetify.theme.dark = value
      storage.get('config', function(error, data) {
        storage.set('config', {
          dark: value,
          port: data.port ? data.port : 9850,
          historyGenerate: data.historyGenerate ? data.historyGenerate : true
        })
      })
    },
    modelSwitchHistoryQRCode(value) {
      storage.get('config', function(error, data) {
        storage.set('config', {
          dark: data.dark ? data.dark : true,
          port: data.port ? data.port : 9850,
          historyGenerate: value
        })
      })
    }
  },
  methods: {
    savePortChange() {
      const ipV4 = Object.values(networkInterfaces()).flat().find(i => i.family == 'IPv4' && !i.internal).address;
      const context = this
      tcpPortUsed.check(parseInt(this.modelInputPort), ipV4).then(
        function(inUse) {
          if (inUse) {
            context.alertMessage =
              'Port ' + context.modelInputPort + ' is already in use!'
          } else {
            storage.set('config', {
              dark: context.modelSwitchDarkMode,
              port: context.modelInputPort,
              historyGenerate: context.modelSwitchHistoryQRCode
            })
            context.messageRestart =
              '3DSend needs to restart to change the port!'
            context.dialogRestart = true
          }
        },
        function(err) {
          context.alertMessage = err
        }
      )
    },
    restoreParams() {
      storage.set('config', {
        dark: true,
        port: 9850,
        historyGenerate: true
      })
      this.$fetch()
      this.messageRestart =
        '3DSend needs to restart to apply changes!'
      this.dialogRestart = true
    },
    relaunch() {
      remote.getCurrentWindow().close()
    }
  }
}
</script>

<style scoped>
.v-input--selection-controls {
  margin-top: 0px;
  padding-top: 0px;
}
</style>