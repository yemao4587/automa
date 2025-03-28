<template>
  <div
    v-if="state.active"
    class="fixed top-0 left-0 h-full w-full bg-black bg-opacity-50 p-4 text-black"
    style="z-index: 99999999"
    @click.self="state.active = false"
  >
    <ui-card
      id="workflows-container"
      class="absolute w-full max-w-2xl"
      padding="p-0"
      style="left: 50%; top: 50px; transform: translateX(-50%)"
    >
      <div class="p-4">
        <label
          class="bg-input flex h-12 items-center rounded-lg px-2 ring-accent transition focus-within:ring-2"
        >
          <img :src="logoUrl" class="h-8 w-8" />
          <input
            ref="inputRef"
            type="text"
            class="h-full flex-1 rounded-lg bg-transparent px-2 focus:ring-0"
            :placeholder="
              paramsState.active
                ? paramsState.workflow.name
                : 'Search workflows...'
            "
            @input="onInput"
            @keydown="onInputKeydown"
          />
          <template v-for="key in state.shortcutKeys" :key="key">
            <span
              class="bg-box-transparent ml-1 inline-block rounded-md border-2 border-gray-300 p-1 text-center text-xs font-semibold capitalize text-gray-600"
              style="min-width: 29px; font-family: inherit"
            >
              {{ getReadableShortcut(key) }}
            </span>
          </template>
        </label>
      </div>
      <div
        class="scroll workflows-list overflow-auto px-4 pb-4"
        style="max-height: calc(100vh - 200px)"
      >
        <div v-if="!state.retrieved" class="mb-2 text-center">
          <ui-spinner color="text-accent" />
        </div>
        <template v-else>
          <div v-if="paramsState.active">
            <ul class="space-y-4 divide-y">
              <li
                v-for="(param, paramIdx) in paramsState.items"
                :key="paramIdx"
              >
                <component
                  :is="paramsList[param.type].valueComp"
                  v-if="paramsList[param.type]"
                  v-model="param.value"
                  :label="param.name"
                  :param-data="param"
                  class="w-full"
                />
                <ui-input
                  v-else
                  v-model="param.value"
                  :type="param.inputType || param.type"
                  :label="param.name"
                  :placeholder="param.placeholder"
                  class="w-full"
                  @keyup.enter="runWorkflow(index, workflow)"
                />
                <p
                  v-if="param.description"
                  title="Description"
                  class="ml-1 text-sm"
                >
                  {{ param.description }}
                </p>
              </li>
            </ul>
          </div>
          <template v-else>
            <p
              v-if="state.query && workflows.length === 0"
              class="text-center text-gray-600"
            >
              Can't find workflows
            </p>
            <ui-list v-else class="space-y-1">
              <ui-list-item
                v-for="(workflow, index) in workflows"
                :id="`list-item-${index}`"
                :key="workflow.id"
                :active="index === state.selectedIndex"
                small
                color="bg-box-transparent list-item-active"
                class="group cursor-pointer"
                @mouseenter="state.selectedIndex = index"
                @click="executeWorkflow(workflow)"
              >
                <div class="w-8">
                  <img
                    v-if="workflow.icon?.startsWith('http')"
                    :src="workflow.icon"
                    class="overflow-hidden rounded-lg"
                    style="height: 26px; width: 26px"
                    alt="Can not display"
                  />
                  <v-remixicon
                    v-else
                    :name="workflow.icon || 'riGlobalLine'"
                    size="26"
                  />
                </div>
                <div class="mx-2 flex-1 overflow-hidden">
                  <p class="text-overflow">
                    {{ workflow.name }}
                  </p>
                  <p class="text-overflow leading-tight text-gray-500">
                    {{ workflow.description }}
                  </p>
                </div>
                <v-remixicon
                  name="riArrowGoForwardLine"
                  class="invisible text-gray-600 group-hover:visible"
                  size="20"
                  rotate="180"
                />
              </ui-list-item>
            </ui-list>
          </template>
        </template>
      </div>
      <div class="flex items-center px-4 py-2">
        <div v-if="paramsState.active" class="pl-2 text-gray-500">
          <div class="flex items-center">
            <p class="mr-4">
              {{ paramsState.workflow.description }}
            </p>
            <p>
              Press
              <span
                class="bg-box-transparent ml-1 inline-block rounded-md border-2 border-gray-300 p-1 text-center text-xs font-semibold text-gray-600"
              >
                Escape
              </span>
              to cancel
            </p>
          </div>
        </div>
        <p
          v-else
          class="inline-flex cursor-pointer items-center text-gray-600"
          @click="openDashboard"
        >
          Open dashboard
          <v-remixicon
            name="riExternalLinkLine"
            class="ml-1 inline-block"
            size="20"
          />
        </p>
        <div class="grow" />
        <ui-button
          v-if="paramsState.active"
          variant="accent"
          @click="executeWorkflowWithParams"
        >
          Execute
        </ui-button>
      </div>
    </ui-card>
  </div>
