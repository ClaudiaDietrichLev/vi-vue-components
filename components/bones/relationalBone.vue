<template>
  <div class="record">
    <div class="single-entry">
      <sl-input
        v-if="state.selection"
        :disabled="true"
        :value="value ? formatString(state.format, state.selection) : ''"
      ></sl-input>
      <sl-combobox
        v-else
        :disabled="boneState.readonly"
        :source="getList"
        hoist
        @sl-item-select="changeEvent"
      >
      </sl-combobox>

      <sl-button @click="openRelationalSelection">
        <sl-icon name="list-ul"></sl-icon>
      </sl-button>

      <sl-button
        v-if="value"
        variant="info"
        outline
        @click="editSelection"
      >
        <sl-icon name="pencil"></sl-icon>
      </sl-button>

      <sl-button
        v-if="!boneState.multiple && !boneState.isEmpty"
        variant="danger"
        outline
        :title="$t('bone.del')"
        class="delete-btn"
        @click="
          () => {
            $emit('change', name, '', lang, index)
            state.selection = null
          }
        "
      >
        <sl-icon name="x"></sl-icon>
      </sl-button>
    </div>

    <Wrapper_nested
      v-if="boneState?.bonestructure['using']"
      :value="value?.['rel']"
      :name="name"
      :index="index"
      :disabled="boneState.bonestructure['readonly']"
      @change="changeEventNested"
    >
    </Wrapper_nested>
  </div>

  <relational-selector
    :open="state.openedSelection"
    :name="boneState.bonestructure['descr']"
    :tab-id="handlerState.tabId"
    :handler="state.moduleInfo['handlerComponent']"
    :module="boneState?.bonestructure['module']"
    @close="relationCloseAction"
  >
  </relational-selector>
</template>

<script lang="ts">
//@ts-nocheck
import { reactive, defineComponent, onMounted, inject, computed, unref, watch } from "vue"
import { Request } from "@viur/vue-utils"
import Wrapper_nested from "@viur/vue-utils/bones/edit/wrapper_nested.vue"
import { useRoute, useRouter } from "vue-router"
import { useDBStore } from "../stores/db"
import handlers from "../handler/handlers"
import relationalSelector from "./components/relationalSelector.vue"

export default defineComponent({
  props: {
    name: String,
    value: Object,
    index: Number,
    lang: String
  },
  components: { Wrapper_nested, ...handlers, relationalSelector },
  emits: ["change"],
  setup(props, context) {
    const boneState = inject("boneState")
    const handlerState = inject("handlerState")
    const formatString = inject("formatString")
    const route = useRoute()
    const router = useRouter()
    const dbStore = useDBStore()
    const state = reactive({
      format: computed(() => {
        return boneState?.bonestructure["format"]
      }),
      openedSelection: false,
      moduleInfo: computed(() => dbStore.getConf(boneState?.bonestructure["module"])),
      selection: null,
      entry: computed(() => {
        if (typeof props.value !== "object") {
          return { dest: props.value, rel: null }
        }
        return props.value
      }),
      skellistdata: null
    })

    function getList(search: String) {
      let params = ""
      if (boneState.bonestructure["type"] === "relational.tree.leaf.file") {
        params = "skelType=leaf&"
      } else if (boneState.bonestructure["type"] === "relational.tree.node.file") {
        params = "skelType=node&"
      }

      return Request.get(`/json/${boneState.bonestructure["module"]}/list?${params}limit=99`).then(async (resp) => {
        //?viurTags$lk=${search.toLowerCase()
        const data = await resp.json()
        state.skellistdata = {}
        for (let e of data["skellist"]) {
          state.skellistdata[e["key"]] = e
        }

        return data["skellist"]?.map((d: any) => {
          return { text: formatString(boneState.bonestructure["format"], { dest: d }), value: d.key, data: d }
        })
      })
    }

    function changeEventNested(val) {
      if (!state.selection) state.selection = []
      if (Object.keys(state.selection).includes("rel")) {
        state.selection["rel"][val.name] = val.value
      } else {
        state.selection["rel"] = { [val.name]: val.value }
      }

      context.emit("change", props.name, state.selection, props.lang, props.index)
    }

    function changeEvent(event) {
      state.selection = { dest: state.skellistdata[event.detail.item.value] }
      context.emit("change", props.name, state.selection, props.lang, props.index)
    }

    onMounted(() => {
      state.selection = props.value
      context.emit("change", props.name, props.value, props.lang, props.index) //init
    })

    function openRelationalSelection() {
      state.openedSelection = true
    }

    function relationCloseAction(selection) {
      state.openedSelection = false
      if (selection) {
        if (state.selection) {
          state.selection["dest"] = selection
        } else {
          state.selection = { dest: selection }
        }
        context.emit("change", props.name, state.selection, props.lang, props.index)
      }
    }

    function editSelection() {
      const mod = boneState.bonestructure["module"]
      const url = `/db/${mod}/edit/${props.value["dest"]["key"]}`
      let route = router.resolve(unref(url))
      dbStore.addOpened(route, mod)
    }

    watch(
      () => props.value,
      (newVal, oldVal) => {
        state.selection = newVal
      }
    )

    return {
      state,
      boneState,
      handlerState,
      dbStore,
      formatString,
      changeEvent,
      changeEventNested,
      getList,
      openRelationalSelection,
      relationCloseAction,
      editSelection
    }
  }
})
</script>

<style scoped>
.single-entry {
  display: flex;
  gap: 10px;
}

sl-input {
  width: 100%;

  &::part(base) {
    border-top-left-radius: 0;
    border-bottom-left-radius: 0;
  }

  &::part(base) {
    background-color: var(--sl-color-neutral-0);
  }
}

sl-combobox {
  width: 100%;

  &::part(input) {
    border-top-left-radius: 0;
    border-bottom-left-radius: 0;

    & :deep(&::part(base)) {
      border: 1px solid red;
    }
  }
}
</style>