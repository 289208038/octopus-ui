<script>
import { cleanForNew } from '@/plugins/steve/normalize';
import CreateEditView from '@/mixins/create-edit-view';
import ResourceYaml from '@/components/ResourceYaml';
import {
  MODE, _VIEW, _EDIT, _CLONE, _STAGE,
  AS_YAML, _FLAGGED, _CREATE,
} from '@/config/query-params';
import { SCHEMA } from '@/config/types';
import { createYaml } from '@/utils/create-yaml';
import Masthead from '@/components/ResourceDetail/Masthead';
import GenericResourceDetail from '@/components/GenericResourceDetail';

// Components can't have asyncData, only pages.
// So you have to call this in the page and pass it in as a prop.
export async function asyncData(ctx) {
  const { params, store, route } = ctx;
  const resource = params.resource;
  const hasCustomEdit = store.getters['type-map/hasCustomEdit'](resource);
  const hasCustomDetail = store.getters['type-map/hasCustomDetail'](resource);
  const realMode = realModeFor(route.query.mode, params.id);

  if ( hasCustomEdit && [_EDIT, _CREATE, _STAGE].includes(realMode) ) {
    const importer = store.getters['type-map/importEdit'](resource);
    const component = (await importer())?.default;

    if ( component?.asyncData ) {
      return component.asyncData(ctx);
    }
  }

  if ( hasCustomDetail && realMode === _VIEW ) {
    const importer = store.getters['type-map/importDetail'](resource);
    const component = (await importer())?.default;

    if ( component?.asyncData ) {
      return component.asyncData(ctx);
    }
  }

  return defaultAsyncData(ctx);
}

function realModeFor(query, id) {
  if ( id ) {
    return query || _VIEW;
  } else {
    return _CREATE;
  }
}

// You can pass in a resource from a edit/someType.vue to use this but with a different type
// e.g. for workload to create a deployment
export async function defaultAsyncData(ctx, resource) {
  const { store, params, route } = ctx;

  // eslint-disable-next-line prefer-const
  let { namespace, id } = params;

  if ( !resource ) {
    resource = params.resource;
  }

  // There are 5 "real" modes that you can start in: view, edit, create, stage, clone
  // These are mapped down to the 3 regular page modes that create-edit-view components
  //  know about:  view, edit, create (stage and clone become "create")
  const realMode = realModeFor(route.query.mode, id);

  const hasCustomDetail = store.getters['type-map/hasCustomDetail'](resource);
  const hasCustomEdit = store.getters['type-map/hasCustomEdit'](resource);
  const asYamlInit = (route.query[AS_YAML] === _FLAGGED) || (realMode !== _VIEW && !hasCustomEdit);
  const schema = store.getters['cluster/schemaFor'](resource);
  const schemas = store.getters['cluster/all'](SCHEMA);

  let originalModel, model, yaml;

  if ( realMode === _CREATE ) {
    if ( !namespace ) {
      namespace = store.getters['defaultNamespace'];
    }

    const data = { type: resource };

    if ( schema.attributes.namespaced ) {
      data.metadata = { namespace };
    }

    originalModel = await store.dispatch('cluster/create', data);
    model = originalModel;

    yaml = createYaml(schemas, resource, data);
  } else {
    let fqid = id;

    if ( schema.attributes.namespaced && namespace ) {
      fqid = `${ namespace }/${ fqid }`;
    }

    originalModel = await store.dispatch('cluster/find', {
      type: resource, id: fqid, opt: { watch: true }
    });
    if (realMode === _VIEW) {
      model = originalModel;
    } else {
      model = await store.dispatch('cluster/clone', { resource: originalModel });
    }
    const link = originalModel.hasLink('rioview') ? 'rioview' : 'view';

    yaml = (await originalModel.followLink(link, { headers: { accept: 'application/yaml' } })).data;

    if ( realMode === _CLONE || realMode === _STAGE ) {
      cleanForNew(model);
      yaml = model.cleanYaml(yaml, realMode);
    }

    if ( model.applyDefaults ) {
      model.applyDefaults(ctx, realMode);
    }
  }

  let mode = realMode;

  if ( realMode === _STAGE || realMode === _CLONE ) {
    mode = _CREATE;
  }

  /*******
   * Important: these need to be declared below as props too if you want to use them
   *******/
  const out = {
    hasCustomDetail,
    hasCustomEdit,
    resource,
    model,
    asYamlInit,
    yaml,
    originalModel,
    mode,
    realMode,
  };
  /*******
   * Important: these need to be declared below as props too if you want to use them
   *******/

  return out;
}

export const watchQuery = [MODE, AS_YAML];

