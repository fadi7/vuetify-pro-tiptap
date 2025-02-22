<template>
  <div v-if="editor" class="vuetify-pro-tiptap" :class="{ dense }">
    <!-- Edit Mode -->
   <!-- <BubbleMenu :editor="editor" :dark="isDark" :disabled="disableToolbar" :items="items" />  -->

    <v-subheader v-if="label">{{ label }}</v-subheader>

    <v-input class="pt-0" v-bind="$attrs" hide-details="auto">
      <v-card
        flat
        :outlined="outlined"
        :dark="isDark"
        :color="isDark ? '#212121' : '#EEEEEE'"
        style="width: 100%"
        v-bind="$attrs"
        :style="{
          borderColor: $attrs['error-messages'] ? '#ff5252' : undefined
        }"
        class="vuetify-pro-tiptap-editor"
        :class="{ 'vuetify-pro-tiptap-editor--fullscreen': isFullscreen }"
      >
        <!-- Toolbar -->
        <TipTapToolbar
          ref="toolbarRef"
          class="vuetify-pro-tiptap-editor__toolbar"
          v-if="!hideToolbar && toolbar && toolbar.length"
          :editor="editor"
          :dark="isDark"
          :disabled="disableToolbar"
          :items="items"
          :color="isDark ? undefined : 'grey lighten-4'"
        >
          <template v-for="item in slotItems" #[item.slot]="{ attrs }">
            <slot :name="item.slot" v-bind="{ editor, attrs }"></slot>
          </template>
        </TipTapToolbar>

        <slot name="editor" v-bind="{ editor, attrs: { class: 'vuetify-pro-tiptap-editor__content', 'data-testid': 'value' } }">
          <editor-content
            class="vuetify-pro-tiptap-editor__content markdown-body"
            :class="[{ __dark: isDark }, editorClass]"
            :editor="editor"
            :style="contentDynamicStyles"
            data-testid="value"
          />
        </slot>

        <slot name="bottom" v-bind="{ editor }">
          <v-toolbar dense flat :color="isDark ? undefined : 'grey lighten-4'">
            <v-spacer />

            <span class="text-overline me-4">{{ editor.storage.characterCount.words() }} {{ t('editor.words') }}</span>
            <span class="text-overline">{{ editor.storage.characterCount.characters() }} {{ t('editor.characters') }}</span>
          </v-toolbar>
        </slot>
      </v-card>
    </v-input>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, unref, computed, watch, onUnmounted } from 'vue-demi'
import type { Ref } from 'vue-demi'
import { string, bool, array, object, oneOfType } from 'vue-types'
import throttle from 'lodash.throttle'
import merge from 'lodash.merge'

import { Editor, EditorContent } from '@tiptap/vue-2'
import type { AnyExtension, EditorOptions } from '@tiptap/vue-2'
import TipTapToolbar from './TipTapToolbar.vue'
//import BubbleMenu from './BubbleMenu/index.vue'
import TiptapKit from '@/core/tiptap-kit'
import type { StarterKitOptions } from '@/core/tiptap-kit'

import { useLocale } from '@/locales'
import toolbarItems from '@/constants/toolbar-items'
import { useMakeToolbarDefinitions } from '@/constants/toolbar-definitions'
import type { ToolbarType, Definitions } from '@/constants/toolbar-definitions'

import useContext from '@/hooks/use-context'

type HandleKeyDown = NonNullable<EditorOptions['editorProps']['handleKeyDown']>
type OnSelectionUpdate = NonNullable<EditorOptions['onSelectionUpdate']>
type OnUpdate = NonNullable<EditorOptions['onUpdate']>

const THROTTLE_WAIT_TIME = 16

const getUnitWithPxAsDefault = (value?: string) => {
  if (!value) return value

  const num = parseInt(value, 10)
  const unit = value.slice(num.toString().length)

  return num + (unit || 'px')
}

const isNumber = (value: unknown): value is number => typeof value === 'number'

