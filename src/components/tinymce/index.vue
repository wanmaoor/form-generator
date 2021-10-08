<template>
  <textarea :id="tinymceId" style="visibility: hidden" />
</template>

<script>
import loadTinymce from '@/utils/loadTinymce'
import { plugins, toolbar } from './config'
import { debounce } from 'throttle-debounce'

let num = 1

export default {
  props: {
    id: {
      type: String,
      default: () => {
        num === 10000 && (num = 1)
        return `tinymce${+new Date()}${num++}`
      }
    },
    value: {
      default: ''
    }
  },
  data() {
    return {
      tinymceId: this.id
    }
  },
  mounted() {
    loadTinymce(tinymce => {
      // eslint-disable-next-line global-require
      require('./zh_CN')
      let conf = {
        selector: `#${this.tinymceId}`,
        language: 'zh_CN',
        menubar: 'file edit insert view format table',
        plugins,
        toolbar,
        height: 300,
        branding: false,
        object_resizing: false,
        end_container_on_empty_block: true,
        powerpaste_word_import: 'clean',
        code_dialog_height: 450,
        code_dialog_width: 1000,
        advlist_bullet_styles: 'square',
        advlist_number_styles: 'default',
        default_link_target: '_blank',
        link_title: false,
        nonbreaking_force_tab: true,
        automatic_uploads: true,
        file_picker_types: 'image',
        file_picker_callback(cb, value, meta) {
          const input = document.createElement('input')
          input.setAttribute('type', 'file')
          input.setAttribute('accept', 'image/*')

          /*
            Note: In modern browsers input[type="file"] is functional without
            even adding it to the DOM, but that might not be the case in some older
            or quirky browsers like IE, so you might want to add it to the DOM
            just in case, and visually hide it. And do not forget do remove it
            once you do not need it anymore.
          */

          input.onchange = function () {
            const file = this.files[0]

            const reader = new FileReader()
            reader.onload = function () {
              /*
                Note: Now we need to register the blob in TinyMCEs image blob
                registry. In the next release this part hopefully won't be
                necessary, as we are looking to handle it internally.
              */
              const id = `blobid${(new Date()).getTime()}`
              const { blobCache } = tinymce.activeEditor.editorUpload
              const base64 = reader.result.split(',')[1]
              const blobInfo = blobCache.create(id, file, base64)
              blobCache.add(blobInfo)

              /* call the callback and populate the Title field with the file name */
              cb(blobInfo.blobUri(), { title: file.name })
            }
            reader.readAsDataURL(file)
          }

          input.click()
        }
      }
      conf = Object.assign(conf, this.$attrs)
      conf.init_instance_callback = editor => {
        if (this.value) editor.setContent(this.value)
        this.vModel(editor)
      }
      tinymce.init(conf)
    })
  },
  destroyed() {
    this.destroyTinymce()
  },
  methods: {
    vModel(editor) {
      // 控制连续写入时setContent的触发频率
      const debounceSetContent = debounce(250, editor.setContent)
      this.$watch('value', (val, prevVal) => {
        if (editor && val !== prevVal && val !== editor.getContent()) {
          if (typeof val !== 'string') val = val.toString()
          debounceSetContent.call(editor, val)
        }
      })

      editor.on('change keyup undo redo', () => {
        this.$emit('input', editor.getContent())
      })
    },
    destroyTinymce() {
      if (!window.tinymce) return
      const tinymce = window.tinymce.get(this.tinymceId)
      if (tinymce) {
        tinymce.destroy()
      }
    }
  }
}
</script>
