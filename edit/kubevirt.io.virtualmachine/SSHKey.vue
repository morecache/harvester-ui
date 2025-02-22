<script>
import LabeledInput from '@/components/form/LabeledInput';
import LabeledSelect from '@/components/form/LabeledSelect';
import InputOrDisplay from '@/components/InputOrDisplay';
import Banner from '@/components/Banner';
import Card from '@/components/Card';
import { clone } from '@/utils/object';
import { SSH } from '@/config/types';
import { _VIEW } from '@/config/query-params';

export default {
  components: {
    Card,
    Banner,
    LabeledInput,
    LabeledSelect,
    InputOrDisplay
  },

  props: {
    value: {
      type:    Array,
      default: () => {
        return [];
      }
    },
    mode: {
      type:     String,
      default: 'edit',
    },
    disableCreate: {
      type:    Boolean,
      default: false
    }
  },

  data() {
    return {
      checkedSsh:       this.value,
      publicKey:        '',
      sshName:          '',
      errors:           [],
      isAll:            false,
      checkAll:         false
    };
  },

  computed: {
    ssh() {
      return this.$store.getters['cluster/all'](SSH);
    },
    sshOption() {
      const choise = this.$store.getters['cluster/all'](SSH);

      return choise.map( (O) => {
        return {
          label: O.metadata.name,
          value: O.metadata.name
        };
      });
    },
    isView() {
      return this.mode === _VIEW || this.disableCreate;
    }
  },

  watch: {
    // checkedSsh() {
    //   this.$emit('update:sshKey', clone(this.checkedSsh));
    // },

    publicKey(neu) {
      const splitSSH = neu.split(/\s+/);

      if (splitSSH.length === 3) {
        if (splitSSH[2].includes('@')) {
          if (splitSSH[2].split('@')) {
            if (!this.sshName) {
              this.sshName = splitSSH[2].split('@')[0];
            }
          }
        }
      }
    },
  },

  methods: {
    show() {
      this.errors = [];
      this.$modal.show('newSSH');
    },

    hide() {
      this.$modal.hide('newSSH');
    },

    async saveKey() {
      this.errors = [];
      if (!this.sshName || !this.publicKey) {
        this.errors.push('Please fill in all required fields.');

        return;
      }

      if (this.sshName.length > 63) {
        const message = this.$store.getters['i18n/t']('harvester.validation.custom.tooLongName', { max: 63 });

        this.errors.push(message);

        return;
      }

      const SSH = 'harvester.cattle.io.keypair';

      try {
        await this.$store.dispatch('cluster/request', {
          method:  'POST',
          headers: {
            'content-type': 'application/json',
            accept:         'application/json',
          },
          url:  `v1/${ SSH }`,
          data: {
            apiVersion: 'vm.cattle.io/v1alpha1',
            kind:       'KeyPair',
            metadata:   { name: this.sshName },
            spec:       { publicKey: this.publicKey },
            type:       SSH
          },
        });

        this.checkedSsh.push(this.sshName);

        this.resetFields();

        this.hide();
      } catch (err) {
        this.errors = [err.message];
      }
    },

    addSSH() {
      this.show();
    },

    cancelKey() {
      this.hide();
    },

    resetFields() {
      this.sshName = '';
      this.publicKey = '';
    },

    update() {
      this.$emit('update:sshKey', clone(this.checkedSsh));
    }
  }
};
</script>

<template>
  <div>
    <div>
      <div class="keyLisk">
        <InputOrDisplay :name="t('harvester.vmPage.input.sshKey')" :value="checkedSsh" :mode="mode" class="mb-20">
          <LabeledSelect
            v-model="checkedSsh"
            :label="t('harvester.vmPage.input.sshKey')"
            :taggable="true"
            :mode="mode"
            :multiple="true"
            :searchable="true"
            :options="sshOption"
            @input="update"
          />
        </InputOrDisplay>
      </div>

      <span v-if="!isView" class="btn btn-sm bg-primary mt-20" @click="addSSH">
        {{ t('harvester.vmPage.buttons.createSSHKey') }}
      </span>
    </div>

    <modal name="newSSH" height="auto">
      <Card>
        <template #title>
          <h4 slot="title" class="text-default-text">
            {{ t('harvester.vmPage.sshTitle') }}
          </h4>
        </template>

        <template #body>
          <LabeledInput
            v-model="sshName"
            :label="t('harvester.vmPage.input.name')"
            class="mb-20"
            required
          />

          <LabeledInput
            v-model="publicKey"
            type="multiline"
            :label="t('harvester.vmPage.input.sshKeyValue')"
            class="mb-20"
            :min-height="160"
            required
          />

          <div v-for="(err,idx) in errors" :key="idx">
            <Banner color="error" :label="err" />
          </div>
        </template>

        <template #actions>
          <button class="btn bg-primary btn-sm mr-20" @click.prevent="cancelKey">
            {{ t('generic.cancel') }}
          </button>
          <button class="btn bg-primary btn-sm" @click.prevent="saveKey">
            {{ t('generic.save') }}
          </button>
        </template>
      </Card>
    </modal>
  </div>
</template>