export default {
  components: {
    ResourceYaml, Masthead, GenericResourceDetail
  },
  mixins: { CreateEditView },

  props: {
    hasCustomDetail: {
      type:    Boolean,
      default: null,
    },
    hasCustomEdit: {
      type:    Boolean,
      default: null,
    },
    resource: {
      type:    String,
      default: null,
    },
    model: {
      type:    Object,
      default: null,
    },
    asYamlInit: {
      type:    Boolean,
      default: null,
    },
    yaml: {
      type:    String,
      default: null,
    },
    originalModel: {
      type:    Object,
      default: null,
    },
    mode: {
      type:    String,
      default: null
    },
  },

  data() {
    // asYamlInit is taken from route query and passed as prop from _id page; asYaml is saved in local data to be manipulated by Masthead
    const {
      asYamlInit: asYaml,
      value: currentValue,
    } = this;

    return {
      asYaml,
      currentValue,
      detailComponent:         this.$store.getters['type-map/importDetail'](this.resource),
      editComponent:           this.$store.getters['type-map/importEdit'](this.resource),
    };
  },

  computed: {
    realMode() {
      // There are 5 "real" modes that you can start in: view, edit, create, stage, clone
      // These are mapped down to the 3 regular page modes that create-edit-view components
      //  know about:  view, edit, create (stage and clone become "create")
      const realMode = realModeFor(this.$route.query.mode, this.model?.id);

      return realMode;
    },

    isView() {
      return this.mode === _VIEW;
    },

    isEdit() {
      return this.mode === _EDIT;
    },

    offerPreview() {
      return [_EDIT, _CLONE, _STAGE].includes(this.mode);
    },

    doneRoute() {
      let name = this.$route.name;

      if ( name.endsWith('-id') ) {
        name = name.replace(/(-namespace)?-id$/, '');
      } else if ( name.endsWith('-create') ) {
        name = name.replace(/-create$/, '');
      }

      return name;
    },

    doneParams() {
      return this.$route.params;
    },

    showComponent() {
      if ( this.isView ) {
        if (this.hasCustomDetail) {
          return this.detailComponent;
        } else if (this.hasCustomEdit) {
          return this.editComponent;
        } else {
          return GenericResourceDetail;
        }
      } else if ( this.hasCustomEdit ) {
        return this.editComponent;
      }

      return null;
    },
  },

  watch: {
    asYamlInit(neu) {
      this.asYaml = neu;
    },
    $route(route) {
      this.asYaml = route.query[AS_YAML] === _FLAGGED;
    }
  },

  methods: {
    // reading yamls from files is most easily tracked when done down in the component that handles other yaml-editing input, YamlEditor, but visually the button to upload lives up here
    readFromFile() {
      const component = this.$refs.resourceyaml;

      if (component) {
        component.readFromFile();
      }
    },
  }
};
</script>

<template>
  <div>
    <Masthead
      :value="originalModel"
      :mode="mode"
      :done-route="doneRoute"
      :real-mode="realMode"
      :as-yaml.sync="asYaml"
      :has-detail-or-edit="(hasCustomDetail || hasCustomEdit)"
    />
    <template v-if="asYaml">
      <div v-if="!isView">
        <button class="btn role-primary ml-20 mb-20" @click="readFromFile">
          {{ t('resourceList.head.readFromFile') }}
        </button>
      </div>
      <ResourceYaml
        ref="resourceyaml"
        :value="model"
        :mode="mode"
        :yaml="yaml"
        :offer-preview="offerPreview"
        :done-route="doneRoute"
        :done-override="model.doneOverride"
      />
    </template>
    <template v-else>
      <component
        :is="showComponent"
        v-model="model"
        :original-value="originalModel"
        :done-route="doneRoute"
        :done-params="doneParams"
        :mode="mode"
        :real-mode="realMode"
        :value="model"
        v-bind="_data"
        class="pl-20 pr-20"
      />
    </template>
  </div>
</template>

<style lang='scss' scoped>
  .actions > * {
    display: inline-block;
  }

  .flat {
    border-collapse: collapse;
    table-layout: fixed;
    width: 100%;
    & th{
      padding-bottom: 1rem;
      text-align: left;
      font-weight: normal;
      color: var(--secondary);
    }

    & :not(THEAD) tr{
      border-bottom: 1px solid var(--border);

      & td {
        padding: 10px 0 10px 0;
      }
    }
    & tr td:last-child, th:last-child{
      text-align: right;
    }

    & tr td:first-child, th:first-child{
      text-align: left;
      margin-left: 15px;
    }

    & .click-row a{
      color: var(--input-text);
    }

    & .click-row:hover{
      @extend .faded;
    }

    & .faded {
      opacity: 0.5
    }
  }
</style>
