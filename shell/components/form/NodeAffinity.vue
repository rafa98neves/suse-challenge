<script>
import debounce from 'lodash/debounce';
import { _VIEW } from '@shell/config/query-params';
import { mapGetters } from 'vuex';
import { get, isEmpty, clone } from '@shell/utils/object';
import { NODE } from '@shell/config/types';
import MatchExpressions from '@shell/components/form/MatchExpressions';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { LabeledInput } from '@components/Form/LabeledInput';
import { randomStr } from '@shell/utils/string';
import ArrayListGrouped from '@shell/components/form/ArrayListGrouped';

export default {
  components: {
    ArrayListGrouped, MatchExpressions, LabeledSelect, LabeledInput
  },

  props: {
    // value should be NodeAffinity or VolumeNodeAffinity
    value: {
      type:    Object,
      default: () => {
        return {};
      }
    },

    mode: {
      type:    String,
      default: 'create'
    },

    // has select for matching fields or expressions (used for node affinity)
    // https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#nodeselectorterm-v1-core
    matchingSelectorDisplay: {
      type:    Boolean,
      default: false,
    },
  },

  data() {
    // VolumeNodeAffinity only has 'required' field
    if (this.value.required) {
      return { nodeSelectorTerms: this.value.required.nodeSelectorTerms };
    } else {
      const { preferredDuringSchedulingIgnoredDuringExecution = [], requiredDuringSchedulingIgnoredDuringExecution = {} } = this.value;
      const { nodeSelectorTerms = [] } = requiredDuringSchedulingIgnoredDuringExecution;
      const allSelectorTerms = [...preferredDuringSchedulingIgnoredDuringExecution, ...nodeSelectorTerms].map((term) => {
        const neu = clone(term);

        neu._id = randomStr(4);
        if (term.preference) {
          Object.assign(neu, term.preference);
          delete neu.preference;
        }

        return neu;
      });

      return {
        allSelectorTerms,
        weightedNodeSelectorTerms: preferredDuringSchedulingIgnoredDuringExecution,
        defaultWeight:             1,
        // rules in MatchExpressions.vue can not catch changes what happens on parent component
        // we need re-render it via key changing
        rerenderNums:              randomStr(4)
      };
    }
  },

  computed: {
    ...mapGetters({ t: 'i18n/t' }),
    isView() {
      return this.mode === _VIEW;
    },
    hasWeighted() {
      return !!this.weightedNodeSelectorTerms;
    },
    node() {
      return NODE;
    },
    affinityOptions() {
      const out = [this.t('workload.scheduling.affinity.preferred'), this.t('workload.scheduling.affinity.required')];

      return out;
    }
  },

  created() {
    this.queueUpdate = debounce(this.update, 500);
  },

  methods: {
    update() {
      const out = {};
      const requiredDuringSchedulingIgnoredDuringExecution = { nodeSelectorTerms: [] };
      const preferredDuringSchedulingIgnoredDuringExecution = [] ;

      this.allSelectorTerms.forEach((term) => {
        if (term.weight) {
          const neu = { weight: term.weight, preference: term };

          preferredDuringSchedulingIgnoredDuringExecution.push(neu);
        } else {
          requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms.push(term);
        }
      });

      if (preferredDuringSchedulingIgnoredDuringExecution.length) {
        out.preferredDuringSchedulingIgnoredDuringExecution = preferredDuringSchedulingIgnoredDuringExecution;
      }
      if (requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms.length) {
        out.requiredDuringSchedulingIgnoredDuringExecution = requiredDuringSchedulingIgnoredDuringExecution;
      }

      this.$emit('input', out);
    },

    remove() {
      this.rerenderNums = randomStr(4);
      this.queueUpdate();
    },

    changePriority(term) {
      if (term.weight) {
        this.$delete(term, 'weight');
      } else {
        this.$set(term, 'weight', 1);
      }
      this.update();
    },

    priorityDisplay(term) {
      return term.weight ? this.t('workload.scheduling.affinity.preferred') : this.t('workload.scheduling.affinity.required');
    },

    updateExpressions(row, expressions) {
      const expressionsMatching = {
        matchFields:      [],
        matchExpressions: []
      };

      if (expressions.length) {
        expressions.forEach((expression) => {
          expressionsMatching[expression.matching || 'matchExpressions'].push(expression);
        });

        if (expressionsMatching.matchFields.length) {
          this.$set(row, 'matchFields', expressionsMatching.matchFields);
        }
        if (expressionsMatching.matchExpressions.length) {
          this.$set(row, 'matchExpressions', expressionsMatching.matchExpressions);
        }

        this.update();
      }
    },

    get,

    isEmpty
  }

};
</script>

<template>
  <div
    class="row"
    @input="queueUpdate"
  >
    <div class="col span-12">
      <ArrayListGrouped
        v-model="allSelectorTerms"
        class="mt-20"
        :mode="mode"
        :default-add-value="{matchExpressions:[]}"
        :add-label="t('workload.scheduling.affinity.addNodeSelector')"
        @remove="remove"
      >
        <template #default="props">
          <div class="row">
            <div class="col span-9">
              <LabeledSelect
                :options="affinityOptions"
                :value="priorityDisplay(props.row.value)"
                :label="t('workload.scheduling.affinity.priority')"
                :mode="mode"
                :data-testid="`node-affinity-priority-index${props.i}`"
                @input="(changePriority(props.row.value))"
              />
            </div>
            <div
              v-if="props.row.value.weight"
              class="col span-3"
            >
              <LabeledInput
                v-model.number="props.row.value.weight"
                :mode="mode"
                type="number"
                min="1"
                max="100"
                :label="t('workload.scheduling.affinity.weight.label')"
                :placeholder="t('workload.scheduling.affinity.weight.placeholder')"
                :data-testid="`node-affinity-weight-index${props.i}`"
              />
            </div>
          </div>
          <MatchExpressions
            :key="rerenderNums"
            :value="matchingSelectorDisplay ? props.row.value : props.row.value.matchExpressions"
            :matching-selector-display="matchingSelectorDisplay"
            :mode="mode"
            class="col span-12 mt-20"
            :type="node"
            :show-remove="false"
            :data-testid="`node-affinity-expressions-index${props.i}`"
            @input="(updateExpressions(props.row.value, $event))"
          />
        </template>
      </ArrayListGrouped>
    </div>
  </div>
</template>

<style>
</style>
