<template>
  <q-bar class="bg-background q-pa-none relative-position">
    <div
      v-if="$q.platform.is.mac && !isFullscreen"
      class="mac-traffic-light-space"
    ></div>
    <img v-else src="icon.png" class="window-logo" alt="application logo" />
    <menu-button
      v-for="(root, index) of menudata"
      :key="index"
      :menudata="root"
      v-model:selected="subMenuOpenFlags[index]"
      :disable="menubarLocked"
      @mouseover="reassignSubMenuOpen(index)"
      @mouseleave="
        root.type === 'button' ? (subMenuOpenFlags[index] = false) : undefined
      "
    />
    <q-space />
    <div class="window-title" :class="{ 'text-warning': isSafeMode }">
      {{ titleText }}
    </div>
    <q-space />
    <title-bar-buttons />
  </q-bar>
</template>

<script setup lang="ts">
import { ref, computed, ComputedRef, watch } from "vue";
import { useStore } from "@/store";
import MenuButton from "@/components/MenuButton.vue";
import TitleBarButtons from "@/components/TitleBarButtons.vue";
import { useQuasar } from "quasar";
import { HotkeyAction, HotkeyReturnType } from "@/type/preload";
import { setHotkeyFunctions } from "@/store/setting";
import {
  generateAndConnectAndSaveAudioWithDialog,
  generateAndSaveAllAudioWithDialog,
  generateAndSaveOneAudioWithDialog,
  connectAndExportTextWithDialog,
} from "@/components/Dialog";
import { base64ImageToUri } from "@/helpers/imageHelper";

export type MenuItemBase<T extends string> = {
  type: T;
  label?: string;
};

export type MenuItemSeparator = MenuItemBase<"separator">;

export type MenuItemRoot = MenuItemBase<"root"> & {
  onClick: () => void;
  subMenu: MenuItemData[];
  icon?: string;
  disableWhenUiLocked: boolean;
};

export type MenuItemButton = MenuItemBase<"button"> & {
  onClick: () => void;
  icon?: string;
  disableWhenUiLocked: boolean;
};

export type MenuItemCheckbox = MenuItemBase<"checkbox"> & {
  checked: ComputedRef<boolean>;
  onClick: () => void;
  disableWhenUiLocked: boolean;
};

export type MenuItemData =
  | MenuItemSeparator
  | MenuItemRoot
  | MenuItemButton
  | MenuItemCheckbox;

export type MenuItemType = MenuItemData["type"];

const store = useStore();
const $q = useQuasar();
const currentVersion = ref("");
window.electron.getAppInfos().then((obj) => {
  currentVersion.value = obj.version;
});
const isSafeMode = computed(() => store.state.isSafeMode);
const uiLocked = computed(() => store.getters.UI_LOCKED);
const menubarLocked = computed(() => store.getters.MENUBAR_LOCKED);
const projectName = computed(() => store.getters.PROJECT_NAME);
const useGpu = computed(() => store.state.useGpu);
const isEdited = computed(() => store.getters.IS_EDITED);
const isFullscreen = computed(() => store.getters.IS_FULLSCREEN);
const engineIds = computed(() => store.state.engineIds);
const engineInfos = computed(() => store.state.engineInfos);
const engineManifests = computed(() => store.state.engineManifests);

const titleText = computed(
  () =>
    (isEdited.value ? "*" : "") +
    (projectName.value !== undefined ? projectName.value + " - " : "") +
    "VOICEVOX" +
    (currentVersion.value ? " - Ver. " + currentVersion.value + " - " : "") +
    (useGpu.value ? "GPU" : "CPU") +
    (isSafeMode.value ? " - ??????????????????" : "")
);

// FIXME: App.vue??????????????????
watch(titleText, (newTitle) => {
  window.document.title = newTitle;
});

const createNewProject = async () => {
  if (!uiLocked.value) {
    await store.dispatch("CREATE_NEW_PROJECT", {});
  }
};

const generateAndSaveAllAudio = async () => {
  if (!uiLocked.value) {
    await generateAndSaveAllAudioWithDialog({
      encoding: store.state.savingSetting.fileEncoding,
      quasarDialog: $q.dialog,
      dispatch: store.dispatch,
    });
  }
};

const generateAndConnectAndSaveAllAudio = async () => {
  if (!uiLocked.value) {
    await generateAndConnectAndSaveAudioWithDialog({
      quasarDialog: $q.dialog,
      dispatch: store.dispatch,
      encoding: store.state.savingSetting.fileEncoding,
    });
  }
};

const generateAndSaveOneAudio = async () => {
  if (uiLocked.value) return;

  const activeAudioKey = store.getters.ACTIVE_AUDIO_KEY;
  if (activeAudioKey == undefined) {
    $q.dialog({
      title: "?????????????????????????????????????????????",
      message: "????????????????????????????????????????????????????????????????????????",
      ok: {
        label: "?????????",
        flat: true,
        textColor: "secondary",
      },
    });
    return;
  }

  await generateAndSaveOneAudioWithDialog({
    audioKey: activeAudioKey,
    encoding: store.state.savingSetting.fileEncoding,
    quasarDialog: $q.dialog,
    dispatch: store.dispatch,
  });
};

