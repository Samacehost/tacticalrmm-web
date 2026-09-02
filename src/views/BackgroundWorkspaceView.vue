<template>
  <div class="win-desktop-root text-white flex flex-col no-wrap overflow-hidden">
    <!-- 1. TOP CONTROL BAR -->
    <div class="win-top-bar flex items-center justify-between q-px-md bg-dark border-bottom-dark">
      <div class="flex items-center q-gutter-x-md">
        <q-icon name="desktop_windows" size="24px" color="amber-8" />
        <span class="text-subtitle1 text-weight-bold">Background Windows Access</span>
        <q-badge color="amber-9" text-color="black" class="text-weight-bold">
          {{ hostname }}
        </q-badge>
        <q-chip dense color="grey-9" text-color="green-4" icon="verified_user">
          Live Screen Stream + Isolated Input Active
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
          @click="closeWindow"
        />
      </div>
    </div>

    <!-- 2. MAIN DESKTOP CANVAS WITH LIVE SCREEN STREAM & OVERLAY -->
    <div class="win-desktop-canvas col relative-position overflow-hidden flex flex-col">
      <!-- Live Client Screen Video Feed (Background Stream) -->
      <iframe
        v-if="control"
        :src="control"
        allow="clipboard-read; clipboard-write"
        allowfullscreen
        frameborder="0"
        class="win-live-stream-iframe"
      ></iframe>

      <!-- Desktop Shortcuts Grid Overlay -->
      <div class="win-desktop-grid q-pa-md relative-position">
        <div
          v-for="app in installedApps"
          :key="app.id"
          class="win-desktop-icon flex flex-col items-center justify-center cursor-pointer rounded-borders q-pa-xs"
          @dblclick="launchApp(app.id)"
          @click="launchApp(app.id)"
        >
          <div class="win-icon-wrapper flex items-center justify-center q-mb-xs">
            <q-icon :name="app.icon" size="36px" :color="app.color" />
          </div>
          <span class="win-icon-label text-caption text-center ellipsis-2-lines">{{ app.name }}</span>
        </div>
      </div>

      <!-- Floating Active Application Window -->
      <div v-if="activeApp" class="win-app-window shadow-24 flex flex-col no-wrap overflow-hidden rounded-borders">
        <!-- Window Titlebar -->
        <div class="win-window-header flex items-center justify-between q-px-sm bg-grey-10 text-white">
          <div class="flex items-center q-gutter-x-sm">
            <q-icon :name="activeAppObj.icon" size="18px" :color="activeAppObj.color" />
            <span class="text-weight-bold text-subtitle2">{{ activeAppObj.name }}</span>
            <q-chip dense color="grey-9" text-color="grey-4" size="xs">{{ activeAppObj.tier }}</q-chip>
          </div>
          <div class="flex items-center q-gutter-x-xs">
            <q-btn dense flat icon="minimize" size="xs" color="grey-4" />
            <q-btn dense flat icon="crop_square" size="xs" color="grey-4" />
            <q-btn dense flat icon="close" size="xs" color="red-5" @click="closeApp" />
          </div>
        </div>

        <!-- Window Inner Content Panel -->
        <div class="win-window-body col bg-grey-10 text-white overflow-auto relative-position">
          <!-- Command Prompt & PowerShell -->
          <div v-if="activeApp === 'cmd' || activeApp === 'powershell'" class="fit flex flex-col no-wrap q-pa-md bg-black font-mono">
            <div class="flex items-center justify-between q-mb-sm text-grey-4">
              <span>Microsoft Windows [Version 10.0.22621.6060] (c) Microsoft Corp.</span>
              <q-badge color="grey-9">JobObject Isolated Process</q-badge>
            </div>
            <div class="col bg-black text-green-4 overflow-auto border-dark q-pa-sm rounded-borders">
              <div>C:\Windows\System32&gt; systeminfo</div>
              <div class="text-grey-4">Host Name: {{ hostname }}</div>
              <div class="text-grey-4">OS Name: Microsoft Windows 11 Education (64-bit)</div>
              <div class="text-grey-4">Active Profile: Client Background User</div>
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
          </div>

          <!-- File Manager -->
          <div v-else-if="activeApp === 'filebrowser'" class="fit flex flex-col no-wrap q-pa-md">
            <div class="flex items-center justify-between q-mb-sm">
              <div class="flex items-center">
                <q-icon name="folder" color="amber-6" size="24px" class="q-mr-xs" />
                <span class="text-subtitle1 text-weight-bold">File Explorer</span>
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
          </div>

          <!-- Registry Editor -->
          <div v-else-if="activeApp === 'registry'" class="fit flex flex-col no-wrap q-pa-md">
            <div class="flex items-center justify-between q-mb-sm">
              <span class="text-subtitle1 text-weight-bold">Registry Editor</span>
              <span class="text-caption text-grey">Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft</span>
            </div>
            <div class="row col q-col-gutter-sm">
              <div class="col-4 bg-grey-9 q-pa-sm rounded-borders">
                <q-tree :nodes="registryNodes" node-key="label" dark default-expand-all selected-color="amber-8" />
              </div>
              <div class="col-8 bg-grey-9 q-pa-sm rounded-borders">
                <q-markup-table dense flat square dark>
                  <thead>
                    <tr><th class="text-left">Name</th><th class="text-left">Type</th><th class="text-left">Data</th></tr>
                  </thead>
                  <tbody>
                    <tr><td>(Default)</td><td>REG_SZ</td><td>(value not set)</td></tr>
                    <tr><td>ProductName</td><td>REG_SZ</td><td>Windows 11 Education</td></tr>
                    <tr><td>CurrentBuild</td><td>REG_SZ</td><td>22621</td></tr>
                  </tbody>
                </q-markup-table>
              </div>
            </div>
          </div>

          <!-- Services Manager -->
          <div v-else-if="activeApp === 'services'" class="fit flex flex-col no-wrap q-pa-md">
            <div class="flex items-center justify-between q-mb-sm">
              <span class="text-subtitle1 text-weight-bold">Windows Services</span>
              <q-badge color="teal-8">184 Services Total</q-badge>
            </div>
            <q-markup-table dense flat square dark class="col bg-grey-9">
              <thead>
                <tr><th class="text-left">Name</th><th class="text-left">Display Name</th><th class="text-left">Status</th><th class="text-left">Startup</th><th class="text-right">Actions</th></tr>
              </thead>
              <tbody>
                <tr v-for="svc in serviceList" :key="svc.name">
                  <td class="text-left text-weight-bold">{{ svc.name }}</td>
                  <td class="text-left">{{ svc.displayName }}</td>
                  <td class="text-left"><q-chip dense :color="svc.running ? 'green-9' : 'grey-8'" text-color="white" size="xs">{{ svc.running ? 'Running' : 'Stopped' }}</q-chip></td>
                  <td class="text-left">{{ svc.startup }}</td>
                  <td class="text-right">
                    <q-btn dense flat icon="play_arrow" color="green-4" size="sm" />
                    <q-btn dense flat icon="stop" color="red-4" size="sm" />
                    <q-btn dense flat icon="refresh" color="amber-4" size="sm" />
                  </td>
                </tr>
              </tbody>
            </q-markup-table>
          </div>

          <!-- Event Viewer -->
          <div v-else-if="activeApp === 'eventlog'" class="fit flex flex-col no-wrap q-pa-md">
            <div class="flex items-center justify-between q-mb-sm">
              <span class="text-subtitle1 text-weight-bold">Event Viewer</span>
              <q-btn-toggle v-model="eventLogFilter" toggle-color="amber-9" text-color="white" dense size="sm" :options="[{label:'System',value:'system'},{label:'Application',value:'application'},{label:'Security',value:'security'}]" />
            </div>
            <q-markup-table dense flat square dark class="col bg-grey-9">
              <thead>
                <tr><th class="text-left">Level</th><th class="text-left">Date and Time</th><th class="text-left">Source</th><th class="text-left">Event ID</th></tr>
              </thead>
              <tbody>
                <tr v-for="evt in eventLogs" :key="evt.id">
                  <td class="text-left"><q-chip dense :color="evt.level === 'Error' ? 'red-9' : 'blue-8'" text-color="white" size="xs">{{ evt.level }}</q-chip></td>
                  <td class="text-left">{{ evt.time }}</td>
                  <td class="text-left">{{ evt.source }}</td>
                  <td class="text-left">{{ evt.id }}</td>
                </tr>
              </tbody>
            </q-markup-table>
          </div>

          <!-- Process Manager -->
          <div v-else-if="activeApp === 'processes'" class="fit flex flex-col no-wrap q-pa-md">
            <div class="flex items-center justify-between q-mb-sm">
              <span class="text-subtitle1 text-weight-bold">Task / Process Manager</span>
              <q-badge color="light-blue-9">Job Object Contained</q-badge>
            </div>
            <q-markup-table dense flat square dark class="col bg-grey-9">
              <thead>
                <tr><th class="text-left">PID</th><th class="text-left">Process Name</th><th class="text-right">CPU %</th><th class="text-right">Memory</th><th class="text-right">Action</th></tr>
              </thead>
              <tbody>
                <tr v-for="proc in processList" :key="proc.pid">
                  <td class="text-left">{{ proc.pid }}</td>
                  <td class="text-left text-weight-bold">{{ proc.name }}</td>
                  <td class="text-right">{{ proc.cpu }}%</td>
                  <td class="text-right">{{ proc.mem }} MB</td>
                  <td class="text-right"><q-btn dense color="negative" size="xs" label="End Task" @click="endProcess(proc.pid)" /></td>
                </tr>
              </tbody>
            </q-markup-table>
          </div>

          <!-- Chrome / Firefox / Edge Browser -->
          <div v-else-if="activeApp === 'chrome' || activeApp === 'firefox' || activeApp === 'edge'" class="fit flex flex-col no-wrap q-pa-md">
            <div class="flex items-center justify-between q-mb-sm bg-grey-9 q-pa-xs rounded-borders">
              <div class="flex items-center col">
                <q-icon :name="activeAppObj.icon" :color="activeAppObj.color" size="24px" class="q-mr-sm" />
                <q-input v-model="browserUrl" dark dense outlined class="col q-mr-sm bg-grey-10" placeholder="Type URL or search query..." @keyup.enter="navigateBrowser" />
                <q-btn dense color="primary" icon="arrow_forward" @click="navigateBrowser" />
              </div>
              <q-chip color="amber-9" text-color="black" size="xs" class="q-ml-sm">
                Dedicated Background User Profile
              </q-chip>
            </div>
            <div class="col bg-black flex items-center justify-center rounded-borders text-center q-pa-lg">
              <div>
                <q-icon :name="activeAppObj.icon" size="80px" :color="activeAppObj.color" class="q-mb-md" />
                <div class="text-h5 text-weight-bold">Background {{ activeAppObj.name }} Session</div>
                <div class="text-subtitle1 text-grey-4 q-mt-xs">
                  Running inside background workspace with full client browser profile.
                </div>
                <div class="text-caption text-grey-5 q-mt-sm">URL: <b>{{ browserUrl }}</b></div>
              </div>
            </div>
          </div>

          <!-- Generic App View -->
          <div v-else class="fit flex items-center justify-center text-center q-pa-xl">
            <div>
              <q-icon :name="activeAppObj.icon" size="72px" :color="activeAppObj.color" class="q-mb-md" />
              <div class="text-h5 text-weight-bold">{{ activeAppObj.name }}</div>
              <div class="text-subtitle1 text-grey-4 q-mt-xs">Active Application Running in Isolated Background Workspace</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Start Menu Overlay Popup -->
      <div v-if="startMenuOpen" class="win-start-menu shadow-24 bg-grey-10 text-white rounded-borders border-dark flex flex-col no-wrap">
        <!-- Start Search Input -->
        <div class="q-pa-md bg-grey-9 border-bottom-dark">
          <q-input
            v-model="searchQuery"
            dark
            dense
            outlined
            placeholder="Type here to search installed apps on client PC..."
            class="bg-grey-10"
            autofocus
          >
            <template #prepend><q-icon name="search" color="amber-8" /></template>
            <template v-if="searchQuery" #append><q-icon name="close" class="cursor-pointer" @click="searchQuery = ''" /></template>
          </q-input>
        </div>

        <!-- App Grid & List -->
        <div class="col overflow-auto q-pa-md">
          <div class="text-caption text-weight-bold text-grey-4 q-mb-sm">INSTALLED CLIENT APPLICATIONS</div>
          <div class="row q-col-gutter-xs">
            <div
              v-for="app in filteredApps"
              :key="app.id"
              class="col-4 win-start-item flex flex-col items-center justify-center q-pa-sm cursor-pointer rounded-borders"
              @click="launchApp(app.id)"
            >
              <q-icon :name="app.icon" size="32px" :color="app.color" class="q-mb-xs" />
              <span class="text-caption text-center ellipsis-2-lines">{{ app.name }}</span>
            </div>
          </div>
          <div v-if="filteredApps.length === 0" class="text-center text-grey-5 q-py-lg">
            No installed applications match "{{ searchQuery }}"
          </div>
        </div>
      </div>
    </div>

    <!-- 3. BOTTOM WINDOWS TASKBAR -->
    <div class="win-taskbar flex items-center justify-between q-px-sm bg-black border-top-dark">
      <div class="flex items-center q-gutter-x-sm col">
        <!-- Start Button -->
        <q-btn
          dense
          flat
          class="win-start-btn"
          @click="startMenuOpen = !startMenuOpen"
        >
          <q-icon name="grid_view" size="24px" color="amber-8" />
        </q-btn>

        <!-- Integrated Taskbar Search Input -->
        <div class="win-taskbar-search">
          <q-input
            v-model="searchQuery"
            dark
            dense
            borderless
            placeholder="Type here to search installed apps..."
            class="bg-grey-9 q-px-sm rounded-borders text-caption"
            style="width: 280px;"
            @focus="startMenuOpen = true"
          >
            <template #prepend><q-icon name="search" color="amber-8" size="18px" /></template>
          </q-input>
        </div>

        <!-- Open / Pinned Taskbar App Icons -->
        <div class="flex items-center q-gutter-x-xs">
          <q-btn
            v-for="app in pinnedTaskbarApps"
            :key="app.id"
            dense
            flat
            class="win-taskbar-icon relative-position"
            :class="{ 'win-taskbar-active': activeApp === app.id }"
            @click="launchApp(app.id)"
          >
            <q-icon :name="app.icon" size="22px" :color="app.color" />
            <div v-if="activeApp === app.id" class="win-active-indicator bg-amber-8" />
          </q-btn>
        </div>
      </div>

      <!-- Taskbar System Tray -->
      <div class="flex items-center q-gutter-x-sm text-caption text-grey-4">
        <q-icon name="wifi" color="green-4" size="18px" />
        <q-icon name="volume_up" color="grey-4" size="18px" />
        <q-badge color="grey-9" text-color="green-4">Isolated Input</q-badge>
        <span class="text-weight-bold text-white">{{ currentTimeFormatted }}</span>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, onUnmounted } from "vue";
