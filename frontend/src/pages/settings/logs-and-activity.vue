<template>
  <settings-page page-title="settingsSections.logs.name">
    <template #content>
      <v-col md="6" class="pt-0 pb-4">
        <v-fade-transition group>
          <h2 key="logs-title" class="text-h6 mb-2">
            {{ t('settings.logsAndActivity.logs') }}
          </h2>
          <v-list
            v-if="logs.length > 0"
            key="log-list"
            lines="two"
            class="mb-2">
            <v-list-item
              v-for="file in logs"
              :key="file.Name ?? undefined"
              :href="getLogFileLink(file.Name ?? '')"
              :title="file.Name ?? undefined"
              :subtitle="getFormattedLogDate(file.DateModified)"
              target="_blank"
              rel="noopener">
              <template #prepend>
                <v-avatar>
                  <v-icon>
                    <i-mdi-file />
                  </v-icon>
                </v-avatar>
              </template>
              <template #append>
                <v-icon>
                  <i-mdi-open-in-new />
                </v-icon>
              </template>
            </v-list-item>
          </v-list>
          <v-card v-else>
            <v-card-title>
              {{ t('settings.logsAndActivity.noLogsFound') }}
            </v-card-title>
          </v-card>
        </v-fade-transition>
      </v-col>
      <v-col md="6" class="pt-0 pb-4">
        <v-fade-transition group>
          <h2 key="activity-title" class="text-h6 mb-2">
            {{ t('settings.logsAndActivity.activity') }}
          </h2>
          <v-list
            v-if="activityList.length > 0"
            key="activity-list"
            lines="two"
            class="mb-2">
            <v-list-item
              v-for="activity in activityList"
              :key="activity.Id"
              :title="activity.Name"
              :subtitle="activity.ShortOverview ?? undefined">
              <template #prepend>
                <v-avatar :color="getColorFromSeverity(activity.Severity)">
                  <v-icon :icon="getIconFromActivityType(activity.Type)" />
                </v-avatar>
              </template>
              <template #append>
                <v-list-item-subtitle class="text-capitalize-first-letter">
                  {{ getFormattedActivityDate(activity.Date) }}
                </v-list-item-subtitle>
              </template>
            </v-list-item>
          </v-list>
          <v-card v-else>
            <v-card-title>
              {{ t('settings.logsAndActivity.noActivityFound') }}
            </v-card-title>
          </v-card>
        </v-fade-transition>
      </v-col>
    </template>
  </settings-page>
</template>

<route lang="yaml">
meta:
  admin: true
</route>

<script setup lang="ts">
import { ref } from 'vue';
import { useTheme } from 'vuetify';
import {
  ActivityLogEntry,
  LogFile,
  LogLevel
} from '@jellyfin/sdk/lib/generated-client';
import { getActivityLogApi } from '@jellyfin/sdk/lib/utils/api/activity-log-api';
import { getSystemApi } from '@jellyfin/sdk/lib/utils/api/system-api';
import { format, formatRelative, parseJSON } from 'date-fns';
import { useI18n } from 'vue-i18n';
import { useRoute } from 'vue-router';
import IMdiLogin from 'virtual:icons/mdi/login';
import IMdiLogout from 'virtual:icons/mdi/logout';
import IMdiLock from 'virtual:icons/mdi/lock';
import IMdiPlay from 'virtual:icons/mdi/play';
import IMdiStop from 'virtual:icons/mdi/stop';
import IMdiHelp from 'virtual:icons/mdi/help';
import { useDateFns, useRemote } from '@/composables';

const { t } = useI18n();
const route = useRoute();
const remote = useRemote();
const theme = useTheme();

route.meta.title = t('settingsSections.logs.name');

const logs = ref<LogFile[]>([]);
const activityList = ref<ActivityLogEntry[]>([]);

/**
 * Return a UI colour given log severity
 */
function getColorFromSeverity(severity: LogLevel | undefined): string {
  switch (severity) {
    case LogLevel.Trace: {
      return theme.current.value.colors.success;
    }
    case LogLevel.Debug: {
      return theme.current.value.colors.accent;
    }
    case LogLevel.Information: {
      return theme.current.value.colors.info;
    }
    case LogLevel.Warning: {
      return theme.current.value.colors.warning;
    }
    case LogLevel.Error: {
      return theme.current.value.colors.error;
    }
    case LogLevel.Critical: {
      return theme.current.value.colors.secondary;
    }
    default: {
      return theme.current.value.colors.primary;
    }
  }
}

/**
 * Gets an icon given an activity type
 */
function getIconFromActivityType(
  type: string | undefined | null
): typeof IMdiLogin {
  switch (type) {
    case 'SessionStarted': {
      return IMdiLogin;
    }
    case 'SessionEnded': {
      return IMdiLogout;
    }
    case 'UserPasswordChanged': {
      return IMdiLock;
    }
    case 'VideoPlayback': {
      return IMdiPlay;
    }
    case 'VideoPlaybackStopped': {
      return IMdiStop;
    }
    default: {
      return IMdiHelp;
    }
  }
}

/**
 * Format activitydates
 */
function getFormattedActivityDate(date: string | undefined): string {
  return date
    ? useDateFns(formatRelative, parseJSON(date), new Date()).value
    : '';
}

/**
 * Format log dates
 */
function getFormattedLogDate(date: string | undefined): string {
  return date ? useDateFns(format, parseJSON(date), 'Ppp').value : '';
}

/**
 * Creates a link to the given type of log file
 */
function getLogFileLink(name: string): string {
  return `${remote.sdk.api?.basePath}/System/Logs/Log?name=${name}&api_key=${remote.auth.currentUserToken}`;
}

/**
 * Fetches logs
 */
async function fetchLogs(): Promise<void> {
  try {
    logs.value = (
      await remote.sdk.newUserApi(getSystemApi).getServerLogs()
    ).data;
  } catch (error) {
    console.error(error);
  }
}

/**
 * Fetches activities
 */
async function fetchActivity(): Promise<void> {
  const minDate = new Date();

  minDate.setDate(minDate.getDate() - 7);

  try {
    activityList.value =
      (
        await remote.sdk
          .newUserApi(getActivityLogApi)
          .getLogEntries({ minDate: minDate.toISOString() })
      ).data.Items ?? [];
  } catch (error) {
    console.error(error);
  }
}

await fetchLogs();
await fetchActivity();
</script>
