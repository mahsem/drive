<template>
  <Navbar
    v-if="!verify?.error && !getEntities.error"
    :root-actions="
      verify?.data &&
      actionItems
        .filter((k) => k.isEnabled?.(verify.data))
        // Remove irrelevant ones
        .slice(1)
        .toSpliced(4, 1)
        .map((k) => ({ ...k, onClick: () => k.action([verify.data]) }))
    "
    :trigger-root="
      () => ((selections = new Set()), store.commit('setActiveEntity', null))
    "
    :root-entity="verify?.data"
  />

  <FolderContentsError
    v-if="verify?.error || getEntities.error"
    :error="verify?.error || getEntities.error"
  />

  <div class="flex flex-col overflow-scroll min-h-full" ref="container" v-else>
    <DriveToolBar
      v-model="rows"
      :action-items="actionItems"
      :selections="selectedEntitities"
      :get-entities="getEntities"
    />
    <NoFilesSection
      v-if="!props.getEntities.data?.length"
      :icon="icon"
      :primary-message="primaryMessage"
      :secondary-message="secondaryMessage"
    />
    <ListView
      v-else-if="$store.state.view === 'list'"
      v-model="selections"
      :folder-contents="rows && grouper(rows)"
      :action-items="actionItems"
      :user-data="userData"
      @dropped="onDrop"
    />
    <GridView
      v-else
      v-model="selections"
      :folder-contents="rows"
      :action-items="actionItems"
      :user-data="userData"
      @dropped="onDrop"
    />
    <InfoPopup :entities="infoEntities" />
  </div>

  <Dialogs
    v-model="dialog"
    :selected-rows="activeEntity ? [activeEntity] : selectedEntitities"
    :root-entity="verify?.data"
    :get-entities="getEntities"
  />
  <FileUploader v-if="$store.state.user.id" @success="getEntities.fetch()" />
</template>
<script setup>
import ListView from "@/components/ListView.vue"
import GridView from "@/components/GridView.vue"
import DriveToolBar from "@/components/DriveToolBar.vue"
import Navbar from "@/components/Navbar.vue"
import NoFilesSection from "@/components/NoFilesSection.vue"
import Dialogs from "@/components/Dialogs.vue"
import FolderContentsError from "@/components/FolderContentsError.vue"
import InfoPopup from "@/components/InfoPopup.vue"
import { getLink } from "@/utils/getLink"
import { toggleFav, clearRecent } from "@/resources/files"
import { allUsers } from "@/resources/permissions"
import { entitiesDownload } from "@/utils/download"
import { RotateCcw } from "lucide-vue-next"
import FileUploader from "@/components/FileUploader.vue"
import Share from "./EspressoIcons/ShareNew.vue"
import Download from "./EspressoIcons/Download.vue"
import Link from "./EspressoIcons/Link.vue"
import Rename from "./EspressoIcons/Rename.vue"
import Move from "./EspressoIcons/Move.vue"
import Info from "./EspressoIcons/Info.vue"
import Preview from "./EspressoIcons/Preview.vue"
import Trash from "./EspressoIcons/Trash.vue"
import { ref, computed, watch } from "vue"
import { useRoute } from "vue-router"
import { useStore } from "vuex"
import { openEntity } from "@/utils/files"
import { toast } from "@/utils/toasts"
import { move, allFolders } from "@/resources/files"

const props = defineProps({
  grouper: { type: Function, default: (d) => d },
  showSort: { type: Boolean, default: true },
  verify: { Object, default: null },
  icon: Object,
  primaryMessage: String,
  secondaryMessage: { type: String, default: "" },
  getEntities: Object,
})
const route = useRoute()
const store = useStore()
import { useInfiniteScroll } from "@vueuse/core"

const dialog = ref(null)
const infoEntities = ref([])
const team = route.params.team
const activeEntity = computed(() => store.state.activeEntity)
const rows = ref(props.getEntities.data)
watch(
  () => props.getEntities.data,
  (val) => {
    rows.value = val
  }
)

const selections = ref(new Set())
const selectedEntitities = computed(
  () =>
    props.getEntities.data?.filter?.(({ name }) =>
      selections.value.has(name)
    ) || []
)

const verifyAccess = computed(() => props.verify?.data || !props.verify)

watch(
  verifyAccess,
  async (data) => {
    if (data)
      await props.getEntities.fetch({
        team,
        order_by:
          store.state.sortOrder.field +
          (store.state.sortOrder.ascending ? " 1" : " 0"),
      })
  },
  { immediate: true }
)

allUsers.fetch({ team })
allFolders.fetch({ team })

// Drag and drop

const onDrop = (targetFile, draggedItem) => {
  if (!targetFile.is_group || draggedItem === targetFile.name) return
  move.submit({
    entity_names: [draggedItem],
    new_parent: targetFile.name,
  })
  const removedIndex = props.getEntities.data.findIndex(
    (k) => k.name === draggedItem
  )
  props.getEntities.data.splice(removedIndex, 1)
  props.getEntities.data.find((k) => k.name === targetFile.name).children += 1
  props.getEntities.setData(data)
}

