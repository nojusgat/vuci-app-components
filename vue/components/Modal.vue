<template>
  <a-modal
    :title="title"
    :visible="visible"
    @cancel="handleCancel"
    :footer="false"
  >
    <vuci-form :uci-config="config" ref="form">
      <vuci-named-section :name="data.sid" v-slot="{ s }" :card="false" :key="data.sid">
        <vuci-form-item-select
          :uci-section="s"
          :label="$t('components.Protocol')"
          name="proto"
          :options="proto"
          initial="static"
          required
        />
        <vuci-form-item-input
          :uci-section="s"
          :label="$t('components.IPAddress')"
          name="address"
          depend="proto == 'static'"
          rules="ip4addr"
        />
        <vuci-form-item-input
          :uci-section="s"
          :label="$t('components.Netmask')"
          name="netmask"
          depend="proto == 'static'"
          rules="netmask4"
        />
        <vuci-form-item-input
          :uci-section="s"
          :label="$t('components.Gateway')"
          name="gateway"
          depend="proto == 'static'"
          rules="ip4addr"
        />
        <vuci-form-item-list
          :uci-section="s"
          :label="$t('components.DNS')"
          name="dns"
          depend="proto == 'static'"
          rules="ipaddr"
        />
      </vuci-named-section>
      <template #footer>
        <a-space style="display: flex; justify-content: flex-end;">
          <a-button type="primary" @click="handleSave" :loading="loading">
            {{ $t('components.Save') }}
          </a-button>
          <a-button type="danger" @click="handleCancel">
            {{ $t('components.Cancel') }}
          </a-button>
        </a-space>
      </template>
    </vuci-form>
  </a-modal>
</template>

<script>
export default {
  name: 'Modal',
  data () {
    return {
      proto: [
        ['static', this.$t('components.Static')],
        ['dhcp', this.$t('components.DHCP')]
      ],
      loading: false
    }
  },
  props: {
    type: {
      type: String,
      default: 'create'
    },
    data: {
      type: Object,
      default: null
    },
    visible: {
      type: Boolean,
      default: false
    },
    config: {
      type: String,
      required: true
    }
  },
  methods: {
    saveFormData () {
      const form = this.$refs.form
      // From VuciForm.vue apply() method
      const promises = []

      if (this.$uci.changed() > 0) {
        const p = new Promise(resolve => {
          this.$uci.save().then(() => {
            this.$uci.apply().then(() => resolve())
          })
        })

        promises.push(p)
      }

      Object.values(form.fields).forEach(item => {
        const p = item.__apply()
        if (window.vuci.isPromise(p)) promises.push(p)
      })

      if (promises.length === 0) {
        const error = new Error(this.$t('components.NoChanges'))
        error.metadata = 'warning'
        throw error
      }

      return Promise.all(promises)
    },
    handleSave () {
      const form = this.$refs.form

      this.loading = true
      // From VuciForm.vue apply() method
      form.validate()
        .then(valid => {
          if (!valid) {
            const error = new Error(this.$t('components.InvalidInputs'))
            error.metadata = 'warning'
            throw error
          }
        })
        .then(() => form.save())
        .then(() => this.saveFormData())
        .then(() => this.$uci.reset())
        .then(() => form.load())
        .then(() => form.reset())
        .then(() => {
          this.$message.success(
            this.type === 'create'
              ? this.$t('components.InterfaceCreated', { name: this.data.name })
              : this.$t('components.InterfaceEdited', { name: this.data.name })
          )
          this.$emit('cancel', { type: 'rerender', sid: null })
        })
        .catch(err => {
          if ('metadata' in err && err.metadata === 'warning') {
            this.$message.warning(err.message || this.$t('components.UnknownError'))
          } else {
            this.$message.error(err.message || this.$t('components.UnknownError'))
          }
        })
        .finally(() => {
          this.loading = false
        })
    },
    handleCancel () {
      if (this.type !== 'create') this.$refs.form.reset()
      this.$emit('cancel', { type: this.type, sid: this.data.sid })
    }
  },
  computed: {
    title () {
      return this.$t('components.ModalTitle', { name: this.data.name })
    }
  }
}
</script>
