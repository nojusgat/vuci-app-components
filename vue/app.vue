<template>
  <div>
    <vuci-form :uci-config="uciConfig" v-if="renderForm">
      <vuci-typed-section type="interface" :columns="columns">
        <template #name="{ s }">
          <vuci-form-item-dummy :uci-section="s" name="name" class="item" />
        </template>
        <template #address="{ s }">
          <vuci-form-item-dummy :uci-section="s" name="address" class="item" />
        </template>
        <template #netmask="{ s }">
          <vuci-form-item-dummy :uci-section="s" name="netmask" class="item" />
        </template>
        <template #actions="{ s }">
          <a-space style="display: flex; justify-content: center">
            <a-button type="primary" @click="handleEdit(s)">{{ $t('components.Edit') }}</a-button>
            <a-button type="danger" @click="handleDelete(s)">{{ $t('components.Delete') }}</a-button>
          </a-space>
        </template>
      </vuci-typed-section>
      <template #footer>
        <a-row type="flex" justify="center" style="gap: 10px; margin-top: 10px;">
          <a-col :flex="3" class="col-input">
            <span>{{ $t('components.InterfaceName') }}</span>
            <a-input v-model="interfaceName" @pressEnter="handleCreate" />
          </a-col>
          <a-col :flex="2" class="col-button">
            <a-button type="primary" @click="handleCreate">
              {{ $t('components.Create') }}
            </a-button>
          </a-col>
        </a-row>
      </template>
    </vuci-form>
    <Modal
      v-bind="modalData"
      :config="uciConfig"
      @cancel="cancelModal"
      :key="modalData.data.sid"
    />
  </div>
</template>

<script>
import Modal from './components/Modal.vue'

export default {
  components: {
    Modal
  },
  data () {
    return {
      renderForm: true,
      uciConfig: 'vuci_components_task',
      columns: [
        { name: 'name', label: this.$t('components.InterfaceName') },
        { name: 'address', label: this.$t('components.IPAddress') },
        { name: 'netmask', label: this.$t('components.Netmask') },
        { name: 'actions', label: '' }
      ],
      interfaceName: '',
      modalData: {
        visible: false,
        type: 'create',
        data: {
          sid: 0,
          name: null
        }
      }
    }
  },
  methods: {
    handleCreate () {
      if (this.interfaceName === '') {
        this.$message.warning(this.$t('components.InterfaceNameEmpty'))
        return
      }

      this.$spin()
      const sid = this.$uci.add(this.uciConfig, 'interface')
      this.$uci.set(this.uciConfig, sid, 'name', this.interfaceName)
      this.$uci.save()
        .then(() => this.$uci.apply())
        .then(() => this.latestSid())
        .then(sid => {
          if (sid === null) throw Error(this.$t('components.FailedInterfaceInfo'))

          this.modalData.type = 'create'
          this.modalData.visible = true
          this.modalData.data.sid = sid
          this.modalData.data.name = this.interfaceName
          this.interfaceName = ''
        })
        .catch(err => {
          this.$message.error(err.message || this.$t('components.UnknownError'))
        })
        .finally(() => {
          this.$spin(false)
        })
    },
    handleEdit (s) {
      this.modalData.type = 'edit'
      this.modalData.visible = true
      this.modalData.data.sid = s['.name']
      this.modalData.data.name = s.name
    },
    handleDelete (s) {
      const _this = this
      this.$confirm({
        title: _this.$t('components.DeleteConfirm', { name: s.name }),
        onOk () {
          return new Promise(resolve => {
            _this.$uci.del(_this.uciConfig, s['.name'])
            _this.$uci.save()
              .then(() => _this.$uci.apply())
              .then(() => resolve())
              .then(() => _this.rerenderForm())
              .then(() => _this.$message.success(_this.$t('components.DeleteSuccess', { name: s.name })))
              .catch(err => {
                _this.$message.error(err.message || _this.$t('components.UnknownError'))
              })
          })
        }
      })
    },
    async cancelModal ({ type, sid }) {
      if (type === 'create') {
        this.$spin()
        this.$uci.del(this.uciConfig, sid)
        await this.$uci.save()
          .then(() => this.$uci.apply())
          .catch(err => {
            this.$message.error(err.message || this.$t('components.UnknownError'))
          })
          .finally(() => {
            this.$spin(false)
          })
      } else if (type === 'rerender') {
        this.rerenderForm()
      }
      this.modalData.visible = false
    },
    async rerenderForm () {
      this.renderForm = false
      await this.$nextTick()
      this.renderForm = true
    },
    async latestSid () {
      this.$uci.reset()
      await this.$uci.load(this.uciConfig)
      const lastElement = this.$uci.sections(this.uciConfig, 'interface').at(-1)
      return lastElement !== null && '.name' in lastElement ? lastElement['.name'] : null
    }
  }
}
</script>

<style scoped>
.item {
  margin-bottom: 0;
}
.col-input {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
  justify-content: center;
}
.col-input input {
  width: 50%;
  min-width: 250px;
}
.col-button {
  display: flex;
  justify-content: center;
}
</style>
