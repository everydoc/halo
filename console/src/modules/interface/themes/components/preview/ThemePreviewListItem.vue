<script lang="ts" setup>
import LazyImage from "@/components/image/LazyImage.vue";
import type { Theme } from "@halo-dev/api-client";
import { VEntity, VEntityField, VTag, VButton } from "@halo-dev/components";
import { toRefs } from "vue";
import { useThemeLifeCycle } from "../../composables/use-theme";

const props = withDefaults(
  defineProps<{
    theme: Theme;
    isSelected?: boolean;
  }>(),
  {
    isSelected: false,
  }
);

const emit = defineEmits<{
  (event: "open-settings"): void;
}>();

const { theme } = toRefs(props);

const { isActivated, handleActiveTheme } = useThemeLifeCycle(theme);
</script>

<template>
  <VEntity :is-selected="isSelected">
    <template #start>
      <VEntityField>
        <template #description>
          <div class="w-32">
            <div
              class="group aspect-w-4 aspect-h-3 block w-full overflow-hidden rounded border bg-gray-100"
            >
              <LazyImage
                :key="theme.metadata.name"
                :src="theme.spec.logo"
                :alt="theme.spec.displayName"
                classes="pointer-events-none object-cover group-hover:opacity-75"
              >
                <template #loading>
                  <div
                    class="flex h-full items-center justify-center object-cover"
                  >
                    <span class="text-xs text-gray-400">
                      {{ $t("core.common.status.loading") }}...
                    </span>
                  </div>
                </template>
                <template #error>
                  <div
                    class="flex h-full items-center justify-center object-cover"
                  >
                    <span class="text-xs text-red-400">
                      {{ $t("core.common.status.loading_error") }}
                    </span>
                  </div>
                </template>
              </LazyImage>
            </div>
          </div>
        </template>
      </VEntityField>
      <VEntityField
        :title="theme.spec.displayName"
        :description="theme.spec.version"
      >
        <template #extra>
          <VTag v-if="isActivated">
            {{ $t("core.common.status.activated") }}
          </VTag>
        </template>
      </VEntityField>
    </template>

    <template #dropdownItems>
      <VButton
        v-if="!isActivated"
        v-close-popper
        block
        type="secondary"
        @click="handleActiveTheme"
      >
        {{ $t("core.common.buttons.active") }}
      </VButton>
      <VButton
        v-close-popper
        block
        type="default"
        @click="emit('open-settings')"
      >
        {{ $t("core.common.buttons.setting") }}
      </VButton>
    </template>
  </VEntity>
</template>
