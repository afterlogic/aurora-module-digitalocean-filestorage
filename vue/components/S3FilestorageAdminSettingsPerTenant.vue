<template>
  <q-scroll-area class="full-height full-width">
    <div class="q-pa-lg ">
      <div class="row q-mb-md">
        <div class="col text-h5" v-t="'CPANELINTEGRATOR.HEADING_SETTINGS_TAB'"></div>
      </div>
      <q-card flat bordered class="card-edit-settings">
        <q-card-section>
          <div class="row">
            <div class="col-2 q-my-sm" v-t="'CPANELINTEGRATOR.LABEL_CPANEL_HOST'"></div>
            <div class="col-5">
              <q-input outlined dense bg-color="white" v-model="region" @keyup.enter="save"/>
            </div>
          </div>
        </q-card-section>
      </q-card>
      <div class="q-pt-md text-right">
        <q-btn unelevated no-caps dense class="q-px-sm" :ripple="false" color="primary" @click="save"
               :label="saving ? $t('COREWEBCLIENT.ACTION_SAVE_IN_PROGRESS') : $t('COREWEBCLIENT.ACTION_SAVE')">
        </q-btn>
      </div>
    </div>
    <q-inner-loading style="justify-content: flex-start;" :showing="loading || saving">
      <q-linear-progress query />
    </q-inner-loading>
    <UnsavedChangesDialog ref="unsavedChangesDialog"/>
  </q-scroll-area>
</template>

<script>
import _ from 'lodash'

import errors from 'src/utils/errors'
import notification from 'src/utils/notification'
import types from 'src/utils/types'
import webApi from 'src/utils/web-api'

import UnsavedChangesDialog from 'src/components/UnsavedChangesDialog'

export default {
  name: 'S3FilestorageAdminSettingsPerTenant',

  components: {
    UnsavedChangesDialog
  },

  data () {
    return {
      saving: false,
      loading: false,
      tenant: null,
      region: ''
    }
  },

  computed: {
    tenantId () {
      return this.$store.getters['tenants/getCurrentTenantId']
    },

    allTenants () {
      return this.$store.getters['tenants/getTenants']
    },
  },

  watch: {
    allTenants () {
      this.populate()
    },
  },

  beforeRouteLeave (to, from, next) {
    if (this.hasChanges() && _.isFunction(this?.$refs?.unsavedChangesDialog?.openConfirmDiscardChangesDialog)) {
      this.$refs.unsavedChangesDialog.openConfirmDiscardChangesDialog(next)
    } else {
      next()
    }
  },

  mounted() {
    this.loading = false
    this.saving = false
    this.populate()
  },

  methods: {
    hasChanges () {
      const tenantCompleteData = types.pObject(this.tenant?.completeData)
      return this.region !== tenantCompleteData['S3Filestorage::Region']
    },

    populate () {
      const tenant = this.$store.getters['tenants/getTenant'](this.tenantId)
      if (tenant) {
        if (tenant.completeData['S3Filestorage::Region'] !== undefined) {
          this.tenant = tenant
          this.region = tenant.completeData['S3Filestorage::Region']
        } else {
          this.getSettings()
        }
      }
    },

    save () {
      if (!this.saving) {
        this.saving = true
        const parameters = {
          TenantId: this.tenantId,
          Region: this.region,
        }
        webApi.sendRequest({
          moduleName: 'S3Filestorage',
          methodName: 'UpdateS3Settings',
          parameters
        }).then(result => {
          this.saving = false
          if (result) {
            const data = {
              'S3Filestorage::Region': parameters.Region,
            }
            this.$store.commit('tenants/setTenantCompleteData', { id: this.tenantId, data })
            notification.showReport(this.$t('COREWEBCLIENT.REPORT_SETTINGS_UPDATE_SUCCESS'))
          } else {
            notification.showError(this.$t('COREWEBCLIENT.ERROR_SAVING_SETTINGS_FAILED'))
          }
        }, response => {
          this.saving = false
          notification.showError(errors.getTextFromResponse(response, this.$t('COREWEBCLIENT.ERROR_SAVING_SETTINGS_FAILED')))
        })
      }
    },

    getSettings () {
      this.loading = true
      const parameters = {
        TenantId: this.tenantId,
      }
      webApi.sendRequest({
        moduleName: 'S3Filestorage',
        methodName: 'GetSettings',
        parameters
      }).then(result => {
        this.loading = false
        if (result) {
          const data = {
            'S3Filestorage::Region': types.pString(result.Region),
          }
          this.$store.commit('tenants/setTenantCompleteData', { id: this.tenantId, data })
        }
      }, response => {
        notification.showError(errors.getTextFromResponse(response))
      })
    }
  }
}
</script>

<style scoped>

</style>
