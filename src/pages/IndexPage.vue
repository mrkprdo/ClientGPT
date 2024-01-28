<template>
  <q-page class="q-pa-sm">
    <div
      id="scrollContainer"
      class="query-response row"
      style="overflow-y: auto"
    >
      <div class="col"></div>
      <div class="col-10">
        <div
          class="row"
          v-for="(history, index) in localData.data[
            Number(localData.selectedIndex)
          ]?.history"
          :key="`${history.timestamp}_${index}`"
        >
          <q-card
            v-if="history.role !== 'system'"
            dark
            flat
            class="col queries"
          >
            <div style="margin: 1rem">
              <span
                class="text-subtitle1"
                :class="
                  history.role === 'user' ? 'text-warning' : 'text-secondary'
                "
              >
                {{ history.role }}:
              </span>
              <vue-markdown
                :source="history.content"
                style="white-space: pre-wrap; overflow-x: auto"
              />
            </div>
          </q-card>
        </div>
      </div>
      <div class="col"></div>
    </div>
    <div class="row">
      <div class="col"></div>
      <q-input
        v-model="userInput"
        dark
        autogrow
        filled
        clearable
        type="textarea"
        rows="2"
        placeholder="Message ClientGPT..."
        class="col-7"
        style="overflow-y: auto; resize: none !important"
      >
        <!-- <template v-slot:hint> Field hint </template> -->
        <template v-slot:append>
          <q-btn
            v-if="!waiting"
            round
            dense
            flat
            icon="send"
            color="warning"
            @click="sendMessage"
            title="Shift+Enter"
          />
          <q-spinner-hourglass v-else color="secondary" size="1em" />
        </template>
      </q-input>
      <div class="col"></div>
    </div>
  </q-page>
  <q-footer
    elevated
    bordered
    style="background-color: #1d1d1d"
    class="footer-field"
  >
    <span style="color: #6d6d6d" class="text-subtitle3">
      Any GPT can make mistakes. Consider checking important information.
    </span>
  </q-footer>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, defineComponent } from "vue";
import { useConfigStore, useLocalDataStore } from "../stores/config";
import VueMarkdown from "vue-markdown-render";
import axios from "axios";
import OpenAI from "openai";

export default defineComponent({
  name: "IndexPage",
  components: {
    VueMarkdown,
  },
  setup() {
    const userInput = ref("");
    const streamText = ref("");
    const waiting = ref(false);
    const config = useConfigStore();
    const localData = useLocalDataStore();

    const handleKeyPress = (event) => {
      // Check if the Shift + Enter key combination is pressed
      if (event.shiftKey && event.keyCode === 13) {
        event.preventDefault();
        if (!waiting.value) {
          sendMessage();
        }
      }
    };

    const appendMessage = (sender, message) => {
      if (
        localData.data[Number(localData.selectedIndex)].history.length === 0
      ) {
        localData.data[Number(localData.selectedIndex)].history.push({
          role: "system",
          content: config.role,
        });
        localData.data[Number(localData.selectedIndex)].title = message;
      }
      switch (sender) {
        case "You":
          localData.data[Number(localData.selectedIndex)].history.push({
            role: "user",
            content: message,
          });
          break;
        case "AI":
          localData.data[Number(localData.selectedIndex)].history.push({
            role: "assistant",
            content: message,
          });
          break;
      }
    };

    const finalizeStreamedText = (message) => {
      // assume last chat history to be streamed
      const lastIndex =
        localData.data[Number(localData.selectedIndex)].history.length - 1;
      localData.data[Number(localData.selectedIndex)].history[lastIndex] = {
        role: "assistant",
        content: message,
      };
    };
    const openai = new OpenAI({
      baseURL: `http://${config.host}:${config.port}/v1`,
      apiKey: "not-needed", // This is the default and can be omitted
      dangerouslyAllowBrowser: true,
    });
    const sendMessage = async () => {
      waiting.value = true;
      if (userInput.value.trim() === "") {
        return;
      }
      const userMessage = userInput.value;
      appendMessage("You", userMessage);
      userInput.value = "";

      try {
        switch (config.apiCall) {
          case "axios":
            const apiUrl = `http://${config.host}:${config.port}/v1/chat/completions`;
            const headers = {
              "Content-Type": "application/json",
            };

            const data = {
              messages: localData.data[Number(localData.selectedIndex)].history,
              temperature: 0.7,
              max_tokens: -1,
              stream: false, // stream not supported in axios (client-side request)
            };
            const timeout = 20000;
            const response = await axios.post(
              apiUrl,
              data,
              { headers },
              timeout
            );
            const aiMessage = response.data.choices[0].message.content;
            appendMessage("AI", aiMessage);
            break;
          case "openai":
            const stream = openai.beta.chat.completions.stream({
              model: "local-model",
              messages: localData.data[Number(localData.selectedIndex)].history,
              stream: true,
            });
            streamText.value = "...";
            appendMessage("AI", streamText.value);
            streamText.value = "";

            stream.on("content", (delta, snapshot) => {
              streamText.value = `${streamText.value}${delta}`;
              finalizeStreamedText(streamText.value);
            });

            // const chatCompletion = await stream.finalChatCompletion();
            // console.log(chatCompletion); // {id: "…", choices: […], …}
            break;
          default:
        }
        // scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight;
      } catch (error) {
        console.error("Error:", error);
      }
      waiting.value = false;
    };

    onMounted(() => {
      window.addEventListener("keydown", handleKeyPress);
    });

    onBeforeUnmount(() => {
      window.removeEventListener("keydown", handleKeyPress);
    });

    return {
      waiting,
      localData,
      userInput,
      sendMessage,
    };
  },
});
</script>
<style lang="scss" scoped>
.query-response {
  height: 80vh;
}

.queries {
  margin: 0.4rem;
}
.footer-field {
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #3b3b3b;
}
</style>
