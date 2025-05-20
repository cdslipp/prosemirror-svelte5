<script lang="ts">
  import { EditorView } from "prosemirror-view"
  import { EditorState } from "prosemirror-state"
  import { createSingleLineEditor } from './state'

  type ProsemirrorEditorProps = Partial<{
    className: string
    editorState: EditorState
    placeholder: string
    view: EditorView | null
    debounceChangeEventsInterval: number
    editorViewProps: Record<string, any>
    onChange: (detail: { editorState: EditorState }) => void
    onTransaction: (detail: { 
      view: EditorView, 
      editorState: EditorState, 
      isDirty: boolean, 
      contentHasChanged: boolean 
    }) => void
    onCustom: (detail: { type?: string, [key: string]: any }) => void
    focusEditor: (() => void) | undefined
  }>

  let { 
    className = "ui-editor",
    editorState: incomingEditorState = createSingleLineEditor(),
    placeholder = '',
    view: externalView = null,
    debounceChangeEventsInterval = 50,
    editorViewProps = {},
    onChange,
    onTransaction,
    onCustom,
    focusEditor = $bindable<(() => void) | undefined>(undefined)
  } = $props()

  // State management with runes
  let editorState = $state(incomingEditorState)
  let view = $state<EditorView | null>(externalView)
  let isDirty = $state(false)
  let dispatchLastEditTimeout = $state<ReturnType<typeof setTimeout> | null>(null)
  let editor = $state<HTMLDivElement | null>(null)

  // Derived state
  let editorIsEmpty = $derived(
    editorState 
      ? editorState.doc.content.size === 0 || 
        (editorState.doc.textContent === "" && editorState.doc.content.size < 3)
      : true
  )

  // Methods
  function focus() {
    view?.focus()
  }

  // Update the bindable focusEditor prop
  $effect(() => {
    focusEditor = focus
  })

  export function blur() {
    editor?.blur()
  }

  // Event handlers
  function dispatchChangeEvent() {
    if (isDirty) {
      onChange?.({ editorState })
      isDirty = false
    }
  }

  // Set up custom event listener
  $effect(() => {
    if (!editor || !onCustom) return

    const handleCustomEvent = (event: CustomEvent) => {
      if (event.detail) {
        const { type, ...detail } = event.detail
        onCustom({ type, ...detail })
      }
    }

    editor.addEventListener('custom', handleCustomEvent as EventListener)
    return () => editor.removeEventListener('custom', handleCustomEvent as EventListener)
  })

  // Editor initialization and cleanup
  $effect(() => {
    if (!editor) return

    view = new EditorView({ mount: editor }, {
      ...editorViewProps,
      state: editorState,
      dispatchTransaction: (transaction) => {
        if (!view) return

        const newState = view.state.apply(transaction)
        const contentHasChanged = !newState.doc.eq(view.state.doc)

        editorState = newState

        if (contentHasChanged) {
          isDirty = true
          if (debounceChangeEventsInterval > 0) {
            if (dispatchLastEditTimeout) clearTimeout(dispatchLastEditTimeout)
            dispatchLastEditTimeout = setTimeout(dispatchChangeEvent, debounceChangeEventsInterval)
          } else {
            setTimeout(dispatchChangeEvent, 0)
          }
        }

        view.updateState(editorState)

        onTransaction?.({ view, editorState, isDirty, contentHasChanged })
      }
    })

    return () => {
      view?.destroy()
    }
  })

  // Sync external editor state changes
  $effect(() => {
    if (view && editorState && !isDirty) {
      view.updateState(editorState)
    }
  })
</script>

<div 
  class={className}
  class:ProseMirror={true}
  class:editor_empty={editorIsEmpty}
  data-placeholder={placeholder}
  bind:this={editor}
  on:focus={() => {}}
  on:blur={() => {}}
  on:keydown={() => {}}
></div>

<style>

  :global(body) {
    --ui-color-placeholder: #AAAAAA;
  }

  :global(.ProseMirror) {
    position: relative;
  }

  :global(.ProseMirror) {
    word-wrap: break-word;
    white-space: pre-wrap;
    -webkit-font-variant-ligatures: none;
    font-variant-ligatures: none;
  }

  :global(.ProseMirror) pre {
    white-space: pre-wrap;
  }

  :global(.ProseMirror) li {
    position: relative;
  }

  :global(.ProseMirror-hideselection *::selection) {
    background: transparent;
  }

  :global(.ProseMirror-hideselection *::-moz-selection) {
    background: transparent;
  }

  :global(.ProseMirror-hideselection) {
    caret-color: transparent;
  }

  :global(.ProseMirror-selectednode) {
    outline: 2px solid #8cf;
  }

  /* Make sure li selections wrap around markers */

  :global(li.ProseMirror-selectednode) {
    outline: none;
  }

  :global(li.ProseMirror-selectednode:after) {
    content: "";
    position: absolute;
    left: -32px;
    right: -2px;
    top: -2px;
    bottom: -2px;
    border: 2px solid #8cf;
    pointer-events: none;
  }

  :global(.ProseMirror .empty-node::before) {
    position: absolute;
    color: #aaa;
    cursor: text;
  }

  :global(.ProseMirror .empty-node:hover::before) {
    color: #777;
  }

  :global(.ProseMirror.editor_empty::before) {
    position: absolute;
    content: attr(data-placeholder);
    pointer-events: none;
    color: var(--ui-color-placeholder);
  }

</style>
