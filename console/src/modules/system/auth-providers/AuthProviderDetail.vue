<script lang="ts" setup>
import { computed, ref } from "vue";
import { apiClient } from "@/utils/api-client";
import { useQuery } from "@tanstack/vue-query";
import type { AuthProvider, Setting } from "@halo-dev/api-client";
import { useRoute } from "vue-router";
import {
  Toast,
  VAvatar,
  VButton,
  VCard,
  VPageHeader,
  VTabbar,
} from "@halo-dev/components";
import { useSettingFormConvert } from "@/composables/use-setting-form";
import { useI18n } from "vue-i18n";

const route = useRoute();
const { t } = useI18n();

const tabs = ref<{ id: string; label: string }[]>([
  {
    id: "detail",
    label: t("core.identity_authentication.tabs.detail"),
  },
]);

const activeTab = ref<string>("detail");

const { data: authProvider } = useQuery<AuthProvider>({
  queryKey: ["auth-provider", route.params.name],
  queryFn: async () => {
    const { data } =
      await apiClient.extension.authProvider.getauthHaloRunV1alpha1AuthProvider(
        {
          name: route.params.name as string,
        }
      );
    return data;
  },
  onSuccess(data) {
    if (data.spec.settingRef?.name) {
      tabs.value.push({
        id: "setting",
        label: t("core.identity_authentication.tabs.setting"),
      });
    }
  },
  refetchOnWindowFocus: false,
  enabled: computed(() => !!route.params.name),
});

// setting
const saving = ref(false);
const group = computed(
  () => authProvider.value?.spec.settingRef?.group as string
);

const { data: setting, refetch: handleFetchSettings } = useQuery<Setting>({
  queryKey: [
    "auth-provider-setting",
    authProvider.value?.spec.settingRef?.name,
  ],
  queryFn: async () => {
    const { data } = await apiClient.extension.setting.getv1alpha1Setting(
      {
        name: authProvider.value?.spec.settingRef?.name as string,
      },
      {
        mute: true,
      }
    );
    return data;
  },
  refetchOnWindowFocus: false,
  enabled: computed(() => !!authProvider.value?.spec.settingRef?.name),
});

const { data: configMap, refetch: handleFetchConfigMap } = useQuery({
  queryKey: [
    "auth-provider-configMap",
    authProvider.value?.spec.configMapRef?.name,
  ],
  queryFn: async () => {
    const { data } = await apiClient.extension.configMap.getv1alpha1ConfigMap(
      {
        name: authProvider.value?.spec.configMapRef?.name as string,
      },
      {
        mute: true,
      }
    );
    return data;
  },
  retry: 0,
  refetchOnWindowFocus: false,
  onError: async () => {
    const data = {};
    data[group.value] = "";
    await apiClient.extension.configMap.createv1alpha1ConfigMap({
      configMap: {
        apiVersion: "v1alpha1",
        data: data,
        kind: "ConfigMap",
        metadata: {
          name: authProvider.value?.spec.configMapRef?.name as string,
        },
      },
    });

    await handleFetchConfigMap();
  },
  enabled: computed(
    () => !!authProvider.value?.spec.configMapRef?.name && !!setting.value
  ),
});

const { configMapFormData, formSchema, convertToSave } = useSettingFormConvert(
  setting,
  configMap,
  group
);

const handleSaveConfigMap = async () => {
  saving.value = true;
  const configMapToUpdate = convertToSave();
  if (!configMapToUpdate || !authProvider?.value) {
    saving.value = false;
    return;
  }

  await apiClient.extension.configMap.updatev1alpha1ConfigMap({
    name: authProvider.value.spec.configMapRef?.name as string,
    configMap: configMapToUpdate,
  });

  Toast.success(t("core.common.toast.save_success"));

  await handleFetchSettings();
  await handleFetchConfigMap();
  saving.value = false;
};
</script>

