<script>
import { mapState, mapGetters } from 'vuex';
import { get } from '@/utils/object';
import { NAMESPACE, RIO } from '@/config/types';
import Card from '@/components/Card';
import { alternateLabel } from '@/utils/platform';

export default {
  components: { Card },
  data() {
    return { confirmName: '', error: '' };
  },
  computed:   {
    names() {
      return this.toRemove.map(obj => obj.nameDisplay).slice(0, 5);
    },

    type() {
      const types = new Set(this.toRemove.reduce((array, each) => {
        array.push(each.type);

        return array;
      }, []));

      if (types.size > 1) {
        return this.t('generic.resource', { count: this.toRemove.length });
      }

      const schema = this.toRemove[0]?.schema;

      if ( !schema ) {
        return `resource${ this.toRemove.length === 1 ? '' : 's' }`;
      }

      if ( this.toRemove.length > 1 ) {
        return this.$store.getters['type-map/pluralLabelFor'](schema).toLowerCase();
      } else {
        return this.$store.getters['type-map/singularLabelFor'](schema).toLowerCase();
      }
    },

    selfLinks() {
      return this.toRemove.map((resource) => {
        return get(resource, 'links.self');
      });
    },

    needsConfirm() {
      const first = this.toRemove[0];

      if ( !first ) {
        return false;
      }
      const type = first.type;

      return (type === NAMESPACE || type === RIO.STACK) && this.toRemove.length === 1;
    },

    preventDeletionMessage() {
      const toRemoveWithWarning = this.toRemove.filter(tr => tr?.preventDeletionMessage);

      if (toRemoveWithWarning.length === 0) {
        return null;
      }

      return toRemoveWithWarning[0].preventDeletionMessage;
    },

    isDeleteDisabled() {
      return !!this.preventDeletionMessage;
    },

    plusMore() {
      const remaining = this.toRemove.length - this.names.length;

      return this.t('promptRemove.andOthers', { count: remaining });
    },

    // if the current route ends with the ID of the resource being deleted, whatever page this is wont be valid after successful deletion: navigate away
    doneLocation() {
      // if deleting more than one resource, this is happening in list view and shouldn't redirect anywhere
      if (this.toRemove.length > 1) {
        return null;
      }
      const currentRoute = this.toRemove[0].currentRoute();
      const out = {};
      const params = { ...currentRoute.params };

      if (params.id === this.toRemove[0]?.metadata?.name || params.id === this.toRemove[0].id) {
        let { name = '' } = currentRoute;

        name = name.slice(0, name.indexOf('-id'));

        if (params.namespace) {
          name = name.slice(0, name.indexOf('-namespace'));
          delete params.namespace;
        }
        delete params.id;

        out.params = params;
        out.name = name;
      }

      return out;
    },

    currentRouter() {
      // ...don't need a router if there's no route to go to
      if (!this.doneLocation) {
        return null;
      } else {
        return this.toRemove[0].currentRouter();
      }
    },

    protip() {
      return this.t('promptRemove.protip', { alternateLabel });
    },

    ...mapState('action-menu', ['showPromptRemove', 'toRemove']),
    ...mapGetters({ t: 'i18n/t' })
  },

  watch:    {
    showPromptRemove(show) {
      if (show) {
        this.$modal.show('promptRemove');
      } else {
        this.$modal.hide('promptRemove');
      }
    }
  },

  methods: {
    close() {
      this.error = '';
      this.$store.commit('action-menu/togglePromptRemove');
    },

    remove() {
      if (this.needsConfirm && this.confirmName !== this.names[0]) {
        this.error = this.t('promptRemove.mismatch');
        // if doneLocation is defined, redirect after deleting
      } else if (this.doneLocation) {
        // doneLocation will recompute to undefined when delete request completes
        const goTo = { ...this.doneLocation };

        Promise.all(this.toRemove.map(resource => resource.remove())).then((results) => {
          // remove() calls 'cluster/request' which returns nothing for 204 responses
          if ((results[0] || {})._status === 200 || !results[0]) {
            this.confirmName = '';
            this.currentRouter.push(goTo);
            this.close();
          }
        }).catch((err) => {
          this.error = err;
        });
      } else {
        this.toRemove.map(resource => resource.remove());

        this.confirmName = '';
        this.close();
      }
    }
  }
};
</script>

<template>
  <modal
    class="remove-modal"
    name="promptRemove"
    :width="350"
    height="auto"
    styles="background-color: var(--nav-bg); border-radius: var(--border-radius); overflow: scroll; max-height: 100vh;"
  >
    <Card :style="{border:'none'}">
      <div slot="title" class="prompt-delete-title">
        <h4 class="text-default-text">
          <t k="promptRemove.title" />
        </h4>
        <div class="title-delete-icon" @click="close">
          <i class="icon icon-x" />
        </div>
      </div>
      <div slot="body">
        <div class="prompt-delete-content mb-10">
          {{ t('promptRemove.attemptingToRemove', {type}) }} <template v-for="(resource, i) in names">
            <template v-if="i<5">
              <a :key="resource" :href="selfLinks[i]">{{ resource }}</a><span v-if="i===names.length-1" :key="resource+2">{{ plusMore }}</span><span v-else :key="resource+1">{{ i === toRemove.length-2 ? ', and ' : ', ' }}</span>
            </template>
          </template>
          <span v-if="needsConfirm" :key="resource">{{ t('promptRemove.confirm') }}</span>
        </div>
        <input v-if="needsConfirm" id="confirm" v-model="confirmName" type="text" />
        <span class="text-warning">{{ preventDeletionMessage }}</span>
        <span class="text-error">{{ error }}</span>
        <!-- <span v-if="!needsConfirm" class="text-info mt-20">{{ protip }}</span> -->
      </div>
      <template slot="actions">
        <button class="btn btn-sm bg-default role-secondary" @click="close">
          {{ t('generic.cancel') }}
        </button>
        <button class="btn btn-sm bg-error" :disabled="isDeleteDisabled" @click="remove">
          {{ t('buttons.remove') }}
        </button>
      </template>
    </Card>
  </modal>
</template>

<style lang='scss'>
    #confirm {
        width: 90%;
        margin-left: 3px;
    }
    .remove-modal {
      border-radius: var(--border-radius);
      overflow: scroll;
      max-height: 100vh;
      & ::-webkit-scrollbar-corner {
        background: rgba(0,0,0,0);
      }
      .prompt-delete-title {
        width: 100%;
        position: relative;
      }
      .title-delete-icon {
        position: absolute;
        top: 0;
        right: 0;
        cursor: pointer;
      }
      .role-secondary:hover {
        box-shadow: 1px 1px 6px 0 var(--primary) !important;
      }
      .card-title {
        padding-top: 1rem;
      }
      .text-info {
        color: #213444;
      }
      .text-error {
        display: inline-block;
        margin-top: 5px;
      }
    }
</style>
