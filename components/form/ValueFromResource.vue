<script>
import { CONFIG_MAP, SECRET, NAMESPACE } from '@/config/types';
import { get } from '@/utils/object';
import LabeledSelect from '@/components/form/LabeledSelect';
import LabeledInput from '@/components/form/LabeledInput';
import { _EDIT } from '@/config/query-params';

export default {
  components: {
    LabeledSelect,
    LabeledInput
  },
  props:      {
    mode: {
      type:    String,
      default: 'create'
    },
    row: {
      type:    Object,
      default: () => {
        return { valueFrom: {} };
      }
    },
    allConfigMaps: {
      type:    Array,
      default: () => []
    },
    allSecrets: {
      type:    Array,
      default: () => []
    },
    // filter resource options by namespace(s) selected in top nav
    namespaced: {
      type:    Boolean,
      default: true
    }
  },
  data() {
    const typeOpts = [
      { value: 'resourceFieldRef', label: 'Resource' },
      { value: 'configMapKeyRef', label: 'ConfigMap' },
      { value: 'secretKeyRef', label: 'Secret key' },
      { value: 'fieldRef', label: 'Field' },
      { value: 'secretRef', label: 'Secret' }];

    let type;

    if (this.row.secretRef) {
      type = 'secretRef';
    } else if (this.row.valueFrom) {
      type = Object.keys((this.row.valueFrom))[0];
    } else {
      type = Object.keys(this.row)[0];
    }

    let refName;
    let name;
    let fieldPath;
    let referenced;
    let key;

    switch (type) {
    case 'resourceFieldRef':
      name = this.row.name;
      type = Object.keys(this.row.valueFrom)[0];
      referenced = this.row.valueFrom[type].containerName;
      key = this.row.valueFrom[type].resource || '';
      refName = referenced;
      break;
    case 'configMapKeyRef':
      name = this.row.name;
      type = Object.keys(this.row.valueFrom)[0];
      key = this.row.valueFrom[type].key || '';
      refName = this.row.valueFrom[type].name;
      break;
    case 'secretRef':
      name = this.row.prefix;
      type = 'secretRef';
      refName = this.row[type].name;
      break;
    case 'secretKeyRef':
      name = this.row.name;
      type = Object.keys(this.row.valueFrom)[0];
      key = this.row.valueFrom[type].key || '';
      refName = this.row.valueFrom[type].name;
      break;
    case 'fieldRef':
      fieldPath = get(this.row.valueFrom, `${ type }.fieldPath`) || '';
      name = this.row.name;
      break;
    default:
      break;
    }

    return {
      typeOpts,
      type,
      refName,
      referenced: refName,
      secrets:    this.allSecrets,
      keys:       [],
      key,
      fieldPath,
      name
    };
  },
  computed: {

    namespaces() {
      if (this.namespaced) {
        const map = this.$store.getters.namespaces();

        return Object.keys(map).filter(key => map[key]);
      } else {
        return this.$store.getters['cluster/findAll'](NAMESPACE);
      }
    },

    sourceOptions() {
      if (this.type === 'configMapKeyRef') {
        return this.allConfigMaps.filter(map => this.namespaces.includes(map?.metadata?.namespace));
      } else if (this.type === 'secretRef' || this.type === 'secretKeyRef') {
        return this.allSecrets.filter(secret => this.namespaces.includes(secret?.metadata?.namespace));
      } else {
        return [];
      }
    },

    hideLabel() {
      return this.mode !== _EDIT;
    }
  },

  watch: {
    type() {
      this.referenced = null;
    },

    referenced(neu, old) {
      if (old) {
        this.key = '';
      }
      if (neu) {
        if (neu.type === SECRET || neu.type === CONFIG_MAP) {
          this.keys = Object.keys(neu.data || {});
        }
        this.refName = neu?.metadata?.name;
      }
    },
  },

  mounted() {
    const typeSelect = this.$refs.typeSelect;

    if (typeSelect && this.mode === 'create') {
      typeSelect.open();
    }
  },

  methods: {
    async  getReferenced(name, type) {
      const resource = await this.$store.dispatch('cluster/find', { id: name, type });

      this.referenced = resource;
    },

    updateRow() {
      const out = { name: this.name || this.refName };
      const old = { ...this.row };

      switch (this.type) {
      case 'configMapKeyRef':
      case 'secretKeyRef':
        out.valueFrom = {
          [this.type]: {
            key: this.key, name: this.refName, optional: false
          }
        };
        break;
      case 'resourceFieldRef':
        out.valueFrom = {
          [this.type]: {
            containerName: this.refName, divisor: 0, resource: this.key
          }
        };
        break;
      case 'fieldRef':
        out.valueFrom = { [this.type]: { apiVersion: 'v1', fieldPath: this.fieldPath } };
        break;
      default:
        delete out.name;
        out.prefix = this.name;
        out[this.type] = { name: this.refName, optional: false };
      }
      this.$emit('input', { value: out, old });
    },

  }
};
</script>

