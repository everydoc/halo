<script lang="ts" setup>
// core libs
import { inject, ref, type Ref } from "vue";

// hooks
import { useSettingFormConvert } from "@/composables/use-setting-form";
import { apiClient } from "@/utils/api-client";

// components
import { Toast, VButton } from "@halo-dev/components";

// types
import type { ConfigMap, Plugin, Setting } from "@halo-dev/api-client";
import { useRouteParams } from "@vueuse/router";
import { useI18n } from "vue-i18n";

const { t } = useI18n();

const group = useRouteParams<string>("group");

const plugin = inject<Ref<Plugin | undefined>>("plugin");
const saving = ref(false);
const setting = ref<Setting>();
const configMap = ref<ConfigMap>();

const { configMapFormData, formSchema, convertToSave } = useSettingFormConvert(
  setting,
  configMap,
  group
);

const handleFetchSettings = async () => {
  if (!plugin?.value) return;
  const { data } = await apiClient.plugin.fetchPluginSetting({
    name: plugin.value.metadata.name,
  });
  setting.value = data;
};

const handleFetchConfigMap = async () => {
  if (!plugin?.value) return;
  const { data } = await apiClient.plugin.fetchPluginConfig({
    name: plugin.value.metadata.name,
  });
  configMap.value = data;
};

const handleSaveConfigMap = async () => {
  saving.value = true;
  const configMapToUpdate = convertToSave();
  if (!configMapToUpdate || !plugin?.value) {
    saving.value = false;
    return;
  }

  const { data: newConfigMap } = await apiClient.plugin.updatePluginConfig({
    name: plugin.value.metadata.name,
    configMap: configMapToUpdate,
  });

  Toast.success(t("core.common.toast.save_success"));

  await handleFetchSettings();
  configMap.value = newConfigMap;
  saving.value = false;
};

await handleFetchSettings();
await handleFetchConfigMap();
</script>
<template>
  <Transition mode="out-in" name="fade">
    <div class="bg-white p-4">
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
      <div v-permission="['system:plugins:manage']" class="pt-5">
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
  </Transition>
</template>
