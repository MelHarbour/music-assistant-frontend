<template>
  <ItemsListing
    itemtype="tracks"
    :show-provider="false"
    :show-favorites-only-filter="true"
    :show-track-number="false"
    :load-paged-data="loadItems"
    :sort-keys="sortKeys"
    :show-album="true"
    :update-available="updateAvailable"
    :title="$t('tracks')"
    :show-search-button="true"
    :allow-key-hooks="true"
    :context-menu-items="[
      {
        label: 'add_url_item',
        labelArgs: [],
        action: () => {
          addUrl();
        },
        icon: 'mdi-link-plus',
      },
    ]"
    icon="mdi-music-note"
    :restore-state="true"
    :total="total"
  />
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import ItemsListing, { LoadDataParams } from '@/components/ItemsListing.vue';
import api from '@/plugins/api';
import {
  EventMessage,
  EventType,
  ImageType,
  MediaType,
} from '@/plugins/api/interfaces';
import { sleep } from '@/helpers/utils';
import router from '@/plugins/router';
import { store } from '@/plugins/store';

defineOptions({
  name: 'Tracks',
});

const { t } = useI18n();
const updateAvailable = ref<boolean>(false);
const total = ref(store.libraryTracksCount);

const sortKeys = [
  'name',
  'name_desc',
  'sort_name',
  'sort_name_desc',
  'duration',
  'duration_desc',
  'timestamp_added',
  'timestamp_added_desc',
  'last_played',
  'last_played_desc',
  'play_count',
  'play_count_desc',
];

onMounted(() => {
  // signal if/when items get added/updated/removed within this library
  const unsub = api.subscribe_multi(
    [
      EventType.MEDIA_ITEM_ADDED,
      EventType.MEDIA_ITEM_UPDATED,
      EventType.MEDIA_ITEM_DELETED,
    ],
    (evt: EventMessage) => {
      // signal user that there might be updated info available for this item
      if (evt.object_id?.startsWith('library://track')) {
        updateAvailable.value = true;
      }
    },
  );
  onBeforeUnmount(unsub);
});

const loadItems = async function (params: LoadDataParams) {
  params.favoritesOnly = params.favoritesOnly || undefined;
  if (params.refresh && !updateAvailable.value) {
    api.startSync([MediaType.TRACK]);
    // prevent race condition with a short sleep
    await sleep(250);
    // wait for sync to finish
    while (api.syncTasks.value.length > 0) {
      if (
        api.syncTasks.value.filter((x) =>
          x.media_types.includes(MediaType.TRACK),
        ).length == 0
      )
        break;
      await sleep(500);
    }
    await sleep(500);
  }
  updateAvailable.value = false;
  setTotals(params);
  return await api.getLibraryTracks(
    params.favoritesOnly,
    params.search,
    params.limit,
    params.offset,
    params.sortBy,
  );
};

const setTotals = async function (params: LoadDataParams) {
  if (!params.favoritesOnly) {
    total.value = store.libraryTracksCount;
    return;
  }
  total.value = await api.getLibraryTracksCount(params.favoritesOnly || false);
};

const addUrl = async function () {
  const url = prompt(t('enter_url'));
  if (!url) return;
  if (!url?.startsWith('http')) {
    alert(t('invalid_input'));
    return;
  }
  api
    .getTrack(url, 'builtin')
    .then((item) => {
      const name = prompt(t('enter_name'), item.name);
      item.name = name || item.name;
      delete item.sort_name;

      const imgUrl = prompt(
        t('image_url'),
        item.metadata.images?.length ? item.metadata.images[0].path : '',
      );
      if (imgUrl) {
        item.metadata.images = [
          {
            type: ImageType.THUMB,
            path: imgUrl,
            provider: 'builtin',
            remotely_accessible: true,
          },
        ];
      }
      api.addItemToLibrary(item).then(() => window.location.reload());
    })
    .catch((e) => alert(e));
};
</script>