<template>
  <div class="row" @input="updateRow">
    <div class="col span-5-of-23">
      <LabeledSelect
        ref="typeSelect"
        v-model="type"
        :multiple="false"
        :options="typeOpts"
        :label="t('workload.container.command.type')"
        :mode="mode"
        option-label="label"
        :hide-label="hideLabel"
        @input="updateRow"
      />
    </div>
    <template v-if="type === 'configMapKeyRef' || type === 'secretRef' || type === 'secretKeyRef'">
      <div class="col span-5-of-23">
        <LabeledSelect
          v-model="referenced"
          :options="sourceOptions"
          :multiple="false"
          :label="t('workload.container.command.source')"
          option-label="metadata.name"
          option-key
          :mode="mode"
          :hide-label="hideLabel"
          @input="updateRow"
        />
      </div>
      <div class="col span-5-of-23">
        <LabeledSelect
          ref="typeSelect"
          v-model="key"
          :multiple="false"
          :options="keys"
          :mode="mode"
          :label="t('workload.container.command.key')"
          option-label="label"
          :hide-label="hideLabel"
          @input="updateRow"
        />
      </div>
    </template>
    <template v-else-if="type==='resourceFieldRef'">
      <div class="col span-5-of-23">
        <LabeledInput v-model="refName" :label="t('workload.container.command.source')" placeholder="e.g. my-container" :mode="mode" :hide-label="hideLabel" />
      </div>

      <div class="col span-5-of-23">
        <LabeledInput v-model="key" :label="t('workload.container.command.key')" placeholder="e.g. requests.cpu" :mode="mode" :hide-label="hideLabel" />
      </div>
    </template>
    <template v-else>
      <div class="col span-5-of-23">
        <LabeledInput v-model="fieldPath" :label="t('workload.container.command.source')" placeholder="e.g. requests.cpu" :mode="mode" :hide-label="hideLabel" />
      </div>

      <div class="col span-5-of-23">
        <LabeledInput
          value="n/a"
          :label="t('workload.container.command.key')"
          placeholder="e.g. requests.cpu"
          disabled
          :mode="mode"
          :hide-label="hideLabel"
          :class="{'hide-background': hideLabel}"
        />
      </div>
    </template>
    <div class="col span-1-of-23">
      <div id="as" :class="{'pb-10': hideLabel}">
        as
      </div>
    </div>
    <div class="col span-5-of-23">
      <LabeledInput v-model="name" :label="t('workload.container.command.prefixOfAlias')" :mode="mode" :hide-label="hideLabel" />
    </div>
    <div class="col span-2-of-23">
      <button v-if="mode!=='view'" id="remove" type="button" class="btn btn-sm role-link" @click="$emit('input', { value:null })">
        remove
      </button>
    </div>
  </div>
</template>

<style lang ="scss" scoped>
  .row{
    display: flex;
    align-items: center;

    .hide-background {
      background: none!important;
    }
  }

  #as {
    text-align:center;
  }
  #remove{
    padding: 0px;
  }

</style>
