<template>
  <div
    class="b3log-editor"
    :class="{'b3log-editor--fullscreen': isFullScreen}"
    :style="`height: ${height || 'auto'}px`">
    <div class="b3log-editor__toolbar">
      <span
        :aria-label="convertHotKey(label.emoji) || 'Emoji'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="showEmojiPanel = true"><v-icon>emoji</v-icon></span>
      <span
        :aria-label="convertHotKey(label.bold) || 'Bold'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="insert('**', '**')"><v-icon>bold</v-icon></span>
      <span
        :aria-label="convertHotKey(label.italic) || 'Italic'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="insert('*', '*')"><v-icon>italic</v-icon></span>
      <span
        :aria-label="convertHotKey(label.quote) || 'Quote'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="insert('> ', '')"><v-icon>quote</v-icon></span>
      <span
        :aria-label="convertHotKey(label.link) || 'Link'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="insert('[', '](http://)')"><v-icon>link</v-icon></span>
      <span
        :aria-label="convertHotKey(label.upload) || 'Upload'"
        class="pipe-tooltipped pipe-tooltipped--e">
        <label>
          <v-icon>upload</v-icon>
          <input multiple="multiple" @change="selectFile" type="file"/>
        </label>
      </span>
      <span
        :aria-label="convertHotKey(label.unorderedList) || 'Unordered List'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="insert('* ', '')"><v-icon>unordered-list</v-icon></span>
      <span
        :aria-label="convertHotKey(label.orderedList) || 'Ordered List'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="insert('1. ', '')"><v-icon>ordered-list</v-icon></span>
      <span
        :aria-label="convertHotKey(label.preview) || 'Preview'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="togglePreview"
        :class="{'b3log-editor__icon--current' : hasPreview}"><v-icon>view</v-icon></span>
      <span
        :aria-label="convertHotKey(label.fullscreen) || 'Fullscreen'"
        class="pipe-tooltipped pipe-tooltipped--e"
        @click="isFullScreen = !isFullScreen"><v-icon>{{ isFullScreen ? 'contract' : 'fullscreen' }}</v-icon></span>
      <a
        :aria-label="convertHotKey(label.help) || 'Help'"
        class="pipe-tooltipped pipe-tooltipped--e"
        target="_blank"
        href="https://hacpai.com/guide/markdown">
        <v-icon>question</v-icon>
      </a>
      <div class="b3log-editor__emoji" v-show="showEmojiPanel">
        <div>
          <span v-for="item in emoji" @click="insert(`${item} `, '', true);textareaValue = $refs.b3logEditor.value">{{item}}</span>
        </div>
        <div class="b3log-editor__emoji-tip">
          <a href="https://www.webpagefx.com/tools/emoji-cheat-sheet/" target="_blank">EMOJI CHEAT SHEET</a>
        </div>
      </div>
    </div>
    <div class="b3log-editor__content">
      <div class="b3log-editor__textarea">
        <textarea
          :placeholder="placeholder || ''"
          @keydown="selectHint"
          @keydown.ctrl="hotkey"
          @keydown.meta="hotkey"
          @paste.prevent="pasteToMarkdown"
          @drop.prevent="dragFile"
          @scroll="syncScroll"
          ref="b3logEditor"
          @input="input"
          @focus="showEmojiPanel = false;"
          :value="textareaValue"></textarea>
        <ul v-show="showHint" class="b3log-editor__hints" :style="`left: ${hintX}px; top: ${hintY}px`">
          <li
            v-for="(item, index) in hintData"
            :class="{'b3log-editor__hints--current': currentHintIndex === index}"
            @click="insertHint(item.value + ' ')">
            {{item.value}} {{item.label}}
          </li>
        </ul>
      </div>
      <div v-show="hasPreview" class="b3log-editor__markdown" ref="b3logView"></div>
    </div>
  </div>
</template>

