<template>
  <v-app>
    <v-app-bar title="Personal LLM Chat" density="compact" :height="48" >
      <v-btn @click="toggleTheme" density="compact">toggle theme</v-btn>
    </v-app-bar>
    <v-main>
      <router-view />
    </v-main>
  </v-app>
</template>

<script setup>
import { useTheme } from 'vuetify'

const theme = useTheme()

function toggleTheme () {
  theme.global.name.value = theme.global.current.value.dark ? 'light' : 'dark'
}

</script>