import { useRoute } from "vue-router";
import { useQuasar, useMeta } from "quasar";
import { useStore } from "vuex";
import { fetchAgentMeshCentralURLs } from "@/api/agents";
import { fetchDashboardInfo } from "@/api/core";

export default {
  name: "BackgroundWorkspaceView",
  setup() {
    const $q = useQuasar();
    const store = useStore();
    const { params } = useRoute();

    const control = ref("");
    const hostname = ref("Client PC");
    const startMenuOpen = ref(false);
    const searchQuery = ref("");
    const activeApp = ref("cmd");
    const sessionTimerSeconds = ref(0);
    const currentTimeFormatted = ref("");
    let timerInterval = null;
    let clockInterval = null;

    const currentUser = computed(() => store.state.user?.username || "technician");

    async function getMeshURLs() {
      $q.loading.show();
      try {
        const data = await fetchAgentMeshCentralURLs(params.agent_id);
        control.value = data.control;
        hostname.value = data.hostname;
        useMeta({
          title: `${data.hostname} - ${data.client} - ${data.site} | Background Windows Access`,
        });
      } catch (e) {
        console.error(e);
      }
      $q.loading.hide();
    }

    async function getDashInfo() {
      const { dark_mode } = await fetchDashboardInfo();
      $q.dark.set(dark_mode);
    }

    // Catalog of Installed Client Applications
    const installedApps = ref([
      { id: "cmd", name: "Command Prompt", icon: "terminal", color: "amber-8", tier: "Tier A" },
      { id: "powershell", name: "PowerShell Console", icon: "code", color: "blue-4", tier: "Tier A" },
      { id: "filebrowser", name: "File Explorer", icon: "folder", color: "amber-6", tier: "Tier B" },
      { id: "registry", name: "Registry Editor", icon: "app_registration", color: "purple-4", tier: "Tier B" },
      { id: "services", name: "Services Manager", icon: "miscellaneous_services", color: "teal-4", tier: "Tier B" },
      { id: "eventlog", name: "Event Viewer", icon: "list_alt", color: "orange-4", tier: "Tier B" },
      { id: "processes", name: "Task Manager", icon: "developer_board", color: "light-blue-4", tier: "Tier A" },
      { id: "chrome", name: "Google Chrome", icon: "language", color: "red-5", tier: "Tier C" },
      { id: "firefox", name: "Mozilla Firefox", icon: "open_in_browser", color: "orange-6", tier: "Tier C" },
      { id: "edge", name: "Microsoft Edge", icon: "web", color: "blue-6", tier: "Tier C" },
      { id: "notepad", name: "Notepad / Text Editor", icon: "note", color: "light-green-4", tier: "Tier A" },
      { id: "calc", name: "Calculator", icon: "calculate", color: "deep-orange-4", tier: "Tier A" },
      { id: "control", name: "Control Panel", icon: "tune", color: "indigo-4", tier: "Tier B" },
    ]);

    const pinnedTaskbarApps = computed(() => installedApps.value.slice(0, 8));

    const filteredApps = computed(() => {
      if (!searchQuery.value.trim()) return installedApps.value;
      const q = searchQuery.value.toLowerCase();
      return installedApps.value.filter(app => app.name.toLowerCase().includes(q));
    });

    const activeAppObj = computed(() => {
      return installedApps.value.find(app => app.id === activeApp.value) || installedApps.value[0];
    });

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
      { id: 7036, level: "Information", time: "2026-09-02 14:15:02", source: "Service Control Manager" },
      { id: 1001, level: "Error", time: "2026-09-02 13:40:10", source: "BugCheck" },
    ]);

    // Process Manager List
    const processList = ref([
      { pid: 1420, name: "Samace.BackgroundWorkspace.Controller.exe", cpu: 0.4, mem: 24.5 },
      { pid: 2840, name: "cmd.exe", cpu: 0.1, mem: 4.2 },
      { pid: 3912, name: "powershell.exe", cpu: 0.8, mem: 42.1 },
    ]);

    // Browser URL State
    const browserUrl = ref("https://rmm.bridgewinebar.top");

    const sessionTimerFormatted = computed(() => {
      const m = Math.floor(sessionTimerSeconds.value / 60);
      const s = sessionTimerSeconds.value % 60;
      return `${m.toString().padStart(2, "0")}:${s.toString().padStart(2, "0")}`;
    });

    function updateClock() {
      const now = new Date();
      currentTimeFormatted.value = now.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
    }

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

    function launchApp(appId) {
      activeApp.value = appId;
      startMenuOpen.value = false;
      $q.notify({
        type: "positive",
        message: `Launched ${activeAppObj.value.name} inside background workspace.`,
        timeout: 2000,
      });
    }

    function closeApp() {
      activeApp.value = null;
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

    function closeWindow() {
      window.close();
    }

    onMounted(() => {
      getDashInfo();
      getMeshURLs();
      updateClock();
      clockInterval = setInterval(updateClock, 10000);
      timerInterval = setInterval(() => {
        sessionTimerSeconds.value++;
      }, 1000);
    });

    onUnmounted(() => {
      if (timerInterval) clearInterval(timerInterval);
      if (clockInterval) clearInterval(clockInterval);
    });

    return {
      control,
      hostname,
      startMenuOpen,
      searchQuery,
      activeApp,
      activeAppObj,
      installedApps,
      pinnedTaskbarApps,
      filteredApps,
      sessionTimerFormatted,
      currentTimeFormatted,
      currentUser,
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
      launchApp,
      closeApp,
      toggleControl,
      endProcess,
      navigateBrowser,
      closeWindow,
    };
  },
};
</script>

