<template>
  <v-card class="llm-panel">
    <v-card-title>
      <div>
        <div class="headline">{{ __('ui_llm_panel_title') }}</div>
        <div class="caption grey--text">{{ __('ui_llm_panel_desc') }}</div>
      </div>
    </v-card-title>
    <v-card-text>
      <v-alert type="warning" outline v-if="!apiKey">
        {{ __('ui_llm_missing_key') }}
      </v-alert>
      <v-layout align-center justify-space-between class="mb-2">
        <div class="caption grey--text">{{ __('ui_llm_model_label') }}: {{ model }}</div>
        <v-btn small flat @click="goToOptions">
          {{ __('ui_llm_open_options') }}
        </v-btn>
      </v-layout>
      <v-textarea
        v-model="prompt"
        :label="__('ui_llm_prompt_label')"
        auto-grow
        rows="3"
        hide-details
      ></v-textarea>
      <v-btn
        class="mt-2"
        color="primary"
        :loading="loading"
        :disabled="loading || !prompt || !apiKey"
        @click="sendPrompt"
      >
        {{ __('ui_llm_send') }}
      </v-btn>
      <v-alert v-if="error" type="error" outline class="mt-3">
        {{ error }}
      </v-alert>
      <v-card class="mt-3" flat v-if="response">
        <v-card-text class="llm-response">
          <pre>{{ response }}</pre>
        </v-card-text>
      </v-card>
    </v-card-text>
  </v-card>
</template>

<script>
import __ from '@/common/i18n'
import storage from '@/common/storage'

export default {
  props: {
    lists: {
      type: Array,
      required: true,
    },
    domainGroups: {
      type: Array,
      required: true,
    },
  },
  data() {
    return {
      prompt: '',
      response: '',
      loading: false,
      error: null,
      apiKey: '',
      model: 'gpt-4o-mini',
    }
  },
  created() {
    this.loadSettings()
  },
  methods: {
    __,
    async loadSettings() {
      const {apiKey, model} = await storage.getLlmSettings()
      this.apiKey = apiKey
      this.model = model
    },
    goToOptions() {
      this.$router.push('/app/options')
    },
    getDomain(url) {
      try {
        return new URL(url).hostname
      } catch (error) {
        return ''
      }
    },
    buildContext() {
      const totalTabs = this.lists.reduce((sum, list) => sum + list.tabs.length, 0)
      const domainSummary = this.domainGroups.map(group => ({
        domain: group.domain,
        count: group.tabs.length,
      }))
      const lists = this.lists.map(list => ({
        title: list.title || this.__('ui_untitled'),
        createdAt: list.time,
        tabs: list.tabs.map(tab => ({
          title: tab.title,
          url: tab.url,
          domain: this.getDomain(tab.url),
        })),
      }))
      return {
        totalLists: this.lists.length,
        totalTabs,
        domainSummary,
        lists,
      }
    },
    async sendPrompt() {
      this.error = null
      this.response = ''
      this.loading = true
      try {
        const {apiKey, model} = await storage.getLlmSettings()
        this.apiKey = apiKey
        this.model = model
        if (!apiKey) {
          this.error = this.__('ui_llm_missing_key')
          return
        }
        const context = this.buildContext()
        const payload = {
          model,
          messages: [
            {
              role: 'system',
              content: 'You are a tab organization assistant. Provide concise, actionable guidance.',
            },
            {
              role: 'user',
              content: `User prompt: ${this.prompt}\n\nTab data:\n${JSON.stringify(context, null, 2)}`,
            },
          ],
          temperature: 0.2,
        }
        const response = await fetch('https://api.openai.com/v1/chat/completions', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${apiKey}`,
          },
          body: JSON.stringify(payload),
        })
        if (!response.ok) {
          const errorBody = await response.json().catch(() => ({}))
          const message = errorBody.error && errorBody.error.message
            ? errorBody.error.message
            : response.statusText
          throw new Error(message)
        }
        const data = await response.json()
        this.response = data.choices && data.choices[0] && data.choices[0].message
          ? data.choices[0].message.content
          : this.__('ui_llm_empty_response')
      } catch (error) {
        this.error = error.message
      } finally {
        this.loading = false
      }
    },
  }
}
</script>

<style scoped>
.llm-panel {
  position: sticky;
  top: 12px;
}
.llm-response {
  white-space: pre-wrap;
}
.llm-response pre {
  margin: 0;
  font-family: inherit;
}
</style>
