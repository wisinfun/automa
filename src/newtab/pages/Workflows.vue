<template>
  <div class="container pt-8 pb-4">
    <h1 class="text-2xl font-semibold mb-6 capitalize">
      {{ t('common.workflow', 2) }}
    </h1>
    <div class="flex items-center space-x-4">
      <ui-input
        id="search-input"
        v-model="state.query"
        :placeholder="`${t(`common.search`)}... (${
          shortcut['action:search'].readable
        })`"
        prepend-icon="riSearch2Line"
        class="flex-1"
      />
      <div class="flex items-center workflow-sort">
        <ui-button
          icon
          class="rounded-r-none border-gray-300 dark:border-gray-700 border-r"
          @click="state.sortOrder = state.sortOrder === 'asc' ? 'desc' : 'asc'"
        >
          <v-remixicon
            :name="state.sortOrder === 'asc' ? 'riSortAsc' : 'riSortDesc'"
          />
        </ui-button>
        <ui-select v-model="state.sortBy" :placeholder="t('sort.sortBy')">
          <option v-for="sort in sorts" :key="sort" :value="sort">
            {{ t(`sort.${sort}`) }}
          </option>
        </ui-select>
      </div>
      <ui-button
        tag="a"
        href="https://automa.site/workflows"
        target="_blank"
        class="inline-block relative"
        @click="browseWorkflow"
      >
        <span
          v-if="state.highlightBrowse"
          class="flex h-3 w-3 absolute top-0 right-0 -mr-1 -mt-1"
        >
          <span
            class="animate-ping absolute inline-flex h-full w-full rounded-full bg-primary opacity-75"
          ></span>
          <span
            class="relative inline-flex rounded-full h-3 w-3 bg-blue-600"
          ></span>
        </span>
        <v-remixicon name="riCompass3Line" class="mr-2 -ml-1" />
        {{ t('workflow.browse') }}
      </ui-button>
      <ui-button @click="importWorkflow({ multiple: true })">
        <v-remixicon name="riUploadLine" class="mr-2 -ml-1" />
        {{ t('workflow.import') }}
      </ui-button>
      <div class="flex">
        <ui-button
          :title="shortcut['action:new'].readable"
          variant="accent"
          class="border-r rounded-r-none"
          @click="newWorkflow"
        >
          {{ t('workflow.new') }}
        </ui-button>
        <ui-popover>
          <template #trigger>
            <ui-button icon class="rounded-l-none" variant="accent">
              <v-remixicon name="riArrowLeftSLine" rotate="-90" />
            </ui-button>
          </template>
          <ui-list>
            <ui-list-item
              v-close-popover
              class="cursor-pointer"
              @click="addHostWorkflow"
            >
              {{ t('workflow.host.add') }}
            </ui-list-item>
          </ui-list>
        </ui-popover>
      </div>
    </div>
    <ui-tabs
      v-model="state.activeTab"
      class="mt-4 space-x-2"
      type="fill"
      style="display: inline-flex; background-color: transparent; padding: 0"
    >
      <ui-tab value="local">
        {{ t('workflow.type.local') }}
      </ui-tab>
      <ui-tab v-if="store.state.user" value="shared">
        {{ t('workflow.type.shared') }}
      </ui-tab>
      <ui-tab v-if="workflowHosts.length > 0" value="host">
        {{ t('workflow.type.host') }}
      </ui-tab>
    </ui-tabs>
    <ui-tab-panels v-model="state.activeTab" class="mt-6">
      <ui-tab-panel value="shared">
        <div class="grid gap-4 grid-cols-4 2xl:grid-cols-5">
          <shared-card
            v-for="workflow in sharedWorkflows"
            :key="workflow.id"
            :data="workflow"
            :show-details="false"
            @click="$router.push(`/workflows/${$event.id}?shared=true`)"
          />
        </div>
      </ui-tab-panel>
      <ui-tab-panel value="host">
        <div class="grid gap-4 grid-cols-4 2xl:grid-cols-5">
          <shared-card
            v-for="workflow in workflowHosts"
            :key="workflow.hostId"
            :data="workflow"
            :menu="workflowHostMenu"
            @click="$router.push(`/workflows/${$event.hostId}/host`)"
            @menuSelected="deleteWorkflowHost(workflow)"
          />
        </div>
      </ui-tab-panel>
      <ui-tab-panel value="local">
        <div v-if="Workflow.all().length === 0" class="py-12 flex items-center">
          <img src="@/assets/svg/alien.svg" class="w-96" />
          <div class="ml-4">
            <h1 class="text-2xl font-semibold max-w-md mb-6">
              {{ t('message.empty') }}
            </h1>
            <ui-button variant="accent" @click="newWorkflow">
              {{ t('workflow.new') }}
            </ui-button>
          </div>
        </div>
        <div v-else class="grid gap-4 grid-cols-4 2xl:grid-cols-5">
          <shared-card
            v-for="workflow in workflows"
            :key="workflow.id"
            :data="workflow"
            @click="$router.push(`/workflows/${$event.id}`)"
          >
            <template #header>
              <div class="flex items-center mb-4">
                <template v-if="!workflow.isDisabled">
                  <ui-img
                    v-if="workflow.icon.startsWith('http')"
                    :src="workflow.icon"
                    class="rounded-lg overflow-hidden"
                    style="height: 40px; width: 40px"
                    alt="Can not display"
                  />
                  <span v-else class="p-2 rounded-lg bg-box-transparent">
                    <v-remixicon :name="workflow.icon" />
                  </span>
                </template>
                <p v-else class="py-2">{{ t('common.disabled') }}</p>
                <div class="flex-grow"></div>
                <button
                  v-if="!workflow.isDisabled"
                  class="invisible group-hover:visible"
                  @click="executeWorkflow(workflow)"
                >
                  <v-remixicon name="riPlayLine" />
                </button>
                <v-remixicon
                  v-if="workflow.isProtected"
                  name="riShieldKeyholeLine"
                  class="text-green-600 dark:text-green-400 ml-2"
                />
                <ui-popover v-if="!workflow.isProtected" class="h-6 ml-2">
                  <template #trigger>
                    <button>
                      <v-remixicon name="riMoreLine" />
                    </button>
                  </template>
                  <ui-list class="space-y-1" style="min-width: 150px">
                    <ui-list-item
                      class="cursor-pointer"
                      @click="
                        updateWorkflow(workflow.id, {
                          isDisabled: !workflow.isDisabled,
                        })
                      "
                    >
                      <v-remixicon name="riToggleLine" class="mr-2 -ml-1" />
                      <span class="capitalize">
                        {{
                          t(
                            `common.${
                              workflow.isDisabled ? 'enable' : 'disable'
                            }`
                          )
                        }}
                      </span>
                    </ui-list-item>
                    <ui-list-item
                      v-for="item in menu"
                      :key="item.id"
                      v-close-popover
                      class="cursor-pointer"
                      @click="menuHandlers[item.id](workflow)"
                    >
                      <v-remixicon :name="item.icon" class="mr-2 -ml-1" />
                      <span class="capitalize">{{ item.name }}</span>
                    </ui-list-item>
                  </ui-list>
                </ui-popover>
              </div>
            </template>
            <template #footer-content>
              <v-remixicon
                v-if="sharedWorkflows[workflow.id]"
                v-tooltip="
                  t('workflow.share.sharedAs', {
                    name: sharedWorkflows[workflow.id]?.name.slice(0, 64),
                  })
                "
                name="riShareLine"
                size="20"
                class="ml-2"
              />
              <v-remixicon
                v-if="hostWorkflows[workflow.id]"
                v-tooltip="t('workflow.host.title')"
                name="riBaseStationLine"
                size="20"
                class="ml-2"
              />
            </template>
          </shared-card>
        </div>
      </ui-tab-panel>
    </ui-tab-panels>
    <ui-modal v-model="workflowModal.show" title="Workflow">
      <ui-input
        v-model="workflowModal.name"
        :placeholder="t('common.name')"
        autofocus
        class="w-full mb-4"
        @keyup.enter="handleWorkflowModal"
      />
      <ui-textarea
        v-model="workflowModal.description"
        :placeholder="t('common.description')"
        height="165px"
        class="w-full dark:text-gray-200"
        max="300"
      />
      <p class="mb-6 text-right text-gray-600 dark:text-gray-200">
        {{ workflowModal.description.length }}/300
      </p>
      <div class="space-x-2 flex">
        <ui-button class="w-full" @click="workflowModal.show = false">
          {{ t('common.cancel') }}
        </ui-button>
        <ui-button variant="accent" class="w-full" @click="handleWorkflowModal">
          {{
            workflowModal.type === 'update'
              ? t('common.update')
              : t('common.add')
          }}
        </ui-button>
      </div>
    </ui-modal>
  </div>
</template>
<script setup>
import { computed, shallowReactive, watch } from 'vue';
import { useStore } from 'vuex';
import { useI18n } from 'vue-i18n';
import { useToast } from 'vue-toastification';
import browser from 'webextension-polyfill';
import { useDialog } from '@/composable/dialog';
import { useShortcut } from '@/composable/shortcut';
import { sendMessage } from '@/utils/message';
import { fetchApi } from '@/utils/api';
import { exportWorkflow, importWorkflow } from '@/utils/workflow-data';
import {
  registerWorkflowTrigger,
  cleanWorkflowTriggers,
} from '@/utils/workflow-trigger';
import { findTriggerBlock, isWhitespace } from '@/utils/helper';
import SharedCard from '@/components/newtab/shared/SharedCard.vue';
import Workflow from '@/models/workflow';

const { t } = useI18n();
const toast = useToast();
const store = useStore();
const dialog = useDialog();

const sorts = ['name', 'createdAt'];
const menu = [
  { id: 'duplicate', name: t('common.duplicate'), icon: 'riFileCopyLine' },
  { id: 'export', name: t('common.export'), icon: 'riDownloadLine' },
  { id: 'rename', name: t('common.rename'), icon: 'riPencilLine' },
  { id: 'delete', name: t('common.delete'), icon: 'riDeleteBin7Line' },
];
const workflowHostMenu = [
  { id: 'delete', name: t('common.delete'), icon: 'riDeleteBin7Line' },
];

const savedSorts = JSON.parse(localStorage.getItem('workflow-sorts') || '{}');
const state = shallowReactive({
  query: '',
  activeTab: 'local',
  sortBy: savedSorts.sortBy || 'createdAt',
  sortOrder: savedSorts.sortOrder || 'desc',
  highlightBrowse: !localStorage.getItem('first-time-browse'),
});
const workflowModal = shallowReactive({
  name: '',
  type: 'update',
  description: '',
});

const hostWorkflows = computed(() => store.state.hostWorkflows || {});
const workflowHosts = computed(() => Object.values(store.state.workflowHosts));
const sharedWorkflows = computed(() => store.state.sharedWorkflows || {});
const workflows = computed(() =>
  Workflow.query()
    .where(({ name }) =>
      name.toLocaleLowerCase().includes(state.query.toLocaleLowerCase())
    )
    .orderBy(state.sortBy, state.sortOrder)
    .get()
);

async function deleteWorkflowHost(workflow) {
  dialog.confirm({
    title: t('workflow.delete'),
    okVariant: 'danger',
    body: t('message.delete', { name: workflow.name }),
    onConfirm: async () => {
      try {
        store.commit('deleteStateNested', `workflowHosts.${workflow.hostId}`);

        await browser.storage.local.set({
          workflowHosts: store.state.sharedWorkflows,
        });
        await cleanWorkflowTriggers(workflow.hostId);
      } catch (error) {
        console.error(error);
      }
    },
  });
}
function addHostWorkflow() {
  dialog.prompt({
    async: true,
    inputType: 'url',
    okText: t('common.add'),
    title: t('workflow.host.add'),
    label: t('workflow.host.id'),
    placeholder: 'abcd123',
    onConfirm: async (value) => {
      try {
        if (isWhitespace(value)) return false;

        let length = 0;
        let isItsOwn = false;
        let isHostExist = false;
        const hostId = value.replace(/\s/g, '');

        workflowHosts.value.forEach((host) => {
          if (hostId === host.hostId) isHostExist = true;

          length += 1;
        });

        if (!store.state.user && length >= 3) {
          toast.error(t('message.rateExceeded'));
          return false;
        }

        Object.values(store.state.hostWorkflows).forEach((host) => {
          if (hostId === host.hostId) isItsOwn = true;
        });

        if (isHostExist || isItsOwn) {
          toast.error(t('workflow.host.messages.hostExist'));
          return false;
        }

        const response = await fetchApi('/host', {
          method: 'POST',
          body: JSON.stringify({ length, hostId }),
        });
        const result = await response.json();

        if (response.status !== 200) {
          const error = new Error(response.statusText);
          error.data = result.data;

          throw error;
        }

        if (result === null) {
          toast.error(t('workflow.host.messages.notFound', { id: hostId }));
          return false;
        }

        result.hostId = hostId;
        result.createdAt = Date.now();

        store.commit('updateStateNested', {
          value: result,
          path: `workflowHosts.${hostId}`,
        });

        const triggerBlock = findTriggerBlock(result.drawflow);
        await registerWorkflowTrigger(hostId, triggerBlock);

        result.drawflow = JSON.stringify(result.drawflow);

        let { workflowHosts: storageHosts } = await browser.storage.local.get(
          'workflowHosts'
        );
        (storageHosts = storageHosts || {})[hostId] = result;

        await browser.storage.local.set({ workflowHosts: storageHosts });

        return true;
      } catch (error) {
        console.error(error);

        toast.error(
          error.data?.show ? error.message : t('message.somethingWrong')
        );

        return false;
      }
    },
  });
}
function browseWorkflow() {
  state.highlightBrowse = false;
  localStorage.setItem('first-time-browse', false);
}
function executeWorkflow(workflow) {
  sendMessage('workflow:execute', workflow, 'background');
}
function updateWorkflow(id, data) {
  Workflow.update({
    where: id,
    data,
  });
}
function newWorkflow() {
  Object.assign(workflowModal, {
    name: '',
    show: true,
    type: 'add',
    description: '',
  });
}
function deleteWorkflow({ name, id }) {
  dialog.confirm({
    title: t('workflow.delete'),
    okVariant: 'danger',
    body: t('message.delete', { name }),
    onConfirm: () => {
      Workflow.delete(id);
    },
  });
}
function renameWorkflow({ id, name, description }) {
  Object.assign(workflowModal, {
    id,
    name,
    description,
    show: true,
    type: 'update',
  });
}
async function handleWorkflowModal() {
  try {
    if (workflowModal.type === 'add') {
      await Workflow.insert({
        data: {
          createdAt: Date.now(),
          name: workflowModal.name || 'Unnamed',
          description: workflowModal.description,
        },
      });
    } else {
      await Workflow.update({
        where: workflowModal.id,
        data: {
          name: workflowModal.name,
          description: workflowModal.description,
        },
      });
    }

    Object.assign(workflowModal, {
      name: '',
      show: false,
      description: '',
    });
  } catch (error) {
    console.error(error);
  }
}
function duplicateWorkflow(workflow) {
  const copyWorkflow = { ...workflow, createdAt: Date.now() };
  const delKeys = ['$id', 'data', 'id', 'isDisabled'];

  delKeys.forEach((key) => {
    delete copyWorkflow[key];
  });

  Workflow.insert({ data: copyWorkflow });
}

const shortcut = useShortcut(['action:search', 'action:new'], ({ id }) => {
  if (id === 'action:search') {
    const searchInput = document.querySelector('#search-input input');
    searchInput?.focus();
  } else {
    newWorkflow();
  }
});
const menuHandlers = {
  export: exportWorkflow,
  rename: renameWorkflow,
  delete: deleteWorkflow,
  duplicate: duplicateWorkflow,
};

watch(
  () => [state.sortOrder, state.sortBy],
  ([sortOrder, sortBy]) => {
    localStorage.setItem(
      'workflow-sorts',
      JSON.stringify({ sortOrder, sortBy })
    );
  }
);
</script>
<style>
.workflow-sort select {
  @apply rounded-l-none !important;
}
</style>