const connectAndExportText = async () => {
  if (!uiLocked.value) {
    await connectAndExportTextWithDialog({
      quasarDialog: $q.dialog,
      dispatch: store.dispatch,
      encoding: store.state.savingSetting.fileEncoding,
    });
  }
};

const importTextFile = () => {
  if (!uiLocked.value) {
    store.dispatch("COMMAND_IMPORT_FROM_FILE", {});
  }
};

const saveProject = () => {
  if (!uiLocked.value) {
    store.dispatch("SAVE_PROJECT_FILE", { overwrite: true });
  }
};

const saveProjectAs = () => {
  if (!uiLocked.value) {
    store.dispatch("SAVE_PROJECT_FILE", {});
  }
};

const importProject = () => {
  if (!uiLocked.value) {
    store.dispatch("LOAD_PROJECT_FILE", {});
  }
};
const closeAllDialog = () => {
  store.dispatch("SET_DIALOG_OPEN", {
    isSettingDialogOpen: false,
  });
  store.dispatch("SET_DIALOG_OPEN", {
    isHelpDialogOpen: false,
  });
  store.dispatch("SET_DIALOG_OPEN", {
    isHotkeySettingDialogOpen: false,
  });
  store.dispatch("SET_DIALOG_OPEN", {
    isToolbarSettingDialogOpen: false,
  });
  store.dispatch("SET_DIALOG_OPEN", {
    isCharacterOrderDialogOpen: false,
  });
  store.dispatch("SET_DIALOG_OPEN", {
    isDefaultStyleSelectDialogOpen: false,
  });
};

const openHelpDialog = () => {
  store.dispatch("SET_DIALOG_OPEN", {
    isHelpDialogOpen: true,
  });
};

const menudata = ref<MenuItemData[]>([
  {
    type: "root",
    label: "????????????",
    onClick: () => {
      closeAllDialog();
    },
    disableWhenUiLocked: false,
    subMenu: [
      {
        type: "button",
        label: "??????????????????",
        onClick: () => {
          generateAndSaveAllAudio();
        },
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "????????????????????????",
        onClick: () => {
          generateAndSaveOneAudio();
        },
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "??????????????????????????????",
        onClick: () => {
          generateAndConnectAndSaveAllAudio();
        },
        disableWhenUiLocked: true,
      },
      { type: "separator" },
      {
        type: "button",
        label: "????????????????????????????????????",
        onClick: () => {
          connectAndExportText();
        },
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "????????????????????????",
        onClick: () => {
          importTextFile();
        },
        disableWhenUiLocked: true,
      },
      { type: "separator" },
      {
        type: "button",
        label: "????????????????????????",
        onClick: createNewProject,
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "????????????????????????????????????",
        onClick: () => {
          saveProject();
        },
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "?????????????????????????????????????????????",
        onClick: () => {
          saveProjectAs();
        },
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "??????????????????????????????",
        onClick: () => {
          importProject();
        },
        disableWhenUiLocked: true,
      },
    ],
  },
  {
    type: "root",
    label: "????????????",
    onClick: () => {
      closeAllDialog();
    },
    disableWhenUiLocked: false,
    subMenu: [],
  },
  {
    type: "root",
    label: "??????",
    onClick: () => {
      closeAllDialog();
    },
    disableWhenUiLocked: false,
    subMenu: [
      {
        type: "button",
        label: "??????????????????",
        onClick() {
          store.dispatch("SET_DIALOG_OPEN", {
            isHotkeySettingDialogOpen: true,
          });
        },
        disableWhenUiLocked: false,
      },
      {
        type: "button",
        label: "????????????????????????????????????",
        onClick() {
          store.dispatch("SET_DIALOG_OPEN", {
            isToolbarSettingDialogOpen: true,
          });
        },
        disableWhenUiLocked: false,
      },
      {
        type: "button",
        label: "???????????????????????????????????????",
        onClick() {
          store.dispatch("SET_DIALOG_OPEN", {
            isCharacterOrderDialogOpen: true,
          });
        },
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "???????????????????????????",
        onClick() {
          store.dispatch("SET_DIALOG_OPEN", {
            isDefaultStyleSelectDialogOpen: true,
          });
        },
        disableWhenUiLocked: true,
      },
      {
        type: "button",
        label: "?????????????????????????????????",
        onClick() {
          store.dispatch("SET_DIALOG_OPEN", {
            isDictionaryManageDialogOpen: true,
          });
        },
        disableWhenUiLocked: true,
      },
      { type: "separator" },
      {
        type: "button",
        label: "???????????????",
        onClick() {
          store.dispatch("SET_DIALOG_OPEN", {
            isSettingDialogOpen: true,
          });
        },
        disableWhenUiLocked: false,
      },
    ],
  },
  {
    type: "button",
    label: "?????????",
    onClick: () => {
      if (store.state.isHelpDialogOpen) closeAllDialog();
      else {
        closeAllDialog();
        openHelpDialog();
      }
    },
    disableWhenUiLocked: false,
  },
]);

