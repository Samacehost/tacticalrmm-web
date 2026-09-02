<template>
  <q-dialog
    ref="dialogRef"
    maximized
    transition-show="slide-up"
    transition-hide="slide-down"
    @hide="onDialogHide"
  >
    <q-card class="bg-grey-10 text-white flex flex-col no-wrap style-dialog" style="height: 100vh; overflow: hidden;">
      <!-- Session Top Bar -->
      <q-bar class="bg-dark text-white q-pa-sm flex items-center justify-between" style="min-height: 50px;">
        <div class="flex items-center q-gutter-x-md">
          <q-icon name="desktop_windows" size="24px" color="amber-8" />
          <span class="text-subtitle1 text-weight-bold">Background Windows Access</span>
          <q-badge color="amber-9" text-color="black" class="text-weight-bold">
            {{ agentHostname }}
          </q-badge>
          <q-chip dense color="grey-9" text-color="green-4" icon="verified_user">
            Connected (Isolated Session)
          </q-chip>
        </div>

        <div class="flex items-center q-gutter-x-sm">
          <span class="text-caption text-grey-4 q-mr-sm">Technician: <b>{{ currentUser }}</b></span>
          <q-chip dense color="blue-10" text-color="white" icon="timer">
            {{ sessionTimerFormatted }}
          </q-chip>
          <q-btn
            dense
            color="amber-9"
            text-color="black"
            icon="block"
            label="Revoke Control"
            no-caps
            class="q-px-sm"
            @click="toggleControl"
          />
          <q-btn
            dense
            color="negative"
            icon="power_settings_new"
            label="End Background Session"
            no-caps
            class="q-px-sm"
            @click="closeModal"
          />
        </div>
      </q-bar>

      <!-- Main Desktop Area -->
      <div class="col flex flex-col no-wrap relative-position bg-grey-9 overflow-hidden">
        <!-- Start Menu Dropdown Overlay -->
        <q-menu
          v-model="startMenuOpen"
          anchor="bottom start"
          self="top start"
          class="bg-grey-10 text-white shadow-10 rounded-borders"
          style="min-width: 320px; border: 1px solid #444;"
        >
          <div class="q-pa-md">
            <div class="text-weight-bold text-subtitle1 q-mb-sm flex items-center">
              <q-icon name="apps" color="amber-8" class="q-mr-xs" /> Application Launcher
            </div>
            <q-separator dark class="q-mb-md" />
            <q-list dense>
              <q-item clickable v-ripple @click="launchTool('cmd')">
                <q-item-section avatar><q-icon name="terminal" color="amber-8" /></q-item-section>
                <q-item-section>Command Prompt (CMD)</q-item-section>
                <q-item-section side><q-chip dense color="green-9" text-color="white" size="xs">Tier A</q-chip></q-item-section>
              </q-item>
              <q-item clickable v-ripple @click="launchTool('powershell')">
                <q-item-section avatar><q-icon name="code" color="blue-4" /></q-item-section>
                <q-item-section>PowerShell</q-item-section>
                <q-item-section side><q-chip dense color="green-9" text-color="white" size="xs">Tier A</q-chip></q-item-section>
              </q-item>
              <q-item clickable v-ripple @click="launchTool('filebrowser')">
                <q-item-section avatar><q-icon name="folder" color="amber-6" /></q-item-section>
                <q-item-section>File Manager</q-item-section>
                <q-item-section side><q-chip dense color="green-9" text-color="white" size="xs">Tier B</q-chip></q-item-section>
              </q-item>
              <q-item clickable v-ripple @click="launchTool('registry')">
                <q-item-section avatar><q-icon name="app_registration" color="purple-4" /></q-item-section>
                <q-item-section>Registry Editor</q-item-section>
                <q-item-section side><q-chip dense color="green-9" text-color="white" size="xs">Tier B</q-chip></q-item-section>
              </q-item>
              <q-item clickable v-ripple @click="launchTool('services')">
                <q-item-section avatar><q-icon name="miscellaneous_services" color="teal-4" /></q-item-section>
                <q-item-section>Services Manager</q-item-section>
                <q-item-section side><q-chip dense color="green-9" text-color="white" size="xs">Tier B</q-chip></q-item-section>
              </q-item>
              <q-item clickable v-ripple @click="launchTool('eventlog')">
                <q-item-section avatar><q-icon name="list_alt" color="orange-4" /></q-item-section>
                <q-item-section>Event Viewer</q-item-section>
                <q-item-section side><q-chip dense color="green-9" text-color="white" size="xs">Tier B</q-chip></q-item-section>
              </q-item>
              <q-item clickable v-ripple @click="launchTool('processes')">
                <q-item-section avatar><q-icon name="developer_board" color="light-blue-4" /></q-item-section>
                <q-item-section>Process Manager</q-item-section>
                <q-item-section side><q-chip dense color="green-9" text-color="white" size="xs">Tier A</q-chip></q-item-section>
              </q-item>
              <q-separator dark class="q-my-sm" />
              <q-item clickable v-ripple @click="launchTool('chrome')">
                <q-item-section avatar><q-icon name="language" color="red-5" /></q-item-section>
                <q-item-section>Google Chrome (Background Profile)</q-item-section>
                <q-item-section side><q-chip dense color="amber-9" text-color="black" size="xs">Tier C</q-chip></q-item-section>
              </q-item>
              <q-item clickable v-ripple @click="launchTool('firefox')">
                <q-item-section avatar><q-icon name="open_in_browser" color="orange-6" /></q-item-section>
                <q-item-section>Mozilla Firefox (Background Profile)</q-item-section>
                <q-item-section side><q-chip dense color="amber-9" text-color="black" size="xs">Tier C</q-chip></q-item-section>
              </q-item>
            </q-list>
          </div>
        </q-menu>

        <!-- Active Administrative Workspaces / Tab Panels -->
        <q-tab-panels v-model="activeTab" animated class="bg-grey-10 text-white fit">
          <!-- Terminal / CMD / PowerShell Panel -->
          <q-tab-panel name="cmd" class="q-pa-md flex flex-col no-wrap fit">
            <div class="flex items-center justify-between q-mb-sm">
              <div class="text-subtitle1 text-weight-bold flex items-center">
                <q-icon name="terminal" color="amber-8" class="q-mr-xs" /> Command Prompt & PowerShell Console
              </div>
              <q-badge color="grey-8">Isolated Process Environment</q-badge>
            </div>
            <div class="col bg-black q-pa-sm text-green-4 font-mono rounded-borders overflow-auto border-grey">
              <div>Microsoft Windows [Version 10.0.22621.6060]</div>
              <div>(c) Microsoft Corporation. All rights reserved.</div>
              <br />
              <div>C:\Windows\System32> systeminfo</div>
              <div class="text-grey-4">Host Name: {{ agentHostname }}</div>
              <div class="text-grey-4">OS Name: Microsoft Windows 11 Education</div>
              <div class="text-grey-4">OS Version: 10.0.22621 N/A Build 22621</div>
              <div class="text-grey-4">System Type: x64-based PC</div>
              <br />
              <div v-for="(line, idx) in cmdOutput" :key="idx">{{ line }}</div>
              <div class="flex items-center q-mt-xs">
                <span class="text-green-4">C:\Windows\System32&gt;&nbsp;</span>
                <input
                  v-model="cmdInput"
                  class="bg-transparent text-white border-none col outline-none font-mono"
                  autofocus
                  @keyup.enter="execCmd"
                />
              </div>
            </div>
          </q-tab-panel>

          <!-- File Browser Panel -->
          <q-tab-panel name="filebrowser" class="q-pa-md fit flex flex-col no-wrap">
            <div class="flex items-center justify-between q-mb-sm">
              <div class="text-subtitle1 text-weight-bold flex items-center">
                <q-icon name="folder" color="amber-6" class="q-mr-xs" /> Background File Manager
              </div>
              <q-breadcrumbs active-color="amber-8" text-color="white" separator="/">
                <q-breadcrumbs-el label="C:" icon="hard_drive" @click="currentPath = 'C:\\'" />
                <q-breadcrumbs-el label="Windows" @click="currentPath = 'C:\\Windows'" />
                <q-breadcrumbs-el label="System32" @click="currentPath = 'C:\\Windows\\System32'" />
              </q-breadcrumbs>
            </div>
            <q-markup-table dense flat square dark class="col bg-grey-9">
              <thead>
                <tr>
                  <th class="text-left">Name</th>
                  <th class="text-left">Type</th>
                  <th class="text-right">Size</th>
                  <th class="text-right">Date Modified</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in fileList" :key="item.name" class="cursor-pointer hover-bg-grey">
                  <td class="text-left">
                    <q-icon :name="item.isDir ? 'folder' : 'insert_drive_file'" :color="item.isDir ? 'amber-6' : 'blue-4'" class="q-mr-xs" />
                    {{ item.name }}
                  </td>
                  <td class="text-left">{{ item.isDir ? 'File Folder' : 'System File' }}</td>
                  <td class="text-right">{{ item.size }}</td>
                  <td class="text-right">{{ item.mtime }}</td>
                </tr>
              </tbody>
            </q-markup-table>
          </q-tab-panel>

          <!-- Registry Editor Panel -->
          <q-tab-panel name="registry" class="q-pa-md fit flex flex-col no-wrap">
            <div class="flex items-center justify-between q-mb-sm">
              <div class="text-subtitle1 text-weight-bold flex items-center">
                <q-icon name="app_registration" color="purple-4" class="q-mr-xs" /> Registry Editor
              </div>
              <span class="text-caption text-grey">HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows</span>
            </div>
            <div class="row col q-col-gutter-sm">
              <div class="col-4 bg-grey-9 q-pa-sm rounded-borders">
                <q-tree
                  :nodes="registryNodes"
                  node-key="label"
                  dark
                  default-expand-all
                  selected-color="amber-8"
                />
              </div>
              <div class="col-8 bg-grey-9 q-pa-sm rounded-borders">
                <q-markup-table dense flat square dark>
                  <thead>
                    <tr>
                      <th class="text-left">Name</th>
                      <th class="text-left">Type</th>
                      <th class="text-left">Data</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr><td>(Default)</td><td>REG_SZ</td><td>(value not set)</td></tr>
                    <tr><td>ProductName</td><td>REG_SZ</td><td>Windows 11 Education</td></tr>
                    <tr><td>CurrentBuild</td><td>REG_SZ</td><td>22621</td></tr>
                    <tr><td>RegisteredOwner</td><td>REG_SZ</td><td>Samace Tech Lab</td></tr>
                  </tbody>
                </q-markup-table>
              </div>
            </div>
          </q-tab-panel>

          <!-- Services Manager Panel -->
          <q-tab-panel name="services" class="q-pa-md fit flex flex-col no-wrap">
            <div class="flex items-center justify-between q-mb-sm">
              <div class="text-subtitle1 text-weight-bold flex items-center">
                <q-icon name="miscellaneous_services" color="teal-4" class="q-mr-xs" /> Windows Services Manager
              </div>
              <q-badge color="teal-8">Total Services: 184</q-badge>
            </div>
            <q-markup-table dense flat square dark class="col bg-grey-9">
              <thead>
                <tr>
                  <th class="text-left">Name</th>
                  <th class="text-left">Display Name</th>
                  <th class="text-left">Status</th>
                  <th class="text-left">Startup Type</th>
                  <th class="text-right">Actions</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="svc in serviceList" :key="svc.name">
                  <td class="text-left text-weight-bold">{{ svc.name }}</td>
                  <td class="text-left">{{ svc.displayName }}</td>
                  <td class="text-left">
                    <q-chip dense :color="svc.running ? 'green-9' : 'grey-8'" text-color="white" size="xs">
                      {{ svc.running ? 'Running' : 'Stopped' }}
                    </q-chip>
                  </td>
                  <td class="text-left">{{ svc.startup }}</td>
                  <td class="text-right">
                    <q-btn dense flat icon="play_arrow" color="green-4" size="sm" />
                    <q-btn dense flat icon="stop" color="red-4" size="sm" />
                    <q-btn dense flat icon="refresh" color="amber-4" size="sm" />
                  </td>
                </tr>
              </tbody>
            </q-markup-table>
          </q-tab-panel>

          <!-- Event Viewer Panel -->
          <q-tab-panel name="eventlog" class="q-pa-md fit flex flex-col no-wrap">
            <div class="flex items-center justify-between q-mb-sm">
              <div class="text-subtitle1 text-weight-bold flex items-center">
                <q-icon name="list_alt" color="orange-4" class="q-mr-xs" /> Event Viewer (System & Application Logs)
              </div>
              <q-btn-toggle
                v-model="eventLogFilter"
                toggle-color="amber-9"
                text-color="white"
                dense
                size="sm"
                :options="[
                  {label: 'System', value: 'system'},
                  {label: 'Application', value: 'application'},
                  {label: 'Security', value: 'security'}
                ]"
              />
            </div>
            <q-markup-table dense flat square dark class="col bg-grey-9">
              <thead>
                <tr>
                  <th class="text-left">Level</th>
                  <th class="text-left">Date and Time</th>
                  <th class="text-left">Source</th>
                  <th class="text-left">Event ID</th>
                  <th class="text-left">Task Category</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="evt in eventLogs" :key="evt.id">
                  <td class="text-left">
                    <q-chip dense :color="evt.level === 'Error' ? 'red-9' : 'blue-8'" text-color="white" size="xs">
                      {{ evt.level }}
                    </q-chip>
                  </td>
                  <td class="text-left">{{ evt.time }}</td>
                  <td class="text-left">{{ evt.source }}</td>
                  <td class="text-left">{{ evt.id }}</td>
                  <td class="text-left">{{ evt.category }}</td>
                </tr>
              </tbody>
            </q-markup-table>
          </q-tab-panel>

          <!-- Process Manager Panel -->
          <q-tab-panel name="processes" class="q-pa-md fit flex flex-col no-wrap">
            <div class="flex items-center justify-between q-mb-sm">
              <div class="text-subtitle1 text-weight-bold flex items-center">
                <q-icon name="developer_board" color="light-blue-4" class="q-mr-xs" /> Background Workspace Process Manager
              </div>
              <q-badge color="light-blue-9">Job Object Container Contained</q-badge>
            </div>
            <q-markup-table dense flat square dark class="col bg-grey-9">
              <thead>
                <tr>
                  <th class="text-left">PID</th>
                  <th class="text-left">Process Name</th>
                  <th class="text-right">CPU %</th>
                  <th class="text-right">Memory (MB)</th>
                  <th class="text-right">Action</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="proc in processList" :key="proc.pid">
                  <td class="text-left">{{ proc.pid }}</td>
                  <td class="text-left font-weight-bold">{{ proc.name }}</td>
                  <td class="text-right">{{ proc.cpu }}%</td>
                  <td class="text-right">{{ proc.mem }} MB</td>
                  <td class="text-right">
                    <q-btn dense color="negative" size="xs" label="End Task" @click="endProcess(proc.pid)" />
                  </td>
                </tr>
              </tbody>
            </q-markup-table>
          </q-tab-panel>

          <!-- Chrome & Firefox Browser Workspace Panel -->
          <q-tab-panel name="chrome" class="q-pa-md fit flex flex-col no-wrap">
            <div class="flex items-center justify-between q-mb-sm bg-grey-9 q-pa-xs rounded-borders">
              <div class="flex items-center col">
                <q-icon name="language" color="red-5" size="24px" class="q-mr-sm" />
                <q-input
                  v-model="browserUrl"
                  dark
                  dense
                  outlined
                  class="col q-mr-sm bg-grey-10"
                  placeholder="Enter URL to navigate in background browser..."
                  @keyup.enter="navigateBrowser"
                />
                <q-btn dense color="primary" icon="arrow_forward" @click="navigateBrowser" />
              </div>
              <q-chip color="amber-9" text-color="black" size="xs" class="q-ml-sm">
                Dedicated User Data Profile (Isolated Process)
              </q-chip>
            </div>
            <div class="col bg-black flex items-center justify-center rounded-borders text-center q-pa-lg">
              <div>
                <q-icon name="language" size="80px" color="red-5" class="q-mb-md" />
                <div class="text-h5 text-weight-bold">Background Chromium Browser Workspace</div>
                <div class="text-subtitle1 text-grey-4 q-mt-xs">
                  Running isolated Chromium process with client profile context.
                </div>
                <div class="text-caption text-grey-5 q-mt-sm">
                  Active URL: <b>{{ browserUrl }}</b>
                </div>
              </div>
            </div>
          </q-tab-panel>
        </q-tab-panels>
      </div>

      <!-- Taskbar Bar -->
      <q-bar class="bg-black text-white q-pa-xs flex items-center justify-between border-top-dark" style="min-height: 44px;">
        <div class="flex items-center q-gutter-x-xs">
          <!-- Start Button -->
          <q-btn
            dense
            color="amber-9"
            text-color="black"
            icon="apps"
            label="Start"
            no-caps
            class="q-px-sm text-weight-bold"
            @click="startMenuOpen = !startMenuOpen"
          />
          <q-separator dark vertical class="q-mx-xs" />

          <!-- Open Tool Taskbar Items -->
          <q-btn
            v-for="t in taskbarTools"
            :key="t.name"
            dense
            :color="activeTab === t.name ? 'amber-9' : 'grey-8'"
            :text-color="activeTab === t.name ? 'black' : 'white'"
            :icon="t.icon"
            :label="t.label"
            no-caps
            class="q-px-sm"
            @click="activeTab = t.name"
          />
        </div>

        <div class="flex items-center text-caption text-grey-4 q-gutter-x-sm">
          <span><q-icon name="shield" color="green-4" /> Isolation Active</span>
          <span>1080p WebRTC</span>
        </div>
      </q-bar>
    </q-card>
  </q-dialog>
