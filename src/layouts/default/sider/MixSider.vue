<template>
  <SiderWrapper
    v-click-outside="handleClickOutside"
    :class="[
      prefixCls,
      {
        mini: getCollapsed,
      },
    ]"
    :width="getMixSideWidth"
    v-bind="getMenuEvents"
  >
    <template #header>
      <AppLogo :class="`${prefixCls}-logo`" :show-title="false" isStatic />
    </template>

    <template #default>
      <n-menu
        :indent="0"
        :options="menuModules"
        :inverted="getMenuInverted"
        key-field="path"
        :class="`${prefixCls}-menu`"
        v-model:value="activePath"
        :render-icon="() => null"
        :render-label="renderMenuLabel"
      />
    </template>

    <template #secondaryMenu>
      <SiderWrapper
        :class="`${prefixCls}-secondary-menu_wrapper`"
        ref="sideRef"
        :style="getMenuStyle"
        :width="openSecMenu ? getMenuWidth : 0"
        :trigger="false"
      >
        <template #header>
          <div class="header_wrapper">
            <span class="title font-segoe"> {{ title }}</span>
            <Icon
              :size="16"
              :icon="getMixSideFixed ? 'ri:pushpin-2-fill' : 'ri:pushpin-2-line'"
              :color="getMenuInverted || getIsDarkMode ? 'white' : undefined"
              class="pushpin"
              @click="handleFixedMenu"
            />
          </div>
        </template>
        <SecMenu
          :inverted="getMenuInverted"
          :splitType="MenuSplitTypeEnum.SECONDARY"
          :parent-path="activePath"
          global-split
          no-collapsed
        />
      </SiderWrapper>
    </template>
  </SiderWrapper>
</template>

<script lang="ts">
  import type { CSSProperties } from 'vue';
  import type { RouteLocationNormalized } from 'vue-router';
  import type { Menu } from '/@/router/types';

  import { reactive, nextTick } from 'vue';
  import { unrefElement, MaybeElementRef } from '@vueuse/core';
  import { computed, onMounted, ref, unref, watch, h } from 'vue';
  import { useRootSetting } from '/@/composables/setting/useRootSetting';
  import { useMenuSetting } from '/@/composables/setting/useMenuSetting';
  import { usePermissionStore } from '/@/store/modules/permission';
  import { useGlobSetting } from '/@/composables/setting';
  import { useDesign } from '/@/composables/web/useDesign';
  import { useI18n } from '/@/composables/web/useI18n';
  import { useGo } from '/@/composables/web/usePage';

  import { SIDE_BAR_MINI_WIDTH, SIDE_BAR_SHOW_TIT_MINI_WIDTH } from '/@/enums/appEnum';
  import { MenuSplitTypeEnum } from '/@/enums/menuEnum';
  import clickOutside from '/@/directives/clickOutside';
  import { getChildrenMenus, getCurrentParentPath, getShallowMenus } from '/@/router/menus';
  import { listenerRouteChange } from '/@/logics/mitt/routeChange';
  import Icon from '/@/components/Icon/Icon.vue';

  import SiderWrapper from './SiderWrapper.vue';
  import SecMenu from '../menu/index.vue';

  export default {
    name: 'LayoutMixSider',
    directives: {
      clickOutside,
    },
  };
