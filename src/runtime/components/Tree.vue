<script lang="ts">
import type { VariantProps } from 'tailwind-variants'
import type { TreeRootProps, TreeRootEmits } from 'reka-ui'
import type { AppConfig } from '@nuxt/schema'
import _appConfig from '#build/app.config'
import theme from '#build/ui/tree'
import { tv } from '../utils/tv'
import type { PartialString, DynamicSlots, MaybeMultipleModelValue, SelectItemKey } from '../types/utils'

const appConfig = _appConfig as AppConfig & { ui: { tree: Partial<typeof theme> } }

const tree = tv({ extend: tv(theme), ...(appConfig.ui?.tree || {}) })

type TreeVariants = VariantProps<typeof tree>

export type TreeItem = {
  /**
   * @IconifyIcon
   */
  icon?: string
  label?: string
  /**
   * @IconifyIcon
   */
  trailingIcon?: string
  defaultExpanded?: boolean
  disabled?: boolean
  value?: string
  slot?: string
  children?: TreeItem[]
  onToggle?(e: Event): void
  onSelect?(e?: Event): void
}

export interface TreeProps<T extends TreeItem, M extends boolean = false, K extends SelectItemKey<T> | undefined = undefined> extends Pick<TreeRootProps<T>, 'expanded' | 'defaultExpanded' | 'selectionBehavior' | 'propagateSelect' | 'disabled'> {
  /**
   * The element or component this component should render as.
   * @defaultValue 'ul'
   */
  as?: any
  /**
   * @defaultValue 'primary'
   */
  color?: TreeVariants['color']
  /**
   * @defaultValue 'md'
   */
  size?: TreeVariants['size']
  /**
   * The key used to get the value from the item.
   * @defaultValue 'value'
   */
  valueKey?: K
  /**
   * The key used to get the label from the item.
   * @defaultValue 'label'
   */
  labelKey?: K
  /**
   * The icon displayed on the right side of a parent node.
   * @defaultValue appConfig.ui.icons.chevronDown
   * @IconifyIcon
   */
  trailingIcon?: string
  /**
   * The icon displayed when a parent node is expanded.
   * @defaultValue appConfig.ui.icons.folderOpen
   * @IconifyIcon
   */
  expandedIcon?: string
  /**
   * The icon displayed when a parent node is collapsed.
   * @defaultValue appConfig.ui.icons.folder
   * @IconifyIcon
   */
  collapsedIcon?: string
  items?: T[]
  /** The controlled value of the Tree. Can be bind as `v-model`. */
  modelValue?: MaybeMultipleModelValue<T, M>
  /** The value of the Tree when initially rendered. Use when you do not need to control the state of the Tree. */
  defaultValue?: MaybeMultipleModelValue<T, M>
  /** Whether multiple options can be selected or not. */
  multiple?: M & boolean
  class?: any
  ui?: PartialString<typeof tree.slots>
}

export type TreeEmits<T, M extends boolean = false> = Omit<TreeRootEmits, 'update:modelValue'> & {
  'update:modelValue': [payload: MaybeMultipleModelValue<T, M>]
}

type SlotProps<T> = (props: { item: T, index: number, level: number, expanded: boolean, selected: boolean }) => any

export type TreeSlots<T extends { slot?: string }> = {
  'item': SlotProps<T>
  'item-leading': SlotProps<T>
  'item-label': SlotProps<T>
  'item-trailing': SlotProps<T>
} & DynamicSlots<T, SlotProps<T>>
</script>

<script setup lang="ts" generic="T extends TreeItem, M extends boolean = false, K extends SelectItemKey<T> | undefined = undefined">
import { computed } from 'vue'
import type { PropType } from 'vue'
import { TreeRoot, TreeItem, useForwardPropsEmits } from 'reka-ui'
import { reactivePick, createReusableTemplate } from '@vueuse/core'
import { get } from '../utils'
import UIcon from './Icon.vue'

const props = withDefaults(defineProps<TreeProps<T, M, K>>(), {
  labelKey: 'label' as never,
  valueKey: 'value' as never
})
const emits = defineEmits<TreeEmits<T, M>>()
const slots = defineSlots<TreeSlots<T>>()

const rootProps = useForwardPropsEmits(reactivePick(props, 'as', 'modelValue', 'defaultValue', 'items', 'multiple', 'expanded', 'disabled', 'propagateSelect'), emits)