// Action Items
const actionItems = computed(() => {
  if (route.name === "Trash") {
    return [
      {
        label: "Restore",
        icon: RotateCcw,
        action: () => (dialog.value = "restore"),
        multi: true,
        important: true,
      },
      {
        label: "Delete forever",
        icon: Trash,
        action: () => (dialog.value = "d"),
        isEnabled: () => route.name === "Trash",
        multi: true,
        important: true,
      },
    ].filter((a) => !a.isEnabled || a.isEnabled())
  } else {
    return [
      {
        label: "Preview",
        icon: Preview,
        action: ([entity]) => openEntity(team, entity),
        isEnabled: (e) => !e.is_link,
      },
      {
        label: "Open",
        icon: "external-link",
        action: ([entity]) => openEntity(team, entity),
        isEnabled: (e) => e.is_link,
      },
      { label: "Divider" },
      {
        label: "Share",
        icon: Share,
        action: () => (dialog.value = "s"),
        isEnabled: (e) => e.share,
        important: true,
      },
      {
        label: "Download",
        icon: Download,
        isEnabled: (e) => !e.is_link,
        action: (entities) => entitiesDownload(team, entities),
        multi: true,
        important: true,
      },
      {
        label: "Copy Link",
        icon: Link,
        action: ([entity]) => getLink(entity),
        important: true,
      },
      { label: "Divider" },
      {
        label: "Move",
        icon: Move,
        action: () => (dialog.value = "m"),
        isEnabled: (e) => e.write,
        multi: true,
        important: true,
      },
      {
        label: "Rename",
        icon: Rename,
        action: () => (dialog.value = "rn"),
        isEnabled: (e) => e.write,
      },

      // {
      //   label: "Move to Team",
      //   icon: Team,
      //   action: (entities) =>
      //     confirm(
      //       `Are you sure you want to move ${entities.length} ${
      //         entities.length === 1 ? "item" : "items"
      //       } to the team?`
      //     ) &&
      //     entities.map((e) =>
      //       togglePersonal.submit({ entity_name: e.name, new_value: 0 })
      //     ),
      //   isEnabled: () => route.name == "Home",
      //   multi: true,
      //   important: true,
      // },
      {
        label: "Show Info",
        icon: Info,
        action: () => infoEntities.value.push(store.state.activeEntity),
        isEnabled: () => !store.state.activeEntity || !store.state.showInfo,
      },
      {
        label: "Hide Info",
        icon: Info,
        action: () => (dialog.value = "info"),
        isEnabled: () => store.state.activeEntity && store.state.showInfo,
      },
      {
        label: "Favourite",
        icon: "star",
        action: (entities) => {
          entities.forEach((e) => (e.is_favourite = true))
          // Hack to cache
          props.getEntities.setData(props.getEntities.data)
          toggleFav.submit({ entities })
        },
        isEnabled: (e) => !e.is_favourite,
        important: true,
        multi: true,
      },
      {
        label: "Unfavourite",
        icon: "star",
        class: "stroke-amber-500 fill-amber-500",
        action: (entities) => {
          entities.forEach((e) => (e.is_favourite = false))
          props.getEntities.setData(props.getEntities.data)
          toggleFav.submit({ entities })
        },
        isEnabled: (e) => e.is_favourite,
        important: true,
        multi: true,
      },
      {
        label: "Remove from Recents",
        icon: "clock",
        action: (entities) => {
          clearRecent.submit({
            entities,
          })
        },
        isEnabled: () => route.name == "Recents",
        important: true,
        multi: true,
      },
      {
        label: "Unshare",
        danger: true,
        icon: "trash-2",
        action: () => (dialog.value = "unshare"),
        isEnabled: (e) =>
          e.owner != "You" && e.user_doctype === "User" && e.everyone !== 1,
      },
      { label: "Divider" },
      {
        label: "Move to Trash",
        danger: true,

        icon: Trash,
        action: () => (dialog.value = "remove"),
        isEnabled: (e) => e.write,
        important: true,
        multi: true,
        danger: true,
      },
    ]
  }
})

const userData = computed(() =>
  allUsers.data ? Object.fromEntries(allUsers.data.map((k) => [k.name, k])) : {}
)

async function newLink() {
  if (!document.hasFocus()) return
  try {
    const text = await navigator.clipboard.readText()
    if (localStorage.getItem("prevClip") === text) return
    new URL(text)
    localStorage.setItem("prevClip", text)
    toast({
      title: "Link detected",
      text,
      buttons: [
        {
          label: "Add",
          action: () => {
            dialog.value = "l"
          },
        },
      ],
    })
  } catch (_) {}
}

// Hacky but performant way to track links - when the user loads page, copies on page, or comes to page
// JS doesn't allow direct reading of clipboard
newLink()
addEventListener("copy", () => document.getElementById("popovers").click())
document.getElementById("popovers").addEventListener("click", newLink)
document.addEventListener("visibilitychange", () => {
  window.focus()
  !document.hidden && setTimeout(newLink, 100)
})
</script>