</script>
<script setup lang="ts">
  const menuModules = ref<Menu[]>();
  const activePath = ref('');
  const childrenMenus = ref<Menu[]>();
  const openSecMenu = ref(false);
  const sideRef: MaybeElementRef = ref();
  const secMenuBorder = reactive<CSSProperties>({
    borderLeftColor: 'transparent',
    borderRightColor: 'transparent',
    borderWidth: undefined,
  });
  const currentRoute = ref<Nullable<RouteLocationNormalized>>(null);

  const { prefixCls } = useDesign('layout-mix-sider');
  const go = useGo();
  const { t } = useI18n();

  const { getIsDarkMode } = useRootSetting();

  const {
    getMenuWidth,
    getCloseMixSidebarOnChange,
    getMixSideTrigger,
    getMenuInverted,
    getMixSideFixed,
    mixSideHasChildrenRef,
    setMenuSetting,
    getIsMixSidebar,
    getCollapsed,
  } = useMenuSetting();

  const { title } = useGlobSetting();
  const permissionStore = usePermissionStore();

  const getMenuStyle = computed((): CSSProperties => {
    return {
      left: unref(getMixSideFixed) ? 0 : `${unref(getMixSideWidth)}px`,
      position: unref(getMixSideFixed) ? 'static' : 'absolute',
      ...secMenuBorder,
    };
  });

  const getTitleColor = computed(() => {
    return unref(getIsDarkMode) || unref(getMenuInverted) ? 'white' : 'var(--app-primary-color)';
  });

  watch(getMixSideFixed, async (isFixed) => {
    const el = unrefElement(sideRef);
    if (!el) return;
    if (isFixed) {
      el.style.transitionProperty = '';
    } else {
      await nextTick();
      const propertyStr = getComputedStyle(el).transitionProperty;
      el.style.transitionProperty = `${propertyStr}, left`;
    }
  });

  watch(
    [getMixSideFixed, openSecMenu],
    ([isFixed, isOpen]) => {
      secMenuBorder.borderLeftColor = isFixed
        ? 'transparent'
        : isOpen
        ? unref(getMenuInverted) && !unref(getIsDarkMode)
          ? 'rgb(18, 41, 64)'
          : 'var(--mix-sider-border-color)'
        : 'transparent';
      secMenuBorder.borderRightColor = isFixed
        ? 'transparent'
        : isOpen
        ? 'var(--mix-sider-border-color)'
        : 'transparent';
      secMenuBorder.borderWidth = !isOpen ? 0 : undefined;
    },
    { immediate: true }
  );

  watch([childrenMenus, getMixSideFixed], ([child, isFixed]) => {
    mixSideHasChildrenRef.value = !!(child && child?.length > 0);
    const isOpen = isFixed && unref(mixSideHasChildrenRef);
    if (isOpen) {
      openSecMenu.value = isOpen;
    }
  });

  const getMixSideWidth = computed(() => {
    return unref(getCollapsed) ? SIDE_BAR_MINI_WIDTH : SIDE_BAR_SHOW_TIT_MINI_WIDTH;
  });

  const getMenuEvents = computed(() => {
    return !unref(getMixSideFixed)
      ? {
          onMouseleave: handleClickOutside,
        }
      : {};
  });

  onMounted(async () => {
    menuModules.value = await getShallowMenus();
  });

  // Menu changes
  watch(
    [() => permissionStore.getLastBuildMenuTime, () => permissionStore.getBackMenuList],
    async () => {
      menuModules.value = await getShallowMenus();
    },
    {
      immediate: true,
    }
  );

  listenerRouteChange((route) => {
    currentRoute.value = route;
    setActive(true);
    if (unref(getCloseMixSidebarOnChange)) {
      closeMenu();
    }
  });

  // Process module menu click
  async function handleModuleClick(path: string, hover = false) {
    const children = await getChildrenMenus(path);
    if (unref(activePath) === path) {
      if (!hover) {
        if (!unref(openSecMenu)) {
          openSecMenu.value = true;
        } else {
          closeMenu();
        }
      } else {
        if (!unref(openSecMenu)) {
          openSecMenu.value = true;
        }
      }
      if (!unref(openSecMenu)) {
        setActive();
      }
    } else {
      openSecMenu.value = true;
      activePath.value = path;
    }

    if (!children || children.length === 0) {
      if (!hover) go(path);
      childrenMenus.value = [];
      closeMenu();
      return;
    }
    childrenMenus.value = children;
  }

  // Set the currently root menu and secondaryMenu
  async function setActive(setChildren = false) {
    const path = currentRoute.value?.path;
    if (!path) return;
    activePath.value = await getCurrentParentPath(path);
    if (unref(getIsMixSidebar)) {
      const modules = menuModules.value || [];
      const activeMenu = modules.find((item) => item.path === unref(activePath));
      const p = activeMenu?.path;
      if (p) {
        const children = await getChildrenMenus(p);
        if (setChildren) {
          childrenMenus.value = children;

          if (unref(getMixSideFixed)) {
            openSecMenu.value = children.length > 0;
          }
        }
        if (children.length === 0) {
          childrenMenus.value = [];
        }
      }
    }
  }

  function handleClickOutside() {
    setActive(true);
    closeMenu();
  }

  function getItemEvents(item: Menu) {
    if (unref(getMixSideTrigger) === 'hover') {
      return {
        onMouseenter: () => handleModuleClick(item.path, true),
        onClick: async () => {
          const children = await getChildrenMenus(item.path);
          if (item.path && children && children.length === 0) go(item.path);
        },
      };
    }
    return {
      onClick: () => handleModuleClick(item.path),
    };
  }

  function handleFixedMenu() {
    setMenuSetting({
      mixSideFixed: !unref(getMixSideFixed),
    });
  }

  function closeMenu() {
    const children = unref(childrenMenus);
    if (unref(getMixSideFixed)) {
      if (!children || children.length === 0) {
        openSecMenu.value = false;
      }
    } else {
      openSecMenu.value = false;
    }
  }

  const renderMenuLabel = (item) => {
    return h('div', { class: 'item_wrapper', ...getItemEvents(item) }, [
      h(Icon, {
        useTranstion: true,
        size: getCollapsed.value ? 18 : 22,
        icon: item.icon || (item.meta && item.meta.icon),
      }),
      h(
        'p',
        {
          class: 'item_label',
        },
        t(item?.label ?? '')
      ),
    ]);
  };
</script>

<style lang="less">
  @prefix-cls: ~'@{namespace}-layout-mix-sider';

  .@{prefix-cls} {
    @header-height: 60px;
    &-logo {
      display: flex;
      justify-content: center;
      align-items: center;
      height: @header-height;
    }
    &-secondary-menu_wrapper {
      z-index: 2;
      overflow: hidden;
      border-left-width: 1px;
      .header_wrapper {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 0 12px;
        height: @header-height;
        .title {
          color: v-bind(getTitleColor);
          font-size: 18px;
          white-space: nowrap;
          font-weight: 600;
        }
      }
    }
    &-menu {
      .n-menu-item {
        height: auto;
        transition: background-color var(--app-transition-duration) var(--n-bezier),
          height var(--app-transition-duration) var(--n-bezier);
        &::before {
          left: 4px;
          right: 4px;
        }
        .n-menu-item-content {
          padding-right: 0;
          .item_wrapper {
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 12px 0;
            transition: padding var(--app-transition-duration) var(--app-bezier);
            p {
              font-size: 12px;
              will-change: transform;
              transition-property: opacity, transform, height, margin;
              transition-duration: var(--app-transition-duration);
              transition-timing-function: var(--app-bezier);
              height: 20px;
              margin-top: 3px;
              .mini & {
                opacity: 0;
                height: 0;
                margin-top: 0;
                transform: scale(0.6);
              }
            }
          }
        }
      }
    }
  }
</style>
