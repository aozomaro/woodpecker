<template>
  <Settings
    :title="$t('repo.settings.registries.creds')"
    :desc="$t('repo.settings.registries.desc')"
    docs-url="docs/usage/registries"
  >
    <template #titleActions>
      <Button
        v-if="selectedRegistry"
        start-icon="back"
        :text="$t('repo.settings.registries.show')"
        @click="selectedRegistry = undefined"
      />
      <Button v-else start-icon="plus" :text="$t('repo.settings.registries.add')" @click="selectedRegistry = {}" />
    </template>

    <div v-if="!selectedRegistry" class="space-y-4 text-wp-text-100">
      <ListItem
        v-for="registry in registries"
        :key="registry.id"
        class="items-center !bg-wp-background-200 !dark:bg-wp-background-100"
      >
        <span>{{ registry.address }}</span>
        <IconButton
          icon="edit"
          class="ml-auto w-8 h-8"
          :title="$t('repo.settings.registries.edit')"
          @click="selectedRegistry = registry"
        />
        <IconButton
          icon="trash"
          class="w-8 h-8 hover:text-wp-control-error-100"
          :is-loading="isDeleting"
          :title="$t('repo.settings.registries.delete')"
          @click="deleteRegistry(registry)"
        />
      </ListItem>

      <div v-if="registries?.length === 0" class="ml-2">{{ $t('repo.settings.registries.none') }}</div>
    </div>

    <div v-else class="space-y-4">
      <form @submit.prevent="createRegistry">
        <InputField v-slot="{ id }" :label="$t('repo.settings.registries.address.address')">
          <!-- TODO: check input field Address is a valid address -->
          <TextField
            :id="id"
            v-model="selectedRegistry.address"
            :placeholder="$t('repo.settings.registries.address.placeholder')"
            required
            :disabled="isEditingRegistry"
          />
        </InputField>

        <InputField v-slot="{ id }" :label="$t('username')">
          <TextField :id="id" v-model="selectedRegistry.username" :placeholder="$t('username')" required />
        </InputField>

        <InputField v-slot="{ id }" :label="$t('password')">
          <TextField :id="id" v-model="selectedRegistry.password" :placeholder="$t('password')" required />
        </InputField>

        <div class="flex gap-2">
          <Button type="button" color="gray" :text="$t('cancel')" @click="selectedRegistry = undefined" />
          <Button
            type="submit"
            color="green"
            :is-loading="isSaving"
            :text="isEditingRegistry ? $t('repo.settings.registries.save') : $t('repo.settings.registries.add')"
          />
        </div>
      </form>
    </div>
  </Settings>
</template>

<script lang="ts" setup>
import { computed, inject, Ref, ref } from 'vue';
import { useI18n } from 'vue-i18n';

import Button from '~/components/atomic/Button.vue';
import IconButton from '~/components/atomic/IconButton.vue';
import ListItem from '~/components/atomic/ListItem.vue';
import InputField from '~/components/form/InputField.vue';
import TextField from '~/components/form/TextField.vue';
import Settings from '~/components/layout/Settings.vue';
import useApiClient from '~/compositions/useApiClient';
import { useAsyncAction } from '~/compositions/useAsyncAction';
import useNotifications from '~/compositions/useNotifications';
import { usePagination } from '~/compositions/usePaginate';
import { Repo } from '~/lib/api/types';
import { Registry } from '~/lib/api/types/registry';

const apiClient = useApiClient();
const notifications = useNotifications();
const i18n = useI18n();

const repo = inject<Ref<Repo>>('repo');
const selectedRegistry = ref<Partial<Registry>>();
const isEditingRegistry = computed(() => !!selectedRegistry.value?.id);

async function loadRegistries(page: number): Promise<Registry[] | null> {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  return apiClient.getRegistryList(repo.value.id, page);
}

const { resetPage, data: registries } = usePagination(loadRegistries, () => !selectedRegistry.value);

const { doSubmit: createRegistry, isLoading: isSaving } = useAsyncAction(async () => {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  if (!selectedRegistry.value) {
    throw new Error("Unexpected: Can't get registry");
  }

  if (isEditingRegistry.value) {
    await apiClient.updateRegistry(repo.value.id, selectedRegistry.value);
  } else {
    await apiClient.createRegistry(repo.value.id, selectedRegistry.value);
  }
  notifications.notify({
    title: i18n.t(
      isEditingRegistry.value ? 'repo.settings.registries.saved' : i18n.t('repo.settings.registries.created'),
    ),
    type: 'success',
  });
  selectedRegistry.value = undefined;
  resetPage();
});

const { doSubmit: deleteRegistry, isLoading: isDeleting } = useAsyncAction(async (_registry: Registry) => {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  const registryAddress = encodeURIComponent(_registry.address);
  await apiClient.deleteRegistry(repo.value.id, registryAddress);
  notifications.notify({ title: i18n.t('repo.settings.registries.deleted'), type: 'success' });
  resetPage();
});
</script>
