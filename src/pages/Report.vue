<template>
  <div class="flex flex-col w-full h-full">
    <PageHeader :title="title">
      <DropdownWithActions
        v-for="group of groupedActions"
        :icon="false"
        :key="group.label"
        :type="group.type"
        :actions="group.actions"
        class="text-xs"
      >
        {{ group.group }}
      </DropdownWithActions>
      <Button
        ref="printButton"
        :icon="true"
        :title="t`Open Report Print View`"
        @click="routeTo(`/report-print/${reportClassName}`)"
      >
        <feather-icon name="printer" class="w-4 h-4"></feather-icon>
      </Button>
    </PageHeader>

    <!-- Filters -->
    <div
      v-if="report && report.filters.length"
      class="grid grid-cols-5 gap-4 p-4 border-b"
    >
      <FormControl
        v-for="field in report.filters"
        :border="true"
        size="small"
        :class="[field.fieldtype === 'Check' ? 'self-end' : '']"
        :show-label="true"
        :key="field.fieldname + '-filter'"
        :df="field"
        :value="report.get(field.fieldname)"
        :read-only="loading"
        @change="async (value) => await report?.set(field.fieldname, value)"
      />
    </div>

    <!-- Report Body -->
    <ListReport v-if="report" :report="report" class="" />
  </div>
</template>
<script lang="ts">
import { computed } from '@vue/reactivity';
import { t } from 'fyo';
import { reports } from 'reports';
import { Report } from 'reports/Report';
import Button from 'src/components/Button.vue';
import FormControl from 'src/components/Controls/FormControl.vue';
import DropdownWithActions from 'src/components/DropdownWithActions.vue';
import PageHeader from 'src/components/PageHeader.vue';
import ListReport from 'src/components/Report/ListReport.vue';
import { fyo } from 'src/initFyo';
import { shortcutsKey } from 'src/utils/injectionKeys';
import { docsPathMap, getReport } from 'src/utils/misc';
import { docsPathRef } from 'src/utils/refs';
import { ActionGroup } from 'src/utils/types';
import { routeTo } from 'src/utils/ui';
import { PropType, defineComponent, inject } from 'vue';

export default defineComponent({
  setup() {
    return { shortcuts: inject(shortcutsKey) };
  },
  props: {
    reportClassName: {
      type: String as PropType<keyof typeof reports>,
      required: true,
    },
    defaultFilters: {
      type: String,
      default: '{}',
    },
  },
  data() {
    return {
      loading: false,
      report: null as null | Report,
    };
  },
  provide() {
    return {
      report: computed(() => this.report),
    };
  },
  components: {
    PageHeader,
    FormControl,
    ListReport,
    DropdownWithActions,
    Button,
  },
  async activated() {
    docsPathRef.value =
      docsPathMap[this.reportClassName] ?? docsPathMap.Reports!;
    await this.setReportData();

    const filters = JSON.parse(this.defaultFilters);
    const filterKeys = Object.keys(filters);
    for (const key of filterKeys) {
      await this.report?.set(key, filters[key]);
    }

    if (filterKeys.length) {
      await this.report?.updateData();
    }

    if (fyo.store.isDevelopment) {
      // @ts-ignore
      window.rep = this;
    }

    this.shortcuts?.pmod.set(this.reportClassName, ['KeyP'], () => {
      routeTo(`/report-print/${this.reportClassName}`);
    });
  },
  deactivated() {
    docsPathRef.value = '';
    this.shortcuts?.delete(this.reportClassName);
  },
  computed: {
    title() {
      return reports[this.reportClassName]?.title ?? t`Report`;
    },
    groupedActions() {
      const actions = this.report?.getActions() ?? [];
      const actionsMap = actions.reduce((acc, ac) => {
        if (!ac.group) {
          ac.group = 'none';
        }

        acc[ac.group] ??= {
          group: ac.group,
          label: ac.label ?? '',
          type: ac.type ?? 'secondary',
          actions: [],
        };

        acc[ac.group].actions.push(ac);
        return acc;
      }, {} as Record<string, ActionGroup>);

      return Object.values(actionsMap);
    },
  },
  methods: {
    routeTo,
    async setReportData() {
      if (this.report === null) {
        this.report = await getReport(this.reportClassName);
      }

      if (!this.report.reportData.length) {
        await this.report.setReportData();
      } else if (this.report.shouldRefresh) {
        await this.report.setReportData(undefined, true);
      }
    },
  },
});
</script>