<script>
  require('../../stylus/components/_editor.styl')
  import TurndownService from 'turndown'
  import allEmoji from './emoji.json'
  import { insertTextAtCaret, ajaxUpload, genUploading, genUploaded, replaceTextareaValue } from './tool'

  export default {
    name: 'v-editor',
    props: {
      placeholder: {
        type: String,
        required: false
      },
      uploadURL: {
        type: String,
        required: false
      },
      label: {
        loading: undefined,
        over: undefined,
        error: undefined,
        emoji: undefined,
        bold: undefined,
        italic: undefined,
        quote: undefined,
        link: undefined,
        upload: undefined,
        unorderedList: undefined,
        orderedList: undefined,
        preview: undefined,
        fullscreen: undefined,
        help: undefined
      },
      uploadMax: {
        type: Number,
        required: false
      },
      value: {
        type: String,
        required: true
      },
      height: {
        type: Number,
        required: false
      },
      fetchUpload: {
        type: Function,
        required: false
      }
    },
    data () {
      return {
        textareaValue: '',
        hintX: 0,
        hintY: 0,
        showHint: false,
        currentHintIndex: 0,
        showEmojiPanel: false,
        hintData: [],
        hasPreview: true,
        isFullScreen: false,
        timerId: undefined,
        emoji: {
          smile: '😄',
          joy: '😂',
          '+1': '👍',
          scream: '😱',
          'smiling_imp': '😈',
          sob: '😭',
          yum: '😋',
          tada: '🎉',
          'ok_hand': '👌',
          pray: '🙏',
          punch: '👊',
          'heart_eyes': '😍'
        }
      }
    },
    watch: {
      value: function (newValue) {
        this.$set(this, 'textareaValue', newValue)
      }
    },
    methods: {
      togglePreview () {
        this.$set(this, 'hasPreview', !this.hasPreview)
        if (this.hasPreview) {
          this.$emit('change', this.$refs.b3logEditor.value, this.$refs.b3logView)
        }
      },
      hotkey (event) {
        switch (event.key) {
          case '/':
            this.$set(this, 'showEmojiPanel', !this.showEmojiPanel)
            event.preventDefault()
            break
          case 'b':
            this.insert('**', '**')
            event.preventDefault()
            break
          case 'i':
            this.insert('*', '*')
            event.preventDefault()
            break
          case 'e':
            this.insert('> ', '')
            event.preventDefault()
            break
          case 'k':
            this.insert('[', '](http://)')
            event.preventDefault()
            break
          case 'l':
            if (event.shiftKey) {
              this.insert('1. ', '')
            } else {
              this.insert('* ', '')
            }
            event.preventDefault()
            break
          case 'd':
            this.$set(this, 'hasPreview', !this.hasPreview)
            event.preventDefault()
            break
          case 'a':
            if (event.shiftKey) {
              this.$set(this, 'isFullScreen', !this.isFullScreen)
              event.preventDefault()
            }
            break
          default:
            break
        }
      },
      selectHint (event) {
        // at hints action
        if (!this.showHint) {
          return
        }
        if (event.keyCode === 40) {
          // down
          event.preventDefault()
          if (this.currentHintIndex === this.hintData.length - 1) {
            this.currentHintIndex = 0
          } else {
            this.currentHintIndex++
          }
        } else if (event.keyCode === 38) {
          // up
          event.preventDefault()
          if (this.currentHintIndex === 0) {
            this.currentHintIndex = this.hintData.length - 1
          } else {
            this.currentHintIndex--
          }
        } else if (event.keyCode === 13) {
          // enter
          event.preventDefault()
          this.$set(this, 'showHint', false)

          const hintValue = this.hintData[this.currentHintIndex].value + ' '

          if (/firefox/i.test(navigator.userAgent)) {
            const valueArray = event.target.value.substr(0, event.target.selectionStart).split(':')
            valueArray.pop()
            event.target.value = valueArray.join(':') + hintValue + event.target.value.substr(event.target.selectionStart)
            event.target.selectionEnd = event.target.selectionStart = (valueArray.join(':') + hintValue).length
            this.$set(this, 'textareaValue', event.target.value)
            this._debounceChange()
            return;
          }

          while (!event.target.value.substr(0, event.target.selectionEnd).endsWith(':') &&
          event.target.value.substr(0, event.target.selectionEnd) !== '') {
            document.execCommand('delete', false)
          }
          document.execCommand('delete', false)
          document.execCommand('insertText', false, hintValue)
        }
      },
      insertHint (value) {
        this.$refs.b3logEditor.focus()
        this.$set(this, 'showHint', false)

        if (/firefox/i.test(navigator.userAgent)) {
          const valueArray = this.$refs.b3logEditor.value.substr(0, this.$refs.b3logEditor.selectionStart).split(':')
          valueArray.pop()
          this.$refs.b3logEditor.value = valueArray.join(':') + value + this.$refs.b3logEditor.value.substr(this.$refs.b3logEditor.selectionStart)
          this.$refs.b3logEditor.selectionEnd = this.$refs.b3logEditor.selectionStart = (valueArray.join(':') + value).length
          this.$set(this, 'textareaValue', this.$refs.b3logEditor.value)
          this._debounceChange()
          return;
        }
        while (!this.$refs.b3logEditor.value.substr(0, this.$refs.b3logEditor.selectionEnd).endsWith(':') &&
        this.$refs.b3logEditor.value.substr(0, this.$refs.b3logEditor.selectionEnd) !== '') {
          document.execCommand('delete', false)
        }
        document.execCommand('delete', false)
        document.execCommand('insertText', false, value)
      },
      input (event) {
        // at and emoji hints
        const valueArray = event.target.value.substr(0, event.target.selectionStart).split('\n')
        const xValue = valueArray.slice(-1).pop()

        const genHintsHTML = (data) => {
          if (data.length === 0) {
            this.$set(this, 'showHint', false)
            return
          }
          const zhReg = xValue.match(/[\u4E00-\u9FA5\uF900-\uFA2D\uFF00-\uFFEF]/g)
          const zhLength = zhReg === null ? 0 : zhReg.length
          this.$set(this, 'hintX', zhLength * 15 + (xValue.length - zhLength) * 9 + 10 + event.target.scrollLeft)
          this.$set(this, 'hintY', valueArray.length * 18 + 35 - event.target.scrollTop)
          this.$set(this, 'currentHintIndex', 0)

          this.$set(this, 'showHint', true)
        }

        const getSearchKey = (splitChar) => {
          const xAtArray = xValue.split(splitChar)
          let searchKey
          if (xAtArray.length > 1) {
            if (xAtArray.length === 2 && xAtArray[0] === '') {
              if ((xAtArray[1] === '' || xAtArray[1].trim() !== '') && xAtArray[1].indexOf(' ') === -1 &&
                xAtArray[1].length < 33) {
                searchKey = xAtArray[1]
              }
            } else {
              const prefAt = xAtArray[xAtArray.length - 2]
              const currentAt = xAtArray.slice(-1).pop()
              if (prefAt.slice(-1) === ' ' && currentAt.indexOf(' ') === -1 &&
                ((currentAt === '' || currentAt.trim() !== '') && currentAt.length < 33)) {
                searchKey = currentAt
              }
            }
          }
          return searchKey
        }

        const searchEmoji = getSearchKey(':')
        if (searchEmoji === undefined) {
          this.$set(this, 'showHint', false)
        } else {
          const matchEmoji = []
          const emojies = searchEmoji === '' ? this.emoji : allEmoji
          Object.keys(emojies).forEach((key) => {
            if (matchEmoji.length > 4) {
              return
            }
            if (key.indexOf(searchEmoji.toLowerCase()) > -1) {
              const emojiItem = {
                value: emojies[key],
                label: key
              }
              matchEmoji.push(emojiItem)
            }
          })
          this.$set(this, 'hintData', matchEmoji)
          genHintsHTML(matchEmoji)
        }
        this.$set(this, 'textareaValue', event.target.value)
        this._debounceChange()
      },
      convertHotKey (v) {
        if (v) {
          if (/Mac/.test(navigator.platform)) {
            return v.replace('ctrl', '⌘').replace('shift', '⇧')
          }
          return v
        }
        return ''
      },
      _debounceChange () {
        if (this.timerId !== undefined) {
          clearTimeout(this.timerId)
        }
        this.$set(this, 'timerId', undefined)
        this.$set(this, 'timerId', setTimeout(() => {
          this.$emit('change', this.$refs.b3logEditor.value, this.hasPreview ? this.$refs.b3logView : undefined)
        }, 500))
      },
      selectFile (event) {
        insertTextAtCaret(this.$refs.b3logEditor,
          genUploading(event.target.files, this.uploadMax, this.label.loading, this.label.over), '')
        if (/firefox/i.test(navigator.userAgent)) {
          this._debounceChange()
        }
        ajaxUpload(this.uploadURL, event.target.files, this.uploadMax, (response) => {
          genUploaded(response.data, this.$refs.b3logEditor, this.label.loading, this.label.error)
          if (/firefox/i.test(navigator.userAgent)) {
            this._debounceChange()
          }
          event.target.value = ''
        }, (response) => {
          event.target.value = ''
          response && alert(response.msg)
        })
      },
      dragFile (event) {
        const files = event.dataTransfer.files
        if (files.length === 0) {
          return
        }
        insertTextAtCaret(this.$refs.b3logEditor,
          genUploading(files, this.uploadMax, this.label.loading, this.label.over), '')
        if (/firefox/i.test(navigator.userAgent)) {
          this._debounceChange()
        }
        ajaxUpload(this.uploadURL, files, this.uploadMax, (response) => {
          genUploaded(response.data, this.$refs.b3logEditor, this.label.loading, this.label.error)
          if (/firefox/i.test(navigator.userAgent)) {
            this._debounceChange()
          }
        }, (response) => {
          response && alert(response.msg)
        })
      },
      pasteToMarkdown (event) {
        if (event.clipboardData.getData('text/html').replace(/(^\s*)|(\s*)$/g, '') !== '') {
          let hasCode = false

          let turndownService = new TurndownService({
            converters: [{
              filter: ['pre', 'code'],
              replacement: function (content) {
                if (content.split('\n').length > 1) {
                  hasCode = true
                }
                return '`' + content + '`'
              }
            }, {
              filter: ['img'],
              replacement: (content, target) => {
                if (target.attributes.length === 1) {
                  return ''
                }

                this.fetchUpload && this.fetchUpload(target.src, (originalURL, url) => {
                  if (/firefox/i.test(navigator.userAgent)) {
                    event.target.value = event.target.value.replace(originalURL, url)
                    this.$set(this, 'textareaValue', event.target.value)
                    this._debounceChange()
                    return
                  }

                  replaceTextareaValue(event.target, originalURL, url)
                })

                return `![${target.alt}](${target.src})`
              }
            }],
            gfm: true
          })
          let markdownStr = turndownService.turndown(event.clipboardData.getData('text/html'))

          if (hasCode) {
            insertTextAtCaret(event.target, event.clipboardData.getData('text/plain'), '', true)
          } else {
            const div = document.createElement('div')
            div.innerHTML = markdownStr
            markdownStr = div.innerText.replace(/\n{2,}/g, '\n\n').replace(/(^\s*)|(\s*)$/g, '', true)
            insertTextAtCaret(event.target, markdownStr, '')
          }

          if (/firefox/i.test(navigator.userAgent)) {
            this._debounceChange()
          }
        } else if (event.clipboardData.getData('text/plain').replace(/(^\s*)|(\s*)$/g, '') !== '' &&
          event.clipboardData.files.length === 0) {
          insertTextAtCaret(event.target, event.clipboardData.getData('text/plain'), '', true)
          if (/firefox/i.test(navigator.userAgent)) {
            this._debounceChange()
          }
        } else if (event.clipboardData.files.length > 0) {
          // upload file
          if (this.uploadURL) {
            insertTextAtCaret(this.$refs.b3logEditor,
              genUploading(event.clipboardData.files, this.uploadMax, this.label.loading, this.label.over), '', true)
            if (/firefox/i.test(navigator.userAgent)) {
              this._debounceChange()
            }
            ajaxUpload(this.uploadURL, event.clipboardData.files, this.uploadMax, (response) => {
              genUploaded(response.data, event.target, this.label.loading, this.label.error)
              if (/firefox/i.test(navigator.userAgent)) {
                this._debounceChange()
              }
            }, (response) => {
              response && alert(response.msg)
            })
          }
        }
      },
      syncScroll (event) {
        if (!this.hasPreview) {
          return
        }
        const textScrollTop = event.target.scrollTop
        const textHeight = event.target.clientHeight
        const textScrollHeight = event.target.scrollHeight
        if ((textScrollTop / textHeight > 0.5)) {
          this.$refs.b3logView.scrollTop = (textScrollTop + textHeight) *
            this.$refs.b3logView.scrollHeight / textScrollHeight - textHeight
        } else {
          this.$refs.b3logView.scrollTop = textScrollTop *
            this.$refs.b3logView.scrollHeight / textScrollHeight
        }
      },
      insert (prefix, suffix, hasReplaced) {
        insertTextAtCaret(this.$refs.b3logEditor, prefix, suffix, hasReplaced)
        if (/firefox/i.test(navigator.userAgent)) {
          this._debounceChange()
        }
      }
    },
    mounted () {
      this.$set(this, 'textareaValue', this.value)
      this._debounceChange()
    }
  }
</script>