const [DefineTreeTemplate, ReuseTreeTemplate] = createReusableTemplate<{ items?: T[], level: number }>({
  props: {
    items: Array as PropType<T[]>,
    level: Number
  }
})

const ui = computed(() => tree({
  color: props.color,
  size: props.size
}))

function getItemLabel(item: T) {
  return get(item, props.labelKey as string)
}

function getItemValue(item?: T) {
  return get(item, props.valueKey as string) ?? get(item, props.labelKey as string)
}

function getDefaultOpenedItems(item: T): string[] {
  const currentItem = item.defaultExpanded ? getItemValue(item) : null
  const childItems = item.children?.flatMap(child => getDefaultOpenedItems(child as T)) ?? []

  return [currentItem, ...childItems].filter(Boolean) as string[]
}

const defaultExpanded = computed(() => props.defaultExpanded ?? props.items?.flatMap(getDefaultOpenedItems))
</script>

<!-- eslint-disable vue/no-template-shadow -->
<template>
  <DefineTreeTemplate v-slot="{ items, level }">
    <li
      v-for="(item, index) in items"
      :key="`${level}-${index}`"
      :class="level > 0 ? ui.itemWithChildren({ class: props.ui?.itemWithChildren }) : ui.item({ class: props.ui?.item })"
    >
      <TreeItem
        v-slot="{ isExpanded, isSelected }"
        as-child
        :level="level"
        :value="item"
        @toggle="item.onToggle"
        @select="item.onSelect"
      >
        <button :disabled="item.disabled || disabled" :class="ui.link({ class: props.ui?.link, selected: isSelected, disabled: item.disabled || disabled })">
          <slot :name="item.slot || 'item'" v-bind="{ item, index, level, expanded: isExpanded, selected: isSelected }">
            <slot :name="item.slot ? `${item.slot}-leading`: 'item-leading'" v-bind="{ item, index, level, expanded: isExpanded, selected: isSelected }">
              <UIcon
                v-if="item.icon"
                :name="item.icon"
                :class="ui.linkLeadingIcon({ class: props.ui?.linkLeadingIcon })"
              />
              <UIcon
                v-else-if="item.children?.length"
                :name="isExpanded ? (expandedIcon ?? appConfig.ui.icons.folderOpen) : (collapsedIcon ?? appConfig.ui.icons.folder)"
                :class="ui.linkLeadingIcon({ class: props.ui?.linkLeadingIcon })"
              />
            </slot>

            <span v-if="getItemLabel(item) || !!slots[item.slot ? `${item.slot}-label`: 'item-label']" :class="ui.linkLabel({ class: props.ui?.linkLabel })">
              <slot :name="item.slot ? `${item.slot}-label`: 'item-label'" v-bind="{ item, index, level, expanded: isExpanded, selected: isSelected }">
                {{ getItemLabel(item) }}
              </slot>
            </span>

            <span v-if="item.trailingIcon || item.children?.length || !!slots[item.slot ? `${item.slot}-trailing`: 'item-trailing']" :class="ui.linkTrailing({ class: props.ui?.linkTrailing })">
              <slot :name="item.slot ? `${item.slot}-trailing`: 'item-trailing'" v-bind="{ item, index, level, expanded: isExpanded, selected: isSelected }">
                <UIcon v-if="item.trailingIcon" :name="item.trailingIcon" :class="ui.linkTrailingIcon({ class: props.ui?.linkTrailingIcon })" />
                <UIcon v-else-if="item.children?.length" :name="trailingIcon ?? appConfig.ui.icons.chevronDown" :class="ui.linkTrailingIcon({ class: props.ui?.linkTrailingIcon })" />
              </slot>
            </span>
          </slot>
        </button>

        <ul v-if="item.children?.length && isExpanded" :class="ui.listWithChildren({ class: props.ui?.listWithChildren })">
          <ReuseTreeTemplate :items="(item.children as T[])" :level="level + 1" />
        </ul>
      </TreeItem>
    </li>
  </DefineTreeTemplate>

  <TreeRoot
    v-bind="rootProps"
    :class="ui.root({ class: [props.class, props.ui?.root] })"
    :get-key="getItemValue"
    :default-expanded="defaultExpanded"
    :selection-behavior="selectionBehavior"
  >
    <ReuseTreeTemplate :items="items" :level="0" />
  </TreeRoot>
</template>
