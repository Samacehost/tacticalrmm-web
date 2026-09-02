<template>
  <div>
    <q-tabs
      v-model="tab"
      dense
      inline-label
      class="text-grey"
      active-color="primary"
      indicator-color="primary"
      align="left"
      narrow-indicator
    >
      <q-tab name="terminal" icon="fas fa-terminal" label="Terminal" />
      <q-tab
        name="filebrowser"
        icon="far fa-folder-open"
        label="File Browser"
      />
      <q-tab
        v-if="$route.query.agentPlatform === 'windows'"
        name="services"
        icon="fas fa-cogs"
        label="Services"
      />
      <q-tab name="processes" icon="fas fa-chart-area" label="Processes" />
      <q-tab
        v-if="$route.query.agentPlatform === 'windows'"
        name="eventlog"
        icon="fas fa-clipboard-list"
        label="Event Log"
      />
      <q-tab v-if="$route.query.agentPlatform === 'windows'" name="registry">
        <div class="flex items-center text-weight-bold text-subtitle2">
          <q-icon :name="`img:${registryIcon}`" size="25px" />
          <span class="q-ml-sm font">Registry</span>
        </div>
      </q-tab>
      <q-tab name="cobrowse" icon="security" label="Co-Browsing (User Chrome)" />
      <q-tab name="winaccess" icon="desktop_windows" label="Background Windows Access" />
    </q-tabs>
    <q-separator />
    <q-tab-panels v-model="tab">
      <q-tab-panel name="terminal" class="q-pa-none">
        <TerminalManager
          v-if="terminalMode === 'new'"
          :agent_id="agent_id"
          :agentPlatform="$route.query.agentPlatform"
          :terminalDefaults="terminalDefaults"
        />
        <iframe
          v-else
          allow="clipboard-read; clipboard-write"
          :src="terminal"
          :style="{
            height: `${$q.screen.height - 30}px`,
            width: `${$q.screen.width}px`,
          }"
        ></iframe>
      </q-tab-panel>
      <q-tab-panel name="processes" class="q-pa-none">
        <ProcessManager :agent_id="agent_id" />
      </q-tab-panel>
      <q-tab-panel
        v-if="$route.query.agentPlatform === 'windows'"
        name="services"
        class="q-pa-none"
      >
        <ServicesManager
          :agent_id="agent_id"
          :agentPlatform="$route.query.agentPlatform"
        />
      </q-tab-panel>
      <q-tab-panel
        v-if="$route.query.agentPlatform === 'windows'"
        name="eventlog"
        class="q-pa-none"
      >
        <EventLogManager
          :agent_id="agent_id"
          :agentPlatform="$route.query.agentPlatform"
        />
      </q-tab-panel>
      <q-tab-panel name="filebrowser" class="q-pa-none">
        <iframe
          allow="clipboard-read; clipboard-write"
          :src="file"
          :style="{
            height: `${$q.screen.height - 30}px`,
            width: `${$q.screen.width}px`,
          }"
        ></iframe>
      </q-tab-panel>
      <q-tab-panel
        v-if="$route.query.agentPlatform === 'windows'"
        name="registry"
        class="q-pa-none"
      >
        <RegistryManager :agent_id="agent_id" />
      </q-tab-panel>

      <!-- Co-Browsing Tab Panel -->
      <q-tab-panel name="cobrowse" class="q-pa-md">
        <div class="row justify-center q-py-xl">
          <div class="col-12 col-md-8 text-center bg-grey-10 q-pa-lg rounded-borders">
            <q-icon name="security" size="64px" color="primary" class="q-mb-md" />
            <div class="text-h5 text-weight-bold q-mb-xs">Authorized Co-Browsing (User Chrome)</div>
            <div class="text-subtitle1 text-grey-4 q-mb-md">
              Initiate single-tab WebRTC co-browsing and remote browser control with explicit client consent.
            </div>
            <div class="q-gutter-x-sm">
              <q-btn
                color="primary"
                icon="security"
                label="Launch Co-Browsing Session"
                size="md"
                @click="openCoBrowse"
              />
            </div>
          </div>
        </div>
      </q-tab-panel>

      <!-- Background Windows Access Dummy Tab Panel -->
      <q-tab-panel name="winaccess" class="q-pa-md">
        <div class="row justify-center q-py-xl">
          <div class="col-12 col-md-8 text-center bg-grey-10 q-pa-lg rounded-borders">
            <q-icon name="desktop_windows" size="64px" color="amber-8" class="q-mb-md" />
            <div class="text-h5 text-weight-bold q-mb-xs">Background Windows Access</div>
            <div class="text-subtitle1 text-grey-4 q-mb-sm">
              Isolated background desktop & headless workspace management module.
            </div>
            <q-chip color="amber-9" text-color="white" icon="construction" label="Dummy Prototype — Feature Under Development" class="q-mb-md" />
            <div>
              <q-btn
                color="amber-8"
                text-color="black"
                icon="desktop_windows"
                label="Launch Background Workspace Access (Dummy)"
                size="md"
                @click="showDummyNotification"
              />
            </div>
          </div>
        </div>
      </q-tab-panel>
    </q-tab-panels>
  </div>
