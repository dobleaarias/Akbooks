<template>
  <div class="flex flex-col w-full h-full">
    <PageHeader :title="t`Print ${title}`">
      <Button class="text-xs" type="primary" @click="savePDF">
        {{ t`Save as PDF` }}
      </Button>
    </PageHeader>

    <div class="outer-container">
      <!-- Report Print Display Area -->
      <div
        class="p-4 bg-gray-25 overflow-auto flex justify-center custom-scroll"
      >
        <!-- Report Print Display Container -->
        <ScaledContainer
          class="shadow-lg border bg-white"
          ref="scaledContainer"
          :scale="scale"
          :width="size.width"
          :height="size.height"
          :show-overflow="true"
        >
          <div class="bg-white mx-auto">
            <div class="p-2">
              <div class="font-semibold text-xl w-full flex justify-between">
                <h1>
                  {{ `${fyo.singles.PrintSettings?.companyName}` }}
                </h1>
                <p class="text-gray-600">
                  {{ title }}
                </p>
              </div>
            </div>

            <!-- Report Data -->
            <div class="grid" :style="rowStyles">
              <template v-for="(row, r) of matrix" :key="`row-${r}`">
                <div
                  v-for="(cell, c) of row"
                  :key="`cell-${r}.${c}`"
                  :class="cellClasses(cell.idx, r)"
                  class="text-sm p-2"
                  style="min-height: 2rem"
                >
                  {{ cell.value }}
                </div>
              </template>
            </div>

            <div class="border-t p-2">
              <p class="text-xs text-right w-full">
                {{ fyo.format(new Date(), 'Datetime') }}
              </p>
            </div>
          </div>
        </ScaledContainer>
      </div>

      <!-- Report Print Settings -->
      <div class="border-l flex flex-col" v-if="report">
        <p class="p-4 text-sm text-gray-600">
          {{
            [
              t`Hidden values will be visible on Print on.`,
              t`Report will use more than one page if required.`,
            ].join(' ')
          }}
        </p>
        <!-- Row Selection -->
        <div class="p-4 border-t">
          <Int
            :show-label="true"
            :border="true"
            :df="{
              label: t`Start From Row Index`,
              fieldtype: 'Int',
              fieldname: 'numRows',
              minvalue: 1,
              maxvalue: report?.reportData.length ?? 1000,
            }"
            :value="start"
            @change="(v) => (start = v)"
          />
          <Int
            class="mt-4"
            :show-label="true"
            :border="true"
            :df="{
              label: t`Number of Rows`,
              fieldtype: 'Int',
              fieldname: 'numRows',
              minvalue: 0,
              maxvalue: report?.reportData.length ?? 1000,
            }"
            :value="limit"
            @change="(v) => (limit = v)"
          />
        </div>

        <!-- Size Selection -->
        <div class="border-t p-4">
          <Select
            :show-label="true"
            :border="true"
            :df="printSizeDf"
            :value="printSize"
            @change="(v) => (printSize = v)"
          />
          <Check
            class="mt-4"
            :show-label="true"
            :border="true"
            :df="{
              label: t`Is Landscape`,
              fieldname: 'isLandscape',
              fieldtype: 'Check',
            }"
            :value="isLandscape"
            @change="(v) => (isLandscape = v)"
          />
        </div>

        <!-- Pick Columns -->
        <div class="border-t p-4">
          <h2 class="text-sm text-gray-600">
            {{ t`Pick Columns` }}
          </h2>
          <div class="border rounded grid grid-cols-2 mt-1">
            <Check
              v-for="(col, i) of report?.columns"
              :show-label="true"
              :key="col.fieldname"
              :df="{
                label: col.label,
                fieldname: col.fieldname,
                fieldtype: 'Check',
              }"
              :value="columnSelection[i]"
              @change="(v) => (columnSelection[i] = v)"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script lang="ts">