if (store.state.isSafeMode) {
  (
    menudata.value.find((data) => data.label === "??????") as MenuItemRoot
  ).subMenu.push({
    type: "button",
    label: "???????????????????????????",
    onClick() {
      store.dispatch("RESTART_APP", {
        isSafeMode: false,
      });
    },
    disableWhenUiLocked: false,
  });
}

const subMenuOpenFlags = ref(
  [...Array(menudata.value.length)].map(() => false)
);

const reassignSubMenuOpen = (idx: number) => {
  if (subMenuOpenFlags.value[idx]) return;
  if (subMenuOpenFlags.value.find((x) => x)) {
    const arr = [...Array(menudata.value.length)].map(() => false);
    arr[idx] = true;
    subMenuOpenFlags.value = arr;
  }
};

const hotkeyMap = new Map<HotkeyAction, () => HotkeyReturnType>([
  ["????????????????????????", createNewProject],
  ["??????????????????", generateAndSaveAllAudio],
  ["????????????????????????", generateAndSaveOneAudio],
  ["??????????????????????????????", generateAndConnectAndSaveAllAudio],
  ["????????????????????????", importTextFile],
  ["????????????????????????????????????", saveProject],
  ["?????????????????????????????????????????????", saveProjectAs],
  ["??????????????????????????????", importProject],
]);

setHotkeyFunctions(hotkeyMap);

// ?????????????????????????????????
async function updateEngines() {
  const engineMenu = menudata.value.find(
    (x) => x.type === "root" && x.label === "????????????"
  ) as MenuItemRoot;
  if (Object.values(engineInfos.value).length === 1) {
    const engineInfo = Object.values(engineInfos.value)[0];
    engineMenu.subMenu = [
      {
        type: "button",
        label: "?????????",
        onClick: () => {
          store.dispatch("RESTART_ENGINES", {
            engineIds: [engineInfo.uuid],
          });
        },
        disableWhenUiLocked: false,
      },
    ].filter((x) => x) as MenuItemData[];
  } else {
    engineMenu.subMenu = [
      ...store.getters.GET_SORTED_ENGINE_INFOS.map(
        (engineInfo) =>
          ({
            type: "root",
            label: engineInfo.name,
            icon:
              engineManifests.value[engineInfo.uuid] &&
              base64ImageToUri(engineManifests.value[engineInfo.uuid].icon),
            subMenu: [
              engineInfo.path && {
                type: "button",
                label: "?????????????????????",
                onClick: () => {
                  store.dispatch("OPEN_ENGINE_DIRECTORY", {
                    engineId: engineInfo.uuid,
                  });
                },
                disableWhenUiLocked: false,
              },
              {
                type: "button",
                label: "?????????",
                onClick: () => {
                  store.dispatch("RESTART_ENGINES", {
                    engineIds: [engineInfo.uuid],
                  });
                },
                disableWhenUiLocked: false,
              },
            ].filter((x) => x),
          } as MenuItemRoot)
      ),
      {
        type: "separator",
      },
      {
        type: "button",
        label: "?????????????????????????????????",
        onClick: () => {
          store.dispatch("RESTART_ENGINES", { engineIds: engineIds.value });
        },
        disableWhenUiLocked: false,
      },
    ];
  }
  engineMenu.subMenu.push({
    type: "button",
    label: "?????????????????????",
    onClick: () => {
      store.dispatch("SET_DIALOG_OPEN", {
        isEngineManageDialogOpen: true,
      });
    },
    disableWhenUiLocked: false,
  });
}
watch([engineInfos, engineManifests], updateEngines, { immediate: true }); // engineInfos???engineManifests????????????????????????????????????????????????

watch(uiLocked, () => {
  // UI??????????????????????????????????????????????????????????????????????????????????????????
  if (uiLocked.value) {
    subMenuOpenFlags.value = [...Array(menudata.value.length)].map(() => false);
  }
});
</script>

<style lang="scss">
@use '@/styles/colors' as colors;

.active-menu {
  background-color: rgba(colors.$primary-rgb, 0.3) !important;
}
</style>

<style scoped lang="scss">
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

.q-bar {
  min-height: vars.$menubar-height;
  -webkit-app-region: drag;
  > .q-btn {
    margin-left: 0;
    -webkit-app-region: no-drag;
  }
}

.window-logo {
  height: vars.$menubar-height;
}

.window-title {
  height: vars.$menubar-height;
  margin-right: 10%;
  text-overflow: ellipsis;
  overflow: hidden;
}

.mac-traffic-light-space {
  margin-right: 70px;
}
</style>