<template>
  <VPageHeader :title="authProvider?.spec.displayName">
    <template #icon>
      <VAvatar
        v-if="authProvider"
        :src="authProvider.spec.logo"
        :alt="authProvider.spec.displayName"
        class="mr-2"
        size="sm"
      />
    </template>
  </VPageHeader>

  <div class="m-0 md:m-4">
    <VCard :body-class="['!p-0']">
      <template #header>
        <VTabbar
          v-model:active-id="activeTab"
          :items="tabs"
          class="w-full !rounded-none"
          type="outline"
        ></VTabbar>
      </template>
      <div class="bg-white">
        <div v-if="activeTab === 'detail'">
          <dl class="divide-y divide-gray-100">
            <div
              class="bg-white px-4 py-5 hover:bg-gray-50 sm:grid sm:grid-cols-6 sm:gap-4 sm:px-6"
            >
              <dt class="text-sm font-medium text-gray-900">
                {{
                  $t("core.identity_authentication.detail.fields.display_name")
                }}
              </dt>
              <dd class="mt-1 text-sm text-gray-900 sm:col-span-3 sm:mt-0">
                {{ authProvider?.spec.displayName }}
              </dd>
            </div>
            <div
              class="bg-white px-4 py-5 hover:bg-gray-50 sm:grid sm:grid-cols-6 sm:gap-4 sm:px-6"
            >
              <dt class="text-sm font-medium text-gray-900">
                {{
                  $t("core.identity_authentication.detail.fields.description")
                }}
              </dt>
              <dd class="mt-1 text-sm text-gray-900 sm:col-span-3 sm:mt-0">
                {{ authProvider?.spec.description }}
              </dd>
            </div>
            <div
              class="bg-white px-4 py-5 hover:bg-gray-50 sm:grid sm:grid-cols-6 sm:gap-4 sm:px-6"
            >
              <dt class="text-sm font-medium text-gray-900">
                {{ $t("core.identity_authentication.detail.fields.website") }}
              </dt>
              <dd class="mt-1 text-sm text-gray-900 sm:col-span-3 sm:mt-0">
                <a
                  v-if="authProvider?.spec.website"
                  :href="authProvider?.spec.website"
                  target="_blank"
                >
                  {{ authProvider.spec.website }}
                </a>
                <span v-else>
                  {{ $t("core.common.text.none") }}
                </span>
              </dd>
            </div>
            <div
              class="bg-white px-4 py-5 hover:bg-gray-50 sm:grid sm:grid-cols-6 sm:gap-4 sm:px-6"
            >
              <dt class="text-sm font-medium text-gray-900">
                {{ $t("core.identity_authentication.detail.fields.help_page") }}
              </dt>
              <dd class="mt-1 text-sm text-gray-900 sm:col-span-3 sm:mt-0">
                <a
                  v-if="authProvider?.spec.helpPage"
                  :href="authProvider?.spec.helpPage"
                  target="_blank"
                >
                  {{ authProvider.spec.helpPage }}
                </a>
                <span v-else>{{ $t("core.common.text.none") }}</span>
              </dd>
            </div>
            <div
              class="bg-white px-4 py-5 hover:bg-gray-50 sm:grid sm:grid-cols-6 sm:gap-4 sm:px-6"
            >
              <dt class="text-sm font-medium text-gray-900">
                {{
                  $t(
                    "core.identity_authentication.detail.fields.authentication_url"
                  )
                }}
              </dt>
              <dd class="mt-1 text-sm text-gray-900 sm:col-span-3 sm:mt-0">
                {{ authProvider?.spec.authenticationUrl }}
              </dd>
            </div>
          </dl>
        </div>
        <div v-if="activeTab === 'setting'" class="bg-white p-4">
          <div>
            <FormKit
              v-if="group && formSchema && configMapFormData"
              :id="group"
              v-model="configMapFormData[group]"
              :name="group"
              :actions="false"
              :preserve="true"
              type="form"
              @submit="handleSaveConfigMap"
            >
              <FormKitSchema
                :schema="formSchema"
                :data="configMapFormData[group]"
              />
            </FormKit>
          </div>
          <div class="pt-5">
            <div class="flex justify-start">
              <VButton
                :loading="saving"
                type="secondary"
                @click="$formkit.submit(group || '')"
              >
                {{ $t("core.common.buttons.save") }}
              </VButton>
            </div>
          </div>
        </div>
      </div>
    </VCard>
  </div>
</template>