import { Report } from 'reports/Report';
import { reports } from 'reports/index';
import Button from 'src/components/Button.vue';
import Check from 'src/components/Controls/Check.vue';
import Int from 'src/components/Controls/Int.vue';
import PageHeader from 'src/components/PageHeader.vue';
import { getReport } from 'src/utils/misc';
import { PropType, defineComponent } from 'vue';
import ScaledContainer from '../TemplateBuilder/ScaledContainer.vue';
import { getPathAndMakePDF } from 'src/utils/printTemplates';
import { OptionField } from 'schemas/types';
import { paperSizeMap, printSizes } from 'src/utils/ui';
import Select from 'src/components/Controls/Select.vue';
import { showSidebar } from 'src/utils/refs';

export default defineComponent({
  props: {
    reportName: {
      type: String as PropType<keyof typeof reports>,
      required: true,
    },
  },
  data() {
    return {
      start: 1,
      limit: 0,
      printSize: 'A4' as typeof printSizes[number],
      isLandscape: false,
      scale: 0.65,
      report: null as null | Report,
      columnSelection: [] as boolean[],
    };
  },
  async mounted() {
    this.report = await getReport(this.reportName);
    this.limit = this.report.reportData.length;
    this.columnSelection = this.report.columns.map(() => true);
    this.setScale();
  },
  watch: {
    size() {
      this.setScale();
    },
  },
  computed: {
    title(): string {
      return reports[this.reportName]?.title ?? this.t`Report`;
    },
    printSizeDf(): OptionField {
      return {
        label: 'Print Size',
        fieldname: 'printSize',
        fieldtype: 'Select',
        options: printSizes
          .filter((p) => p !== 'Custom')
          .map((name) => ({ value: name, label: name })),
      };
    },
    matrix(): { value: string; idx: number }[][] {
      if (!this.report) {
        return [];
      }

      const columns = this.report.columns
        .map((col, idx) => ({ value: col.label, idx }))
        .filter((_, i) => this.columnSelection[i]);

      const matrix: { value: string; idx: number }[][] = [columns];
      const start = Math.max(this.start - 1, 1);
      const end = Math.min(start + this.limit, this.report.reportData.length);
      for (const i in this.report.reportData.slice(start, end)) {
        const row = this.report.reportData[Number(i) + start];

        matrix.push([]);
        for (const j in row.cells) {
          if (!this.columnSelection[j]) {
            continue;
          }

          const value = row.cells[j].value;
          matrix.at(-1)?.push({ value, idx: Number(j) });
        }
      }

      return matrix;
    },
    rowStyles(): Record<string, string> {
      const style: Record<string, string> = {};
      const numColumns = this.columnSelection.filter(Boolean).length;
      style['grid-template-columns'] = `repeat(${numColumns}, minmax(0, auto))`;
      return style;
    },
    size(): { width: number; height: number } {
      const size = paperSizeMap[this.printSize];
      const long = size.width > size.height ? size.width : size.height;
      const short = size.width <= size.height ? size.width : size.height;

      if (this.isLandscape) {
        return { width: long, height: short };
      }

      return { width: short, height: long };
    },
  },
  methods: {
    setScale() {
      const width = this.size.width * 37.2;
      let containerWidth = window.innerWidth - 26 * 16;
      if (showSidebar.value) {
        containerWidth -= 12 * 16;
      }

      this.scale = Math.min(containerWidth / width, 1);
    },
    async savePDF(): Promise<void> {
      // @ts-ignore
      const innerHTML = this.$refs.scaledContainer.$el.children[0].innerHTML;
      if (typeof innerHTML !== 'string') {
        return;
      }

      const name = this.title + ' - ' + this.fyo.format(new Date(), 'Date');
      await getPathAndMakePDF(
        name,
        innerHTML,
        this.size.width,
        this.size.height
      );
    },
    cellClasses(cIdx: number, rIdx: number): string[] {
      const classes: string[] = [];
      if (!this.report) {
        return classes;
      }

      const col = this.report.columns[cIdx];
      const isFirst = cIdx === 0;
      if (col.align) {
        classes.push(`text-${col.align}`);
      }

      if (rIdx === 0) {
        classes.push('font-semibold');
      }

      classes.push('border-t');
      if (!isFirst) {
        classes.push('border-l');
      }

      return classes;
    },
  },
  components: { PageHeader, Button, Check, Int, ScaledContainer, Select },
});
</script>
<style scoped>
.outer-container {
  display: grid;
  grid-template-columns: auto var(--w-quick-edit);
  @apply h-full overflow-auto;
}
</style>
