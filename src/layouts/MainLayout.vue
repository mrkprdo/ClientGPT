<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated bordered style="background-color: #1d1d1d">
      <q-toolbar>
        <q-btn
          flat
          dense
          round
          icon="menu"
          aria-label="Menu"
          @click="toggleLeftDrawer"
          title="Ctrl + B"
        />
        <q-toolbar-title class="text-weight-bolder">
          <a href="/" style="text-decoration: none">
            <span class="text-warning">Client</span
            ><span class="text-secondary">GPT</span>
          </a>
        </q-toolbar-title>
        <span class="text-warning">{{ version }}</span>
      </q-toolbar>
    </q-header>
    <q-drawer
      v-model="leftDrawerOpen"
      elevated
      style="background-color: #3b3b3b; padding-left: 15px; padding-right: 15px"
    >
      <q-tabs
        v-model="tab"
        dense
        class="text-grey"
        active-color="secondary"
        indicator-color="warning"
        align="center"
        narrow-indicator
      >
        <q-tab name="chats" label="Chats" />
        <q-tab name="config" label="Config" />
      </q-tabs>
      <q-tab-panels v-model="tab" animated style="height: 90%" dark>
        <q-tab-panel name="chats">
          <q-btn
            color="grey-9"
            size="md"
            style="width: 100%; margin-bottom: 2vh"
            icon="edit_square"
            label="Create New Chat"
            @click="handleCreate()"
          />
          <div
            v-for="(data, index) in localData.data"
            :key="data.timestamp"
            style="
              display: flex;
              justify-content: space-evenly;
              align-items: center;
              gap: 1vw;
            "
          >
            <q-btn
              v-if="data.history.length > 1"
              size="sm"
              style="
                width: 100%;
                margin-bottom: 0.5vh;
                white-space: nowrap;
                overflow: hidden;
                text-overflow: ellipsis;
              "
              icon="article"
              align="left"
              class="text-primary"
              :label="truncate(data.title)"
              :color="index === localData.selectedIndex ? 'grey-10' : 'grey-8'"
              @click="
                () => {
                  localData.selectedIndex = index;
                  leftDrawerOpen = false;
                }
              "
            />
            <q-icon
              v-if="data.history.length > 1"
              size="xs"
              color="info"
              name="delete"
              @click="handleDelete(index)"
              style="cursor: pointer"
            ></q-icon>
          </div>
        </q-tab-panel>

        <q-tab-panel name="config">
          <q-item-label class="text-white" header>
            Server Configuration
          </q-item-label>
          <div
            v-for="item in serverConfig"
            :key="item.field"
            class="config-form"
          >
            <q-input
              v-if="item.type != 'select'"
              v-model="item.model"
              dark
              dense
              standout
              :title="item?.title"
              :v-model="item.value"
              :label="item.field"
              :placeholder="item.placeholder"
              :type="item.type"
            />
            <q-select
              v-else
              v-model="item.model"
              dark
              dense
              standout
              :title="item.title"
              :v-model="item.value"
              :label="item.field"
              :options="['axios', 'openai']"
            ></q-select>
          </div>
          <q-btn
            round
            size="sm"
            icon="save"
            color="grey-8"
            class="float-right bg-white"
            style="margin-top: 10px"
            @click="handleSave"
          />
        </q-tab-panel>
      </q-tab-panels>
      <q-item-label align="center" header>
        <a
          class="text-warning"
          href="https://twitter.com/mrkprdo"
          style="text-decoration: none"
          >@mrkprdo</a
        >
      </q-item-label>
    </q-drawer>

    <q-page-container style="background-color: #3b3b3b">
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script>
import { useQuasar } from "quasar";
import { defineComponent, onMounted, onBeforeUnmount, ref } from "vue";
import { version } from "../../package.json";
import { useConfigStore, useLocalDataStore } from "../stores/config";

export default defineComponent({
  name: "MainLayout",

  setup() {
    const $q = useQuasar();
    const config = useConfigStore();
    const localData = useLocalDataStore();
    const leftDrawerOpen = ref(false);
    const tab = ref("chats");
    const host = ref(config.host);
    const port = ref(Number(config.port));
    const role = ref(config.role);
    const apiCall = ref(config.apiCall);
    const serverConfig = ref([
      {
        field: "Host",
        value: "",
        placeholder: "192.168.x.x",
        type: "text",
        model: host,
      },
      {
        field: "Port",
        value: 0,
        placeholder: 1234,
        type: "number",
        model: port,
      },
      {
        field: "Role",
        value: "",
        placeholder:
          "You are an AI assistant that responds to what the user queries.",
        type: "text",
        model: role,
        title: "Role will be added as pre-prompt.",
      },
      {
        field: "Api Call",
        value: "",
        type: "select",
        model: apiCall,
        title:
          "Selecting 'axios' will wait for response before displaying. Selecting 'openai' will stream the response.",
      },
    ]);

    const handleSave = () => {
      config.host = host.value;
      config.port = Number(port.value);
      config.role = role.value;
      config.apiCall = apiCall.value;

      $q.notify({
        message: "Server configuration saved!",
        position: "top",
        icon: "save",
      });
    };

    const truncate = (str) => {
      if (str.length <= 25) {
        return str;
      } else {
        return str.slice(0, 22) + "...";
      }
    };

    const handleCreate = () => {
      leftDrawerOpen.value = false;
      localData.data.unshift({
        title: "",
        timestamp: Date.now(),
        history: [],
      });
      localData.selectedIndex = 0;
    };

    const handleDelete = (index) => {
      if (localData.data.length > 0) {
        localData.data.pop(index);
      }
      if (localData.data.length === 0) {
        localData.data.push({
          title: "",
          timestamp: 0,
          history: [],
        });
        localData.selectedIndex = 0;
      }
    };
    const handleKeyPress = (event) => {
      if (event.ctrlKey && ["b", "B"].includes(event.key)) {
        leftDrawerOpen.value = !leftDrawerOpen.value;
      }
    };

    onMounted(() => {
      window.addEventListener("keydown", handleKeyPress);
      handleCreate();
    });

    onBeforeUnmount(() => {
      window.removeEventListener("keydown", handleKeyPress);
    });

    return {
      tab,
      version,
      localData,
      serverConfig,
      leftDrawerOpen,
      truncate,
      handleSave,
      handleCreate,
      handleDelete,
      toggleLeftDrawer: () => {
        leftDrawerOpen.value = !leftDrawerOpen.value;
      },
    };
  },
});
</script>
<style lang="scss" scoped>
.config-form {
  padding: 2px;
}
</style>
