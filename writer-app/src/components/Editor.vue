<script setup lang="ts">
import { useEditor, EditorContent } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import Placeholder from '@tiptap/extension-placeholder'
import { watch, onBeforeUnmount, ref } from 'vue'

const props = defineProps<{ modelValue: string, typography: any }>()
const emit = defineEmits(['update:modelValue'])

const canUndo = ref(false)
const canRedo = ref(false)

const editor = useEditor({
  content: props.modelValue,
  extensions: [StarterKit, Placeholder.configure({ placeholder: '开始构思你的伟大作品...' })],
  editorProps: { attributes: { class: 'focus:outline-none min-h-full' } },
  onUpdate: ({ editor }) => emit('update:modelValue', editor.getHTML()),
  onTransaction: ({ editor }) => {
    canUndo.value = editor.can().undo()
    canRedo.value = editor.can().redo()
    
    // 🔥 核心：光标安全距离（打字机滚动）算法 🔥
    setTimeout(() => {
      const scrollBox = document.getElementById('editor-scroll-box') // 获取外部滚动容器
      if (!scrollBox) return
      const { state, view } = editor
      if (!state.selection.empty) return // 如果是划选文字，不干预滚动

      try {
        // 获取当前光标在屏幕上的绝对坐标
        const coords = view.coordsAtPos(state.selection.head)
        // 计算安全红线（屏幕高度 - 底部安全距离）
        const safeBottom = window.innerHeight - props.typography.paddingBottom
        
        // 如果光标掉到了安全红线以下，强制把滚动条往上推！
        if (coords.bottom > safeBottom) {
          scrollBox.scrollBy({ top: coords.bottom - safeBottom + 20, behavior: 'smooth' })
        }
      } catch (e) {}
    }, 10) // 延迟10ms等待DOM渲染
  }
})

watch(() => props.modelValue, (newValue) => {
  const isSame = editor.value?.getHTML() === newValue
  if (!isSame && editor.value) {
    editor.value.commands.setContent(newValue, { emitUpdate: false })
  }
})

onBeforeUnmount(() => editor.value?.destroy())

const undo = () => editor.value?.chain().focus().undo().run()
const redo = () => editor.value?.chain().focus().redo().run()
const syncContentWithHistory = (html: string) => {
  if (editor.value) editor.value.chain().focus().selectAll().insertContent(html).run()
}

defineExpose({ undo, redo, canUndo, canRedo, syncContentWithHistory })
</script>

<template>
  <div class="novel-editor w-full h-full text-gray-800" :style="{
    '--font-size': props.typography.fontSize + 'px',
    '--line-height': props.typography.lineHeight,
    '--letter-spacing': props.typography.letterSpacing + 'px',
    '--paragraph-spacing': props.typography.emptyLine ? '2em' : (props.typography.paragraphSpacing + 'px'),
    '--padding-x': props.typography.paddingX + 'px',
    '--padding-bottom': props.typography.paddingBottom + 'px',
    '--text-indent': props.typography.textIndent + 'em',
  }">
    <editor-content :editor="editor" />
  </div>
</template>

<style>
.novel-editor .tiptap {
  padding-left: var(--padding-x); padding-right: var(--padding-x);
  /* 这里的 paddingBottom 作为保底空白，真正的防遮挡由上面的 JS 算法接管 */
  padding-bottom: var(--padding-bottom); 
}
.novel-editor .tiptap p {
  text-indent: var(--text-indent); line-height: var(--line-height); margin-bottom: var(--paragraph-spacing);
  font-size: var(--font-size); letter-spacing: var(--letter-spacing); outline: none;
}
.novel-editor .tiptap p.is-editor-empty:first-child::before { 
  color: #9ca3af; content: attr(data-placeholder); float: left; height: 0; pointer-events: none; 
}
</style>