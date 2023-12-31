<template>
  <Popover
    @close="emitFilterChange"
    placement="bottom-end"
    v-if="fields.length"
  >
    <template #target="{ togglePopover }">
      <Button :icon="true" @click="togglePopover()">
        <span class="flex items-center">
          <Icon name="filter" size="12" class="stroke-current text-gray-700" />
          <span class="ms-1">
            <template v-if="activeFilterCount > 0">
              {{ filterAppliedMessage }}
            </template>
            <template v-else>
              {{ t`Filter` }}
            </template>
          </span>
        </span>
      </Button>
    </template>
    <template #content>
      <div>
        <div class="p-2">
          <template v-if="explicitFilters.length">
            <div class="flex flex-col gap-2">
              <div
                :key="filter.fieldname + getRandomString()"
                v-for="(filter, i) in explicitFilters"
                class="flex items-center justify-between text-base gap-2"
              >
                <div
                  class="
                    cursor-pointer
                    w-4
                    h-4
                    flex
                    items-center
                    justify-center
                    text-gray-600
                    hover:text-gray-800
                    rounded-md
                    group
                  "
                >
                  <span class="hidden group-hover:inline-block">
                    <feather-icon
                      name="x"
                      class="w-4 h-4 cursor-pointer"
                      :button="true"
                      @click="removeFilter(filter)"
                    />
                  </span>
                  <span class="group-hover:hidden">
                    {{ i + 1 }}
                  </span>
                </div>
                <FormControl
                  :border="true"
                  size="small"
                  class="w-24"
                  :df="{
                    placeholder: t`Field`,
                    fieldname: 'fieldname',
                    fieldtype: 'Select',
                    options: fieldOptions,
                  }"
                  :value="filter.fieldname"
                  @change="(value) => (filter.fieldname = value)"
                />
                <FormControl
                  :border="true"
                  size="small"
                  class="w-24"
                  :df="{
                    placeholder: t`Condition`,
                    fieldname: 'condition',
                    fieldtype: 'Select',
                    options: conditions,
                  }"
                  :value="filter.condition"
                  @change="(value) => (filter.condition = value)"
                />
                <FormControl
                  :border="true"
                  size="small"
                  class="w-24"
                  :df="{
                    placeholder: t`Value`,
                    fieldname: 'value',
                    fieldtype: 'Data',
                  }"
                  :value="filter.value"
                  @change="(value) => (filter.value = value)"
                />
              </div>
            </div>
          </template>
          <template v-else>
            <span class="text-base text-gray-600">{{
              t`No filters selected`
            }}</span>
          </template>
        </div>
        <div
          class="
            text-base
            border-t
            p-2
            flex
            items-center
            text-gray-600
            cursor-pointer
            hover:bg-gray-100
          "
          @click="addNewFilter"
        >
          <feather-icon name="plus" class="w-4 h-4" />
          <span class="ms-2">{{ t`Add a filter` }}</span>
        </div>
      </div>
    </template>
  </Popover>
</template>
<script lang="ts">
import { t } from 'fyo';
import { Field, FieldTypeEnum } from 'schemas/types';
import { fyo } from 'src/initFyo';
import { getRandomString } from 'utils';
import Button from './Button.vue';
import FormControl from './Controls/FormControl.vue';
import Icon from './Icon.vue';
import Popover from './Popover.vue';
import { defineComponent } from 'vue';

const conditions = [
  { label: t`Is`, value: '=' },
  { label: t`Is Not`, value: '!=' },
  { label: t`Contains`, value: 'like' },
  { label: t`Does Not Contain`, value: 'not like' },
  { label: t`Greater Than`, value: '>' },
  { label: t`Less Than`, value: '<' },
  { label: t`Is Empty`, value: 'is null' },
  { label: t`Is Not Empty`, value: 'is not null' },
] as const;

type Condition = typeof conditions[number]['value'];

type Filter = {
  fieldname: string;
  condition: Condition;
  value: string;
  implicit: boolean;
};

export default defineComponent({
  name: 'FilterDropdown',
  components: {
    Popover,
    Button,
    Icon,
    FormControl,
  },
  props: { schemaName: { type: String, required: true } },
  emits: ['change'],
  data() {
    return {
      filters: [],
    } as { filters: Filter[] };
  },
  created() {
    this.addNewFilter();
  },
  methods: {
    getRandomString,
    addNewFilter(): void {
      const df = this.fields[0];
      if (!df) {
        return;
      }

      this.addFilter(df.fieldname, 'like', '', false);
    },
    addFilter(
      fieldname: string,
      condition: Condition,
      value: string,
      implicit?: boolean
    ): void {
      this.filters.push({ fieldname, condition, value, implicit: !!implicit });
    },
    removeFilter(filter: Filter): void {
      this.filters = this.filters.filter((f) => f !== filter);
    },
    setFilter(filters: Record<string, Filter>, implicit?: boolean): void {
      this.filters = [];

      Object.keys(filters).map((fieldname) => {
        let parts = filters[fieldname];
        let condition, value;

        if (Array.isArray(parts)) {
          condition = parts[0];
          value = parts[1];
        } else {
          condition = '=';
          value = parts;
        }

        this.addFilter(fieldname, condition, value, implicit);
      });

      this.emitFilterChange();
    },
    emitFilterChange(): void {
      const filters: Record<string, [Condition, string]> = {};
      for (const { condition, value, fieldname } of this.filters) {
        if (value === '' && condition) {
          continue;
        }

        filters[fieldname] = [condition, value];
      }

      this.$emit('change', filters);
    },
  },
  computed: {
    fields(): Field[] {
      const excludedFieldsTypes: string[] = [
        FieldTypeEnum.Table,
        FieldTypeEnum.Attachment,
        FieldTypeEnum.AttachImage,
      ];
      const fields = fyo.schemaMap[this.schemaName]?.fields ?? [];
      return fields.filter((f) => {
        if (f.filter) {
          return true;
        }

        if (excludedFieldsTypes.includes(f.fieldtype)) {
          return false;
        }

        if (f.computed || f.meta || f.readOnly) {
          return false;
        }

        return true;
      });
    },
    fieldOptions(): { label: string; value: string }[] {
      return this.fields.map((df) => ({
        label: df.label,
        value: df.fieldname,
      }));
    },
    conditions(): typeof conditions {
      return conditions;
    },
    explicitFilters(): Filter[] {
      return this.filters.filter((f) => !f.implicit);
    },
    activeFilterCount(): number {
      return this.explicitFilters.filter((filter) => filter.value).length;
    },
    filterAppliedMessage(): string {
      if (this.activeFilterCount === 1) {
        return this.t`1 filter applied`;
      }

      return this.t`${this.activeFilterCount} filters applied`;
    },
  },
});
</script>
