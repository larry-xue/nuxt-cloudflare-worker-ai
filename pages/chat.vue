<template>
  <UDashboardPage class="pb-8">
    <UDashboardPanel grow>
      <UDashboardNavbar title="Chat">
        <template #right>
          <UFormGroup label="" name="model" class="w-100">
          <USelect v-model="state .model" :options="['gpt-3.5-turbo', 'gpt-4']">
          </USelect>
        </UFormGroup>
        </template>
      </UDashboardNavbar>

      <UDivider class="mb-4" />

      <div class="w-full h-full overflow-x-hidden overflow-y-auto">
        <ClientOnly>
          <div v-for="msg in messages" :ref="(el) => messageBoxRef = el">
            <Message :is-me="msg.isMe" :message="msg.message" />
          </div>
        </ClientOnly>
      </div>

      <UForm :state="state" :validate-on="['submit']" class="flex gap-1 flex-nowrap w-full box-border px-2">
        <UFormGroup label="" name="input" class="shrink w-full">
          <UTextarea autoresize :rows="1" @keyup.enter.exact="sendMessage" v-model="state.message" :maxrows="10"
            placeholder="Type a message..." :ref="(el) => textareaRef = el" />
        </UFormGroup>
        <UFormGroup label="" name="output" class="shrink-0 w-auto flex items-end">
          <UButton v-bind:loading="pending" @click="sendMessage" label="Send" class="w-full" />
        </UFormGroup>
      </UForm>
    </UDashboardPanel>
  </UDashboardPage>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import type { MessageProps } from '@/components/message/Message.client.vue';
const toast = useToast()
const config = useRuntimeConfig()

const state = ref({
  model: 'gpt-3.5-turbo',
  message: ''
})
const messageBoxRef = ref<Element | ComponentPublicInstance>()
const textareaRef = ref<Element | ComponentPublicInstance>()
const pending = ref(false)

const messages = ref<MessageProps[]>([
  {
    isMe: false,
    message: {
      from: {
        name: 'Bot',
        avatar: 'https://i.pravatar.cc/300'
      },
      body: 'Hello, how can I help you today?',
    },
    isPending: false
  }
])


function getRecentMessage() {
  // get latest 3 messages
  return messages.value.slice(-3).filter(m => m.isMe || m.gotResponse).map(m => {
    return { role: m.isMe ? 'user' : 'assistant', content: m.message.body }
  })
}

async function getAiResponse() {
  const data = await $fetch<{
    response: string
  }>(
    '/api/chat', {
    method: 'POST',
    body: getRecentMessage()
  })
  return data.response || "I'm sorry, I don't have an answer for that."
}

function sendMessage() {
  if (pending.value) return
  if (!state.value.message) {
    // notify
    return toast.add({
      title: 'Notification',
      description: 'Please enter a message.',
      color: 'gray',
      icon: 'i-heroicons-exclamation-circle',
      timeout: 3000,
    })
  }

  const message = {
    isMe: true,
    message: {
      from: {
        name: 'Me',
      },
      body: state.value.message
    },
    isPending: false,
  }
  messages.value.push(message)

  // get ai response
  const botResponse = {
    isMe: false,
    message: {
      from: {
        name: 'Bot',
        avatar: 'https://i.pravatar.cc/300'
      },
      body: '...',
    },
    isPending: true,
    gotResponse: false
  }

  pending.value = true
  getAiResponse().then((response) => {
    messages.value[messages.value.length - 1].message.body = response
    messages.value[messages.value.length - 1].gotResponse = true
  }).catch((e) => {
    toast.add({ title: 'Error', description: 'Something went wrong...', color: 'red', icon: 'i-heroicons-exclamation-circle' })
    messages.value[messages.value.length - 1].message.body = "I'm sorry, something went wrong, please try again."
  }).finally(() => {
    pending.value = false
  })

  messages.value.push(botResponse)

  nextTick(() => {
    // focus textarea
  })

  state.value.message = ''
}
</script>