</template>

<script>
import { ref, computed, onMounted, onUnmounted } from "vue";
import { useDialogPluginComponent, useQuasar } from "quasar";
import { useStore } from "vuex";

export default {
  name: "BackgroundWorkspaceModal",
  props: {
    agentId: {
      type: String,
      required: true,
    },
    hostname: {
      type: String,
      default: "Client PC",
    },
  },
  emits: [...useDialogPluginComponent.emits],
  setup(props) {
    const $q = useQuasar();
    const store = useStore();
    const { dialogRef, onDialogHide, onDialogOK } = useDialogPluginComponent();

    const startMenuOpen = ref(false);
    const activeTab = ref("cmd");
    const sessionTimerSeconds = ref(0);
    let timerInterval = null;

    const currentUser = computed(() => store.state.user?.username || "technician");
    const agentHostname = computed(() => props.hostname || "DESKTOP-I2SCRCQ");

    // Command Prompt State
    const cmdInput = ref("");
    const cmdOutput = ref([
      "Service container initialized cleanly.",
      "JobObject containment policy enforced.",
    ]);

    // File Browser State
    const currentPath = ref("C:\\Windows\\System32");
    const fileList = ref([
      { name: "cmd.exe", isDir: false, size: "284 KB", mtime: "2026-08-15 14:20" },
      { name: "powershell.exe", isDir: false, size: "448 KB", mtime: "2026-08-15 14:20" },
      { name: "regedit.exe", isDir: false, size: "360 KB", mtime: "2026-08-15 14:20" },
      { name: "drivers", isDir: true, size: "--", mtime: "2026-08-20 10:12" },
      { name: "GroupPolicy", isDir: true, size: "--", mtime: "2026-08-20 10:12" },
    ]);

    // Registry Tree State
    const registryNodes = ref([
      { label: "HKEY_LOCAL_MACHINE", children: [{ label: "HARDWARE" }, { label: "SOFTWARE" }, { label: "SYSTEM" }] },
      { label: "HKEY_CURRENT_USER", children: [{ label: "Software" }, { label: "Control Panel" }] },
    ]);

    // Services Manager State
    const serviceList = ref([
      { name: "tacticalrmm", displayName: "Tactical RMM Agent", running: true, startup: "Automatic" },
      { name: "wuauserv", displayName: "Windows Update", running: true, startup: "Manual" },
      { name: "Spooler", displayName: "Print Spooler", running: false, startup: "Disabled" },
      { name: "WinDefend", displayName: "Microsoft Defender Antivirus", running: true, startup: "Automatic" },
    ]);

    // Event Viewer Logs
    const eventLogFilter = ref("system");
    const eventLogs = ref([
      { id: 7036, level: "Information", time: "2026-09-02 14:15:02", source: "Service Control Manager", category: "None" },
      { id: 1001, level: "Error", time: "2026-09-02 13:40:10", source: "BugCheck", category: "System" },
      { id: 4624, level: "Information", time: "2026-09-02 12:00:00", source: "Microsoft-Windows-Security-Auditing", category: "Logon" },
    ]);

    // Process Manager List
    const processList = ref([
      { pid: 1420, name: "Samace.BackgroundWorkspace.Controller.exe", cpu: 0.4, mem: 24.5 },
      { pid: 2840, name: "cmd.exe", cpu: 0.1, mem: 4.2 },
      { pid: 3912, name: "powershell.exe", cpu: 0.8, mem: 42.1 },
      { pid: 5120, name: "chrome.exe (Background Profile)", cpu: 1.2, mem: 128.4 },
    ]);

    // Browser URL State
    const browserUrl = ref("https://rmm.bridgewinebar.top");

    const taskbarTools = ref([
      { name: "cmd", label: "CMD / PowerShell", icon: "terminal" },
      { name: "filebrowser", label: "Files", icon: "folder" },
      { name: "registry", label: "Registry", icon: "app_registration" },
      { name: "services", label: "Services", icon: "miscellaneous_services" },
      { name: "eventlog", label: "Event Viewer", icon: "list_alt" },
      { name: "processes", label: "Processes", icon: "developer_board" },
    ]);

    const sessionTimerFormatted = computed(() => {
      const m = Math.floor(sessionTimerSeconds.value / 60);
      const s = sessionTimerSeconds.value % 60;
      return `${m.toString().padStart(2, "0")}:${s.toString().padStart(2, "0")}`;
    });

    function execCmd() {
      if (!cmdInput.value.trim()) return;
      cmdOutput.value.push(`C:\\Windows\\System32> ${cmdInput.value}`);
      if (cmdInput.value.trim() === "cls") {
        cmdOutput.value = [];
      } else {
        cmdOutput.value.push(`[Executed in Background Workspace]: ${cmdInput.value}`);
      }
      cmdInput.value = "";
    }

    function launchTool(toolName) {
      activeTab.value = toolName;
      startMenuOpen.value = false;
      $q.notify({
        type: "positive",
        message: `Launched ${toolName.toUpperCase()} inside background workspace.`,
        timeout: 2000,
      });
    }

    function toggleControl() {
      $q.notify({
        type: "warning",
        color: "amber-9",
        textColor: "black",
        message: "Control input revoked. Session converted to View-Only mode.",
        timeout: 3000,
      });
    }

    function endProcess(pid) {
      processList.value = processList.value.filter(p => p.pid !== pid);
      $q.notify({
        type: "negative",
        message: `Terminated process PID ${pid} inside background workspace.`,
        timeout: 2000,
      });
    }

    function navigateBrowser() {
      if (!browserUrl.value.startsWith("http")) {
        browserUrl.value = "https://" + browserUrl.value;
      }
      $q.notify({
        type: "info",
        message: `Navigating background browser to ${browserUrl.value}`,
        timeout: 2000,
      });
    }

    function closeModal() {
      if (timerInterval) clearInterval(timerInterval);
      onDialogOK();
    }

    onMounted(() => {
      timerInterval = setInterval(() => {
        sessionTimerSeconds.value++;
      }, 1000);
    });

    onUnmounted(() => {
      if (timerInterval) clearInterval(timerInterval);
    });

    return {
      dialogRef,
      onDialogHide,
      closeModal,
      startMenuOpen,
      activeTab,
      sessionTimerSeconds,
      sessionTimerFormatted,
      currentUser,
      agentHostname,
      cmdInput,
      cmdOutput,
      execCmd,
      currentPath,
      fileList,
      registryNodes,
      serviceList,
      eventLogFilter,
      eventLogs,
      processList,
      browserUrl,
      taskbarTools,
      launchTool,
      toggleControl,
      endProcess,
      navigateBrowser,
    };
  },
};
</script>

<style scoped>
.font-mono {
  font-family: 'Consolas', 'Courier New', monospace;
}
.border-grey {
  border: 1px solid #333;
}
.border-top-dark {
  border-top: 1px solid #333;
}
.hover-bg-grey:hover {
  background-color: rgba(255, 255, 255, 0.05);
}
</style>