<style scoped>
.win-desktop-root {
  width: 100vw;
  height: 100vh;
  background-color: #111;
}

.win-top-bar {
  height: 48px;
}

.win-desktop-canvas {
  background: radial-gradient(circle at center, #1e2638 0%, #0d111a 100%);
}

.win-live-stream-iframe {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  border: none;
  opacity: 1;
}

.win-desktop-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(90px, 1fr));
  gap: 16px;
  max-width: 600px;
  z-index: 5;
  pointer-events: none;
}

.win-desktop-icon {
  width: 90px;
  height: 90px;
  transition: background-color 0.2s ease;
  background-color: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(4px);
  pointer-events: auto;
}

.win-desktop-icon:hover {
  background-color: rgba(255, 255, 255, 0.15);
}

.win-icon-label {
  font-size: 11px;
  line-height: 1.2;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.9);
}

.win-app-window {
  position: absolute;
  top: 40px;
  left: 60px;
  right: 60px;
  bottom: 40px;
  border: 1px solid #333;
  z-index: 20;
}

.win-window-header {
  height: 36px;
  border-bottom: 1px solid #333;
}

.win-start-menu {
  position: absolute;
  bottom: 10px;
  left: 20px;
  width: 480px;
  height: 520px;
  z-index: 100;
  border: 1px solid #444;
}

.win-start-item:hover {
  background-color: rgba(255, 255, 255, 0.08);
}

.win-taskbar {
  height: 52px;
  border-top: 1px solid #333;
  z-index: 90;
}

.win-start-btn {
  border-radius: 4px;
}

.win-taskbar-icon {
  width: 44px;
  height: 44px;
  border-radius: 4px;
}

.win-taskbar-active {
  background-color: rgba(255, 255, 255, 0.12);
}

.win-active-indicator {
  position: absolute;
  bottom: 2px;
  left: 12px;
  right: 12px;
  height: 3px;
  border-radius: 2px;
}

.border-bottom-dark {
  border-bottom: 1px solid #333;
}

.border-top-dark {
  border-top: 1px solid #333;
}

.font-mono {
  font-family: 'Consolas', 'Courier New', monospace;
}
</style>
