<script setup>
import { computed, watch, ref } from 'vue';
import { useSidebarContext } from './provider';
import { useRoute } from 'vue-router';
import Policy from 'dashboard/components/policy.vue';
import SidebarGroupHeader from './SidebarGroupHeader.vue';
import SidebarGroupLeaf from './SidebarGroupLeaf.vue';
import SidebarSubGroup from './SidebarSubGroup.vue';
import SidebarGroupEmptyLeaf from './SidebarGroupEmptyLeaf.vue';

const props = defineProps({
  name: { type: String, required: true },
  label: { type: String, required: true },
  icon: { type: [String, Object, Function], default: null },
  to: { type: Object, default: null },
  activeOn: { type: Array, default: () => [] },
  children: { type: Array, default: undefined },
});

const {
  expandedItem,
  setExpandedItem,
  resolvePath,
  resolvePermissions,
  resolveFeatureFlag,
  isAllowed,
} = useSidebarContext();

const parentEl = ref(null);

const locateLastChild = () => {
  parentEl.value?.querySelectorAll('.child-item').forEach((child, index) => {
    if (index === parentEl.value.querySelectorAll('.child-item').length - 1) {
      child.classList.add('last-child-item');
    }
  });
};

const navigableChildren = computed(() => {
  return props.children?.flatMap(child => child.children || child) || [];
});

const route = useRoute();

const isExpanded = computed(() => expandedItem.value === props.name);
const isExpandable = computed(() => props.children);
const hasChildren = computed(
  () => Array.isArray(props.children) && props.children.length > 0
);

const accessibleItems = computed(() => {
  if (!hasChildren.value) return [];
  return props.children.filter(child => isAllowed(child.to));
});

const hasAccessibleItems = computed(() => {
  // default true so that rendering is not blocked
  if (!hasChildren.value) return true;

  return accessibleItems.value.length > 0;
});

const isActive = computed(() => {
  if (props.to) {
    if (route.path === resolvePath(props.to)) return true;

    return props.activeOn.includes(route.name);
  }

  return false;
});

// We could use the RouterLink isActive too, but our routes are not always
// nested correctly, so we need to check the active state ourselves
// TODO: Audit the routes and fix the nesting and remove this
const activeChild = computed(() => {
  const pathSame = navigableChildren.value.find(
    child => child.to && route.path === resolvePath(child.to)
  );
  if (pathSame) return pathSame;

  const pathSatrtsWith = navigableChildren.value.find(
    child => child.to && route.path.startsWith(resolvePath(child.to))
  );
  if (pathSatrtsWith) return pathSatrtsWith;

  return navigableChildren.value.find(child =>
    child.activeOn?.includes(route.name)
  );
});

const hasActiveChild = computed(() => {
  return activeChild.value !== undefined;
});

watch(expandedItem, locateLastChild, {
  immediate: true,
});
</script>

<!-- eslint-disable-next-line vue/no-root-v-if -->
<template>
  <Policy
    v-if="hasAccessibleItems"
    :permissions="resolvePermissions(to)"
    :feature-flag="resolveFeatureFlag(to)"
    as="li"
    class="text-sm cursor-pointer select-none gap-1 grid"
  >
    <SidebarGroupHeader
      :icon
      :name
      :label
      :to
      :is-active="isActive"
      :has-active-child="hasActiveChild"
      :expandable="hasChildren"
      :is-expanded="isExpanded"
      @toggle="setExpandedItem(name)"
    />
    <ul
      v-if="hasChildren"
      v-show="isExpanded || hasActiveChild"
      ref="parentEl"
      class="list-none m-0 grid sidebar-group-children"
    >
      <template v-for="child in children" :key="child.name">
        <SidebarSubGroup
          v-if="child.children"
          v-bind="child"
          :is-expanded="isExpanded"
          :active-child="activeChild"
        />
        <SidebarGroupLeaf
          v-else
          v-show="isExpanded || activeChild?.name === child.name"
          v-bind="child"
          :active="activeChild?.name === child.name"
        />
      </template>
    </ul>
    <ul v-else-if="isExpandable && isExpanded">
      <SidebarGroupEmptyLeaf />
    </ul>
  </Policy>
</template>
