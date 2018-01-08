<template>
  <textarea></textarea>
</template>

<script>
const theme = 'neo'
const mode = { name: 'javascript', globalVars: true }
const CodeMirror = require('codemirror/lib/codemirror.js')
require('codemirror/lib/codemirror.css')
var beautify = require('js-beautify').js_beautify

require.ensure([], function (require) {
  //require(`codemirror/theme/${theme}.css`)
  require(`./themes/${theme}.css`)
  require('codemirror/addon/display/fullscreen.css')
  require('codemirror/addon/display/fullscreen.js')
})

export default {
  data: function () {
    return {
      content: '',
      editor: null,
      keyMap: { 'Ctrl-Space': 'autocomplete', "Enter": this.onEnterBeautify }
    }
  },

  props: {
    code: String,
    value: String,
    options: {
      type: Object,
      default: function () {
        return { 
          extraKeys: this.keyMap,
          mode: mode,
          line: true,
          lineNumbers: true,
          lineWrapping: true,
          lint: true,
          gutters: ['CodeMirror-linenumbers', 'CodeMirror-lint-markers'],
          highlightSelectionMatches: { minChars: 1 },
          matchBrackets: true,
          autoCloseBrackets: true,
          styleActiveLine: true

        }
      }
    },
    lintOptions: {
      type: Object,
      default: function () {
        return { sub: true }
      }
    },
    completions: {
      type: Function,
      default: (word) => {}
    },
    replaceRange: {
      type: Object,
      default: function () {
        return null
      }
    }
  },

  methods: {
    cursorContext (cm) {
      let cursor = cm.getCursor(), line = cm.getLine(cursor.line)
      let start = cursor.ch, end = cursor.ch
      while (start && /[\wæøåÆØÅ]/.test(line.charAt(start - 1))) --start
      while (end < line.length && /[\wæøåÆØÅ]/.test(line.charAt(end))) ++end
      let word = line.slice(start, end)

      return {
        word,
        start,
        end,
        line: cursor.line
      }
    },

    onEnterBeautify() {
      let cursor = this.editor.getCursor()
      let cursorLineIndex = cursor.line
      let cursorCharIndex = cursor.ch
      let isCursorLinePrecededOnlyWithEmptyLines = this.isLinePrecededOnlyWithEmptyLines(cursorLineIndex)

      let lastCharacterBeforeCursor = this.getLastCharacterBeforeCursor(cursorLineIndex, cursorCharIndex)
      
      if(this.shouldBeautifyCode(cursorLineIndex, lastCharacterBeforeCursor, isCursorLinePrecededOnlyWithEmptyLines)){
        this.beautifyCode(cursorLineIndex, lastCharacterBeforeCursor, isCursorLinePrecededOnlyWithEmptyLines)
      }

      return CodeMirror.Pass;
    },

    shouldBeautifyCode(cursorLineIndex, lastCharacterBeforeCursor, isCursorLinePrecededOnlyWithEmptyLines) {
      let lastLineIndex = this.editor.lineCount()-1
      return ((cursorLineIndex < lastLineIndex && (!isCursorLinePrecededOnlyWithEmptyLines || lastCharacterBeforeCursor))) 
         || (cursorLineIndex===lastLineIndex && this.editor.getLine(cursorLineIndex)!=="")
    },

    getLastCharacterBeforeCursor(cursorLineIndex, cursorCharIndex) {
      let lastCharacterBeforeCursor = this.editor.getLine(cursorLineIndex)[cursorCharIndex-1]
      if(cursorCharIndex !== 0 && (!lastCharacterBeforeCursor || !lastCharacterBeforeCursor.trim())){
        lastCharacterBeforeCursor = this.editor.getLine(cursorLineIndex).trim().slice(-1)
      }
      return lastCharacterBeforeCursor
    },

    isLinePrecededOnlyWithEmptyLines(lineNumber) {
      for(let i = 0; i<lineNumber; i++) {
        if(this.editor.getLine(i) && this.editor.getLine(i).trim())
          return false
      }
      return true
    },

    beautifyCode (cursorLineIndex, lastCharacterBeforeCursor, isCursorLinePrecededOnlyWithEmptyLines) {
      let beautifiedCode = beautify(this.editor.getValue(), {"end_with_newline": true,  "preserve_newlines": true, "max_preserve_newlines": 100, "indent_size": 2})

      let newCursorLineIndex = isCursorLinePrecededOnlyWithEmptyLines ? 0 : cursorLineIndex
      let newCursorCharIndex = lastCharacterBeforeCursor ? beautifiedCode.split('\n')[newCursorLineIndex].lastIndexOf(lastCharacterBeforeCursor) + 1 : 0

      this.editor.setValue(beautifiedCode)
      this.editor.focus()
      this.editor.setCursor({line: newCursorLineIndex, ch: newCursorCharIndex})
    }
  },

  created () {
    // Require language mode config & basic addons.
    require(`codemirror/mode/javascript/javascript.js`)
    require('codemirror/addon/lint/lint.js')
    require('codemirror/addon/lint/lint.css')
    require('codemirror/addon/lint/javascript-lint.js')
    require('codemirror/addon/hint/show-hint.css')
    require('codemirror/addon/hint/show-hint.js')
    require('codemirror/addon/hint/javascript-hint.js')
    require('codemirror/addon/edit/closebrackets.js')
    require('codemirror/addon/search/match-highlighter.js')
    require('codemirror/addon/selection/active-line.js')
    require('codemirror/addon/edit/matchbrackets.js')
  },

  mounted () {
    this.options.hintOptions = {
      hint: (cm, options) => {
        // Get cursor context
        let c = this.cursorContext(cm)
        // Completions from props
        let suggestions = this.completions(c.word)

        return {
          list: suggestions,
          from: CodeMirror.Pos(c.line, c.start),
          to: CodeMirror.Pos(c.line, c.end)
        }
      }
    }

    const options = { ...this.options, lint: this.lintOptions, theme: theme }
    this.editor = CodeMirror.fromTextArea(this.$el, options)
    this.editor.setValue(this.code || this.value || this.content)
    this.editor.addKeyMap(this.keyMap)

    this.editor.on('change', (cm) => {
      this.content = cm.getValue()
      if (!!this.$emit) {
        this.$emit('changed', this.content)
        this.$emit('input', this.content)
      }
    })

    this.editor.on('cursorActivity', (cm) => {
      if (!!this.$emit) {
        this.$emit('cursor', this.cursorContext(cm))
      }
    })

    if (!window.JSHINT) window.JSHINT = require('jshint').JSHINT
  },

  beforeDestroy () { window.JSHINT = null },

  watch: {
    code (newVal, oldVal) {
      const editorValue = this.editor.getValue()
      if (newVal !== editorValue) {
        let scrollInfo = this.editor.getScrollInfo()
        this.editor.setValue(newVal)
        this.content = newVal
        this.editor.scrollTo(scrollInfo.left, scrollInfo.top)
      }
    },

    value (newVal, oldVal) {
      const editorValue = this.editor.getValue()
      if (newVal !== editorValue) {
        let scrollInfo = this.editor.getScrollInfo()
        this.editor.setValue(newVal)
        this.content = newVal
        this.editor.scrollTo(scrollInfo.left, scrollInfo.top)
      }
    },

    /**
     * Replace string from & to a certain range. If `to` is equal to `from` it will just insert a string at that position.
     * @see https://codemirror.net/doc/manual.html#replaceRange
     */
    replaceRange: {
      handler (newVal, oldVal) {
        if (newVal && this.editor) {
          const cursor = this.editor.getCursor()
          const cursorLineAndCh = {
            line: cursor.line,
            ch: cursor.ch
          }
          const insert = {
            ...newVal,
            from: newVal.from || cursorLineAndCh,
            to: newVal.to || cursorLineAndCh
          }

          this.editor.replaceRange(
            insert.replacement,
            insert.from,
            insert.to
          )
        }
      },
      deep: true
    }
  }
}
</script>

<style scoped>
.CodeMirror-code {
  /*line-height: 1.6em;
  font-family: Menlo, Monaco, Consolas, "Courier New", monospace;*/
}
</style>