</template>

<script>
// composition imports
import { ref, computed, onMounted } from "vue";
import { useRoute } from "vue-router";
import { useQuasar, useMeta } from "quasar";
import {
  fetchAgentMeshCentralURLs,
  fetchAgentTerminalDefaults,
  runBackgroundWorkspace,
} from "@/api/agents";
import { fetchDashboardInfo } from "@/api/core";

// ui imports
import ProcessManager from "@/components/agents/remotebg/ProcessManager.vue";
import ServicesManager from "@/components/agents/remotebg/ServicesManager.vue";
import EventLogManager from "@/components/agents/remotebg/EventLogManager.vue";
import RegistryManager from "@/components/agents/remotebg/RegistryManager.vue";
import registryIcon from "../assets/windows-registry.png";
import TerminalManager from "@/components/agents/remotebg/TerminalManager.vue";
import CoBrowseModal from "@/components/modals/CoBrowseModal.vue";

export default {
  name: "RemoteBackground",
  components: {
    ServicesManager,
    EventLogManager,
    ProcessManager,
    RegistryManager,
    TerminalManager,
  },
  setup() {
    // setup quasar
    const $q = useQuasar();

    // vue router
    const { params } = useRoute();

    // meshcentral tabs
    const terminal = ref("");
    const file = ref("");
    const tab = ref("terminal");
    const terminalMode = ref("legacy");
    const terminalDefaults = ref(null);

    const agent_id = computed(() => params.agent_id);

    function openCoBrowse() {
      $q.dialog({
        component: CoBrowseModal,
        componentProps: {
          agentId: agent_id.value,
        },
      });
    }

    function showDummyNotification() {
      runBackgroundWorkspace(agent_id.value, "windows");
    }

    async function getMeshURLs() {
      const data = await fetchAgentMeshCentralURLs(params.agent_id);
      terminal.value = data.terminal;
      file.value = data.file;
      useMeta({
        title: `${data.hostname} - ${data.client} - ${data.site} | Remote Background`,
      });
    }

    async function getDashInfo() {
      const { dark_mode } = await fetchDashboardInfo();
      $q.dark.set(dark_mode);
      $q.loadingBar.setDefaults({ size: "0px" });
    }

    async function getTerminalDefaults() {
      try {
        const data = await fetchAgentTerminalDefaults(params.agent_id);
        terminalDefaults.value = data;

        // TODO remove this after a few releases as all agents should be updated by then
        const wantsNewTerminal = data?.terminal_mode === "new";
        const supportsNewTerminal = data?.supports_new_terminal === true;

        if (wantsNewTerminal && !supportsNewTerminal) {
          terminalMode.value = "legacy";

          $q.notify({
            type: "warning",
            message:
              "New terminal mode requires agent version 2.11.0 or higher. Reverting to legacy terminal mode. Please update the agent to use the new terminal.",
            timeout: 6000,
          });
          return;
        }

        terminalMode.value = wantsNewTerminal ? "new" : "legacy";
      } catch (e) {
        terminalMode.value = "legacy";

        $q.notify({
          type: "negative",
          message:
            e?.response?.data?.detail || "Failed to load terminal settings",
        });
      }
    }

    // vue lifecycle hooks
    onMounted(() => {
      getDashInfo();
      getMeshURLs();
      getTerminalDefaults();
    });

    return {
      // reactive data
      terminal,
      file,
      tab,
      agent_id,
      registryIcon,
      terminalMode,
      terminalDefaults,

      // methods
      openCoBrowse,
      showDummyNotification,
    };
  },
};
</script>
