<template>
  <Wrapper :class="prefixCls" :title="title">
    <NSelect
      v-model:value="valRef"
      class="min-w-110px"
      size="small"
      v-bind="$attrs"
      :options="getOptions"
      @update-value="handleChange"
    />
  </Wrapper>
</template>

<script lang="ts">
  export default {
    inheritAttrs: false,
  };
</script>

<script lang="ts" setup>
  import { PropType, computed, watch } from 'vue';

  import { useDesign } from '/@/composables/web/useDesign';
  import { useI18n } from '/@/composables/web/useI18n';
  import { toWritableRef } from '/@/composables/utilities/toWritableRef';
  import Wrapper from './Wrapper.vue';

  import { baseHandler } from '../handler';
  import { HandlerEnum } from '../enum';

  const props = defineProps({
    event: {
      type: Number as PropType<HandlerEnum>,
    },
    title: {
      type: String,
    },
    val: {
      type: [String, Number] as PropType<string | number>,
    },
    options: {
      type: Array as PropType<LabelValueOptions>,
      default: () => [],
    },
  });

  const { prefixCls } = useDesign('setting-select-item');
  const { t } = useI18n();

  const valRef = toWritableRef(props, 'val');

  const getOptions = computed(() => {
    return props.options.map((item) => {
      return {
        ...item,
        label: t(item.label),
      };
    });
  });

  watch(
    () => props.options,
    (val) => {
      const _list = val.map((item) => item.value);
      if (!_list.includes(valRef.value)) {
        valRef.value = _list.length > 0 ? _list[0] : undefined;
      }
      handleChange(valRef.value);
    }
  );

  function handleChange(e) {
    props.event && baseHandler(props.event, e);
  }
</script>
