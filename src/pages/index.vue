<template>
  <div class="pa-4">
    <div>
      <v-card class="mx-auto" max-width="800" density="compact">
        <v-card-title>대화목록</v-card-title>
        <v-divider></v-divider>

        <!-- https://vuetifyjs.com/en/components/virtual-scroller/#item-height -->
        <v-virtual-scroll :items="dialogData.items" height="320" item-height="48">

          <template v-slot:default="{ item }">

            <v-list-item title="유저"  v-show="item.user_type == 'user'">
              <template v-slot:prepend>
                <v-avatar color="primary">
                    <span class="text-h5">🙍🏻‍♂️</span>
                  </v-avatar>
              </template>
              <v-list-item-subtitle class="font-weight-medium">
              {{item.content }}
              </v-list-item-subtitle>
            </v-list-item>

            <v-list-item title="LLM"  v-show="item.user_type == 'llm'">
              <template v-slot:prepend>
                <v-avatar color="warning">
                    <span class="text-h5">🤖</span>
                  </v-avatar>
              </template>
              <v-list-item-subtitle>
              {{item.content }}
              </v-list-item-subtitle>
            </v-list-item>
          </template>
          

        </v-virtual-scroll>

        <v-card-text>
          <v-row>
            <v-col cols="10">
              <v-textarea bg-color="amber-lighten-4" color="orange orange-darken-4" label="" :rows="3"
                v-model="dialogData.userDialog"
                :loading="dialogData.userDialogLoading"
                density="compact" :hide-details="true" 
                @keyup.alt.enter="userEnter" @keyup.cmd.enter="userEnter"
                />
            </v-col>

            <v-col cols="2" >
              <v-btn class="h-50 mt-n1 mb-2 text-capitalize text-body-2	" block density="compact"
                  
                  @click="userEnter">
                입력<br/>
                <template v-if="config.is_mac" >
                  (cmd + Enter)
                </template>
                <template v-else>
                  (Alt + Enter)
                </template>
                
              </v-btn>

              <v-btn class="h-50" block density="compact" @click="dialogClear">
                Clear
              </v-btn>
            </v-col>

          </v-row>

        </v-card-text>

        
      </v-card>

    </div>

    <v-btn @click="debugPrint">
      debug
    </v-btn>

    <v-overlay class="align-center justify-center" v-model="dialogData.modelLoading.loading" >
      <v-card title="Loading" width="500">
        <v-card-text>
        <v-progress-linear color="light-blue" height="10" :model-value="dialogData.modelLoading.progress * 10" striped></v-progress-linear>
        {{ dialogData.modelLoading.text }}
        </v-card-text>
      </v-card>
    </v-overlay>
    
  </div>
</template>
  
<script setup>
import { MLCEngine } from "@mlc-ai/web-llm";

import { reactive, onMounted  } from "vue";



const is_mac = /(Mac|iPhone|iPod|iPad)/i.test(navigator.userAgent);
const config = reactive({is_mac});

const dialogData = reactive({
  userDialog: "", userDialogLoading: false,

  // items = {user_type: user, body: "대화내용"}
  // user_type : "llm" or "user"
  // content : 사용자의 경우 작성내용, llm 의 경우 그에 따른 답변
  items: [],

  messages: [
    { role: "system", content: "Please answer in Korean." }
  ],
  modelLoading: {
    loading: true,
    progress: 0, text: ""
  },
});

function dialogClear(){
  dialogData.items = [];
  dialogData.messages = [
    { role: "system", content: "Please answer in Korean." }
  ];
}

const initProgressCallback = (initProgress) => {
  dialogData.modelLoading.progress = initProgress.progress;
  dialogData.modelLoading.text = initProgress.text;
  if(initProgress.progress >= 1){
    dialogData.modelLoading.loading = false;
  }
  console.log(initProgress);
}


const engine = new MLCEngine({
  initProgressCallback: initProgressCallback
});

onMounted(async () => {
  // shader-f16 check
  // https://developer.chrome.com/blog/new-in-webgpu-120 
  const adapter = await navigator.gpu.requestAdapter();
  const hasShaderF16 = adapter.features.has("shader-f16");

  const header = hasShaderF16
    ? `enable f16;
      alias min16float = f16;`
    : `alias min16float = f32;`;

    
  // "Qwen2.5-0.5B-Instruct-q4f16_1-MLC":"Qwen2.5-0.5B-Instruct-q4f32_1-MLC"
  // "Qwen2.5-1.5B-Instruct-q4f16_1-MLC":"Qwen2.5-1.5B-Instruct-q4f32_1-MLC"
  // 
  const selectedModel = hasShaderF16
    ?"Qwen2.5-0.5B-Instruct-q4f16_1-MLC":"Qwen2.5-0.5B-Instruct-q4f32_1-MLC"

  await engine.reload(selectedModel);
});



async function userEnter(){
  const content = dialogData.userDialog;
  dialogData.userDialog = "";
  dialogData.userDialogLoading = true;

  dialogData.items.push({
    user_type: "user", content
  })

  dialogData.messages.push(
    { role: "user", content }
  );
  await queryLLM();
}


async function queryLLM(){
  
  dialogData.items.push(
    { user_type: "llm", content: "" }
  );

  // Chunks is an AsyncGenerator object
  const chunks = await engine.chat.completions.create({
    messages: dialogData.messages,
    temperature: 0.3,
    stream: true, // <-- Enable streaming
    stream_options: { include_usage: true },
  });

  let reply = "";
  for await (const chunk of chunks) {
    reply += chunk.choices[0]?.delta.content || "";
    dialogData.items[dialogData.messages.length - 1]["content"] = reply;
    console.log(reply);
    if (chunk.usage) {
      console.log(chunk.usage); // only last chunk has usage
    }
  }

  dialogData.userDialogLoading = false;

  const fullReply = await engine.getMessage();
  dialogData.messages.push(
    { role: "assistant", content: fullReply }
  );
  
}

function debugPrint(){
  debugger
  console.log(dialogData)
}

</script>