</template>
<script setup>
import ParameterInputValue from '@/components/newtab/workflow/edit/Parameter/ParameterInputValue.vue';
import ParameterJsonValue from '@/components/newtab/workflow/edit/Parameter/ParameterJsonValue.vue';
import RendererWorkflowService from '@/service/renderer/RendererWorkflowService';
import { debounce, parseJSON } from '@/utils/helper';
import { sendMessage } from '@/utils/message';
import workflowParameters from '@business/parameters';
import cloneDeep from 'lodash.clonedeep';
import {
  computed,
  inject,
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  shallowReactive,
  watch,
} from 'vue';
import browser from 'webextension-polyfill';

const paramsList = {
  string: {
    id: 'string',
    name: 'Input (string)',
    valueComp: ParameterInputValue,
  },
  json: {
    id: 'json',
    name: 'Input (JSON)',
    valueComp: ParameterJsonValue,
  },
};

const os = navigator.appVersion.indexOf('Mac') !== -1 ? 'mac' : 'win';

const logoUrl = browser.runtime.getURL(
  process.env.NODE_ENV === 'development' ? '/icon-dev-128.png' : '/icon-128.png'
);

const inputRef = ref(null);
const state = shallowReactive({
  query: '',
  active: false,
  workflows: [],
  shortcutKeys: [],
  selectedIndex: -1,
});
const paramsState = reactive({
  items: [],
  workflow: {},
  active: false,
  paramNames: [],
  activeIndex: 0,
  inputtedVal: '',
});

const rootElement = inject('rootElement');

const workflows = computed(() =>
  state.workflows.filter((workflow) =>
    workflow.name.toLocaleLowerCase().includes(state.query.toLocaleLowerCase())
  )
);