export default defineComponent({
  components: {
    TipTapToolbar,
    EditorContent,
 //   BubbleMenu
  },
  props: {
    value: string().def(''),
    dark: bool().def(false),
    dense: bool().def(false),
    outlined: bool().def(true),
    disabled: bool().def(false),
    label: string(),
    placeholder: string(),
    toolbar: array<ToolbarType>().def(toolbarItems),
    // append: array<string>().def([]),
    hideToolbar: bool().def(false),
    disableToolbar: bool().def(false),
    maxWidth: oneOfType<string | number>([String, Number]),
    minHeight: oneOfType<string | number>([String, Number]),
    maxHeight: oneOfType<string | number>([String, Number]),

    // Editor
    extensions: array<AnyExtension>().def([]),
    config: object<Partial<StarterKitOptions>>().def({}),
    editorClass: oneOfType<string | string[] | Record<string, any>>([String, Array, Object])
  },
  setup(props, { emit }) {
    const toolbarRef = ref<Record<string, any> | null>(null)
    const editor: Ref<Editor | null> = ref(null)
    const isFullscreen = ref<boolean>(false)

    const root = useContext()
    const { t } = useLocale()

    const { items } = useMakeToolbarDefinitions({
      editor,
      isFullscreen,
      toolbar: props.toolbar
    })

    const isDark = computed<boolean>(() => props.dark || root?.$vuetify.theme.dark || false)

    const slotItems = computed<(Definitions & { slot: string })[]>(() => {
      return unref(items).filter(item => item.type === 'slot') as (Definitions & { slot: string })[]
    })

    const contentDynamicStyles = computed(() => {
      const maxWidth = getUnitWithPxAsDefault(isNumber(props.maxWidth) ? String(props.maxWidth) : props.maxWidth)

      const maxHeightStyle = {
        maxWidth: maxWidth,
        width: !maxWidth ? undefined : '100%',
        margin: !maxWidth ? undefined : '0 auto',
        backgroundColor: unref(isDark) ? '#1E1E1E' : '#FFFFFF'
      }
      if (unref(isFullscreen)) return { height: '100%', overflowY: 'auto', ...maxHeightStyle }

      const minHeight = getUnitWithPxAsDefault(isNumber(props.minHeight) ? String(props.minHeight) : props.minHeight)
      const maxHeight = getUnitWithPxAsDefault(isNumber(props.maxHeight) ? String(props.maxHeight) : props.maxHeight)

      return {
        minHeight,
        maxHeight,
        overflowY: 'auto',
        ...maxHeightStyle
      }
    })

    const onValueChange = throttle((val: string) => {
      if (!unref(editor)) return

      const html = unref(editor)?.getHTML()
      if (html === val) return

      unref(editor)?.commands.setContent(val)
    }, THROTTLE_WAIT_TIME)

    const onDisabledChange = (val: boolean) => unref(editor)?.setEditable(!val)

    watch(() => props.value, onValueChange)
    watch(() => props.disabled, onDisabledChange)

    const handleKeyDown = throttle<HandleKeyDown>(function (view, event) {
      if (event.key === 'Enter' && root?.$listeners.enter && !event.shiftKey) {
        emit('enter')

        return true
      }

      return false
    }, THROTTLE_WAIT_TIME)

    const onSelectionUpdate = throttle<OnSelectionUpdate>(function ({ editor }) {
      unref(toolbarRef)?.onUpdate(editor)
    }, THROTTLE_WAIT_TIME)

    const onUpdate = throttle<OnUpdate>(function ({ editor }) {
      emit('input', editor.getHTML())
    }, THROTTLE_WAIT_TIME)

    const onCreate = () => {
      const TiptapKitConfig: Partial<StarterKitOptions> = merge(
        {
          bold: {},
          blockquote: {
            HTMLAttributes: {
              class: 'blockquote'
            }
          },
          history: {
            depth: 10
          },
          heading: {
            levels: [1, 2, 3, 4, 5, 6]
          },
          placeholder: {
            placeholder: () => props.placeholder || ''
          },
          textAlign: {
            types: ['heading', 'paragraph', 'image']
          },
          focus: {
            className: 'focus'
          },
          color: {},
          highlight: { multicolor: true },
          image: {
            inline: true
          },
          link: {
            openOnClick: true
          },
          table: {
            HTMLAttributes: {
              class: 'table-wrapper'
            }
          },
          taskList: {
            HTMLAttributes: {
              class: 'task-list'
            }
          },
          taskItem: {
            HTMLAttributes: {
              class: 'task-list-item'
            }
          },
          textStyle: {},
          underline: {},
          video: {}
        } as Partial<StarterKitOptions>,
        root?.$vuetifyProTiptap?.config,
        props.config
      )

      editor.value = new Editor({
        content: props.value,
        editorProps: {
          handleKeyDown: handleKeyDown
        },
        onSelectionUpdate: onSelectionUpdate,
        onUpdate: onUpdate,
        extensions: [TiptapKit.configure(TiptapKitConfig), ...props.extensions],
        autofocus: false,
        editable: true,
        injectCSS: true
      })
    }

    onCreate()

    onUnmounted(() => unref(editor)?.destroy())

    return {
      toolbarRef,
      editor,
      isFullscreen,
      items,
      isDark,
      slotItems,
      contentDynamicStyles,
      t
    }
  }
})
</script>

<style lang="scss">
@import '@/styles/editor.scss';
</style>
