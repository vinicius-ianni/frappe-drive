<template>
  <FrappeListView
    row-key="name"
    :columns="selectedColumns"
    :rows="formattedRows"
    :options="{
      selectable: true,
      showTooltip: true,
      resizeColumn: true,
    }"
  >
    <ListHeader />
    <template v-if="formattedRows.length">
      <div v-for="group in formattedRows" :key="group.group">
        <ListGroupHeader :group="group">
          <slot
            name="group-header"
            v-if="$slots['group-header']"
            v-bind="{ group }"
          />
        </ListGroupHeader>
        <ListGroupRows :group="group">
          <ListRow
            v-for="row in group.rows"
            :key="row.name"
            :row="row"
            @dblclick="openFile(row)"
          >
            <template #="{ column, item }">
              <ListRowItem
                :column="column"
                :row="row"
                :item="item"
                :align="column.align"
              >
                <template v-if="column.key === 'options'">
                  <Dropdown :options="dropdownActionItems(row)">
                    <Button class="bg-white">
                      <FeatherIcon name="more-horizontal" class="h-4 w-4" />
                    </Button>
                  </Dropdown>
                </template>
              </ListRowItem>
            </template>
          </ListRow>
        </ListGroupRows>
      </div>
    </template>
    <Loader v-else />
    <ListSelectBanner>
      <template #actions="{ selections }">
        <div class="flex gap-3">
          <Button
            v-for="(item, index) in actionItems
              .filter(
                (i) =>
                  !i.isEnabled ||
                  [...selections]
                    .map((n) => entities.find((e) => e.name === n))
                    .every((e) => i.isEnabled(e, [...selections].length !== 1))
              )
              .filter(
                (i) => i.important && ([...selections].length === 1 || i.multi)
              )"
            :key="index"
            @click="
              () =>
                handleAction(
                  [...selections].map((n) =>
                    entities.find((e) => e.name === n)
                  ),
                  item
                )
            "
          >
            <FeatherIcon
              v-if="typeof item.icon === 'string'"
              :name="item.icon"
              class="h-4 w-4"
              :class="
                item.label === 'Unfavourite'
                  ? 'stroke-yellow-500 fill-yellow-500'
                  : ''
              "
            />
            <component
              :is="item.icon"
              v-else
              class="h-4 w-auto text-gray-800"
              :class="item.danger ? 'text-red-500' : ''"
            />
          </Button>
        </div>
      </template>
    </ListSelectBanner>
  </FrappeListView>
</template>
<script setup>
import {
  Avatar,
  Button,
  FeatherIcon,
  ListSelectBanner,
  ListHeader,
  ListRowItem,
  ListGroupRows,
  ListRow,
  ListGroupHeader,
  ListView as FrappeListView,
  Dropdown,
} from "frappe-ui"
import Loader from "@/components/Loader.vue"
import { formatMimeType } from "@/utils/format"
import { getIconUrl } from "@/utils/getIconUrl"
import { useInfiniteScroll } from "@vueuse/core"
import { useStore } from "vuex"
import { useRoute } from "vue-router"
import { ref, computed, h } from "vue"
import Folder from "./MimeIcons/Folder.vue"
import { openEntity } from "@/utils/files"

const store = useStore()
const route = useRoute()

const props = defineProps({
  folderContents: {
    type: Object,
    default: null,
  },
  overrideCanLoadMore: {
    type: Boolean,
    default: false,
  },
  actionItems: {
    type: Array,
  },
})

const multi = ref(false)
const container = ref(null)

const entities = computed(() => store.state.currentViewEntites)
const formattedRows = computed(() => {
  let groups = Object.keys(props.folderContents)
    .map((k) => ({
      group: k,
      rows: props.folderContents[k] || [],
      collapsed: false,
    }))
    .filter((g) => g.rows.length)
  groups = groups.map((g) => ({
    ...g,
    rows: g.rows.map((r) => ({
      ...r,
      owner: { label: r.owner, image: r.user_image },
    })),
  }))
  return groups
})

const selectedColumns = [
  {
    label: "Name",
    key: "title",
    getLabel: ({ row }) =>
      row.file_ext
        ? row.title.slice(0, row.title.length - row.file_ext.length)
        : row.title,
    prefix: ({ row }) =>
      row.is_group
        ? h(Folder)
        : h("img", { src: getIconUrl(formatMimeType(row.mime_type)) }),
    width: 2.5,
  },
  {
    label: "Owner",
    key: "owner",
    prefix: ({ row }) =>
      h(Avatar, {
        shape: "circle",
        image: row.user_image,
        label: row.full_name,
        size: "sm",
        tooltip: "Hey",
      }),
  },
  {
    label: route.name === "Recents" ? "Last Accessed" : "Last Modified",
    getLabel: ({ row }) => row.relativeModified,
    key: "modified",
  },
  {
    label: "Size",
    key: "file_size",
    getLabel: ({ row }) => row.file_size || "-",
  },
  { label: "", key: "options", align: "right", width: "10px" },
].filter((k) => (k.enabled ? k.enabled(route.name) : true))

const emit = defineEmits([
  "showEntityContext",
  "showEmptyEntityContext",
  "fetchFolderContents",
  "updateOffset",
  "update-multi",
])
useInfiniteScroll(container, () => emit("updateOffset"), {
  direction: "bottom",
  distance: 150,
  interval: 2000,
  canLoadMore: () => props.overrideCanLoadMore,
})

function openFile(entity) {
  store.commit("setEntityInfo", [entity])
  openEntity(entity)
}

function handleAction(selectedItems, action) {
  store.commit("setEntityInfo", selectedItems)
  action.onClick(selectedItems)
}

function dropdownActionItems(row) {
  return props.actionItems
    .filter((a) => !a.isEnabled || a.isEnabled(row))
    .map((a) => ({
      ...a,
      onClick: () => {
        store.commit("setEntityInfo", [row])
        a.onClick([row])
      },
    }))
}
</script>