function getReadableShortcut(str) {
  const list = {
    option: {
      win: 'alt',
      mac: 'option',
    },
    mod: {
      win: 'ctrl',
      mac: '⌘',
    },
  };
  const regex = /option|mod/g;
  const replacedStr = str.replace(regex, (match) => {
    return list[match][os];
  });

  return replacedStr;
}
function clearParamsState() {
  Object.assign(paramsState, {
    items: [],
    workflow: {},
    active: false,
    activeIndex: 0,
    inputtedVal: '',
  });
}
function sendExecuteCommand(workflow, options = {}) {
  const workflowData = {
    ...workflow,
    includeTabId: true,
    options: { ...options, checkParams: false },
  };
  RendererWorkflowService.executeWorkflow(workflowData);

  state.active = false;
}
function executeWorkflow(workflow) {
  if (!workflow) return;

  let triggerData = workflow.trigger;
  if (!triggerData) {
    const triggerNode = workflow.drawflow?.nodes?.find(
      (node) => node.label === 'trigger'
    );
    triggerData = triggerNode?.data;
  }

  if (triggerData?.parameters?.length > 0) {
    const keys = new Set();
    const params = [];
    triggerData.parameters.forEach((param) => {
      if (keys.has(param.name)) return;

      params.push(param);
      keys.add(param.name);
    });

    const parameters = cloneDeep(triggerData.parameters).map((item) => ({
      ...item,
      value: item.defaultValue,
    }));

    paramsState.workflow = workflow;
    paramsState.items = parameters;

    paramsState.active = true;
  } else {
    sendExecuteCommand(workflow);
  }

  inputRef.value.value = '';
  state.query = '';
  paramsState.inputtedVal = '';
}
function getParamsValues(params) {
  const getParamVal = {
    string: (str) => str,
    number: (num) => (Number.isNaN(+num) ? 0 : +num),
    json: (value) => parseJSON(value, null),
    default: (value) => value,
  };

  return params.reduce((acc, param) => {
    const valueFunc =
      getParamVal[param.type] ||
      paramsList[param.type]?.getValue ||
      getParamVal.default;
    const value = valueFunc(param.value || param.defaultValue);
    acc[param.name] = value;

    return acc;
  }, {});
}
function executeWorkflowWithParams() {
  const variables = getParamsValues(paramsState.items);
  sendExecuteCommand(paramsState.workflow, { data: { variables } });

  clearParamsState();
}
function onKeydown(event) {
  const { ctrlKey, altKey, metaKey, key, shiftKey } = event;

  if (key === 'Escape') {
    if (paramsState.active) {
      clearParamsState();
    } else {
      state.active = false;
    }
    return;
  }

  const shortcuts = window._automaShortcuts;
  if (!shortcuts || shortcuts.length < 1) return;

  const automaShortcut = shortcuts.every((shortcutKey) => {
    if (shortcutKey === 'mod') return ctrlKey || metaKey;
    if (shortcutKey === 'shift') return shiftKey;
    if (shortcutKey === 'option') return altKey;

    return shortcutKey === key.toLowerCase();
  });
  if (automaShortcut) {
    event.preventDefault();
    state.active = true;
    state.shortcutKeys = shortcuts;
  }
}
function onInputKeydown(event) {
  const { key } = event;

  if (key !== 'Escape') {
    event.stopPropagation();
  }

  if (['ArrowDown', 'ArrowUp'].includes(key)) {
    let nextIndex = state.selectedIndex;
    const maxIndex = workflows.value.length - 1;

    if (key === 'ArrowDown') {
      nextIndex += 1;
      if (nextIndex > maxIndex) nextIndex = 0;
    } else if (key === 'ArrowUp') {
      nextIndex -= 1;
      if (nextIndex < 0) nextIndex = maxIndex;
    }

    state.selectedIndex = nextIndex;
    return;
  }

  if (key === 'Enter') {
    if (paramsState.active) return;
    executeWorkflow(workflows.value[state.selectedIndex]);
  }
}
function checkInView(container, element, partial = false) {
  const cTop = container.scrollTop;
  const cBottom = cTop + container.clientHeight;

  const eTop = element.offsetTop;
  const eBottom = eTop + element.clientHeight;

  const isTotal = eTop >= cTop && eBottom <= cBottom;
  const isPartial =
    partial &&
    ((eTop < cTop && eBottom > cTop) || (eBottom > cBottom && eTop < cBottom));

  return isTotal || isPartial;
}
function onInput(event) {
  const { value } = event.target;

  if (paramsState.active) {
    paramsState.inputtedVal = value;
    paramsState.activeIndex = value.split(';').length - 1;
  } else {
    state.query = value;
  }
}
function openDashboard() {
  sendMessage('open:dashboard', '', 'background');
}

watch(inputRef, () => {
  if (!inputRef.value) return;

  inputRef.value.focus();
});
watch(
  () => state.active,
  async () => {
    if (!state.retrieved && state.active) {
      const {
        workflows: localWorkflows,
        workflowHosts,
        teamWorkflows,
      } = await browser.storage.local.get([
        'workflows',
        'workflowHosts',
        'teamWorkflows',
      ]);
      state.workflows = [
        ...Object.values(workflowHosts || {}),
        ...Object.values(localWorkflows || {}),
        ...Object.values(Object.values(teamWorkflows || {})[0] || {}),
      ];

      state.retrieved = true;
    } else if (!state.active) {
      clearParamsState();
      state.query = '';
      state.selectedIndex = -1;
    }
  }
);
watch(
  () => state.selectedIndex,
  debounce((activeIndex) => {
    const container = rootElement.shadowRoot.querySelector(
      '#workflows-container .workflows-list'
    );
    const element = rootElement.shadowRoot.querySelector(
      `#list-item-${activeIndex}`
    );

    if (element && !checkInView(container, element)) {
      element.scrollIntoView({
        block: 'nearest',
        behavior: 'smooth',
      });
    }
  }, 100)
);

window.initPaletteParams = (data) => {
  paramsState.items = data.params;
  paramsState.workflow = data.workflow;
  paramsState.active = true;

  state.active = true;
};

onMounted(() => {
  browser.storage.local.get('automaShortcut').then(({ automaShortcut }) => {
    if (Array.isArray(automaShortcut) && automaShortcut.length < 1) return;

    let keys = ['mod', 'shift', 'e'];
    if (automaShortcut) keys = automaShortcut.split('+');

    state.shortcutKeys = keys;
    window._automaShortcuts = keys;
  });

  window.addEventListener('keydown', onKeydown);
  Object.assign(paramsList, workflowParameters());
});
onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown);
});
</script>
