<template>
  <div id="app">
    <header class="header">
      <div class="container">
        <div class="header-left">
          <h1>
            <router-link to="/">JSX Live Editor</router-link>
          </h1>
          <h2>{{ version }}, <a target="_blank" href="https://github.com/sodatea/vue-jsx-playground">check out source code</a></h2>
        </div>
        <div class="header-right">
          <select aria-label="Select JSX mode" class="form-control" v-model="mode">
            <option value="vue">Vue</option>
            <option value="react">React</option>
          </select>
          <button class="form-control save-button" @click="saveGist">Save as Gist</button>
        </div>
      </div>
    </header>
    <div class="editors">
      <editor-window title="input" width="500px" style="margin: 0 20px">
        <code-mirror class="input" v-model="code" :options="editorOptions"></code-mirror>
      </editor-window>
      <editor-window title="result" width="500px" style="margin: 0 20px">
        <div class="result">
          <pre class="code cm-s-default"><code v-html="result"></code></pre>
          <div class="error" v-show="error">{{ error }}</div>
        </div>
      </editor-window>
    </div>
  </div>
</template>

<script>
import { EditorWindow } from 'vue-windows'
import highlight from 'cm-highlight'
import CodeMirror from 'vue-cm'
import axios from 'axios'
import progress from 'nprogress'
import 'codemirror/mode/javascript/javascript'
import 'codemirror/mode/jsx/jsx'

import { version as BABEL_VERSION } from '@babel/standalone/package.json'
import { version as VUE_JSX_VERSION } from '@vue/babel-preset-jsx/package.json'

export default {
  name: 'JSXEditor',
  components: {
    CodeMirror,
    EditorWindow
  },
  data() {
    const defaultValue = `
<div id="welcome">
  <h1>Hello World!</h1>
</div>
`.trim()
    const { input, mode } = this.$route.query
    return {
      result: 'Loading...',
      error: '',
      mode: mode || 'vue',
      code: input || defaultValue,
      version: `@babel/standalone@${BABEL_VERSION} & @vue/babel-preset-jsx@${VUE_JSX_VERSION}`,
      editorOptions: {
        mode: 'jsx',
        tabSize: 2,
        indentWithTabs: false,
        extraKeys: {
          Tab: cm => {
            cm.replaceSelection(' '.repeat(cm.getOption('tabSize')))
          }
        }
      }
    }
  },
  created() {
    if (this.$route.name === 'gist') {
      this.fetchGist(this.$route.params.id)
    }
  },
  mounted() {
    this.transform()
  },
  watch: {
    code() {
      this.transform()
    },
    mode() {
      this.transform()
      this.$router.push({
        query: {
          ...this.$route.query,
          mode: this.mode
        }
      })
    }
  },
  methods: {
    async transform() {
      const code = this.code
      try {
        const [babel, vueJSXPreset] = await Promise.all([
          import('@babel/standalone'),
          import('@vue/babel-preset-jsx')
        ])
        const transformOptions = {
          presets: [],
          plugins: []
        }
        if (this.mode === 'vue') {
          transformOptions.presets.push(vueJSXPreset)
        } else if (this.mode === 'react') {
          transformOptions.presets.push('react')
        }
        const result = babel.transform(code, transformOptions)
        this.result = highlight(result.code, {
          mode: 'jsx'
        })
        this.error = null
      } catch (err) {
        this.error = err.message
      }
    },
    async saveGist() {
      progress.start()
      const res = await axios.post(`https://api.github.com/gists?access_token=${process.env.APP_GH_TOKEN}`, {
        description: 'Saved by https://jsx.egoist.sh',
        files: {
          [`${this.mode}.jsx`]: {
            content: this.code
          }
        }
      })
      this.$router.push(`/gist/${res.data.id}`)
      progress.done()
    },
    async fetchGist(id) {
      progress.start()
      const { data } = await axios.get(`https://api.github.com/gists/${id}?access_token=${process.env.APP_GH_TOKEN}`)
      if (data.files['vue.jsx']) {
        this.mode = 'vue'
        this.code = data.files['vue.jsx'].content
      } else if (data.files['react.jsx']) {
        this.mode = 'react',
        this.code = data.files['react.jsx'].content
      }
      progress.done()
    }
  }
}
</script>

<style src="normalize.css/normalize.css"></style>
<style src="codemirror/lib/codemirror.css"></style>
<style src="vue-windows/dist/vue-windows.css"></style>
<style src="nprogress/nprogress.css"></style>

<style lang="scss">
html, body, #app, .CodeMirror {
  height: 100%;
}
body {
  margin: 0;
  font: 14px/1.4 -apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Oxygen,Ubuntu,Cantarell,Fira Sans,Helvetica Neue,sans-serif;
}
* {
  box-sizing: border-box;
}
.container {
  max-width: 1080px;
  margin: 0 auto;
  height: 100%;
}
.header {
  height: 80px;
  background-color: #4fc08d;
  color: white;
  >.container {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  h1 {
    margin: 0;
    font-weight: 400;
    a {
      text-decoration: none;
      color: white;
    }
  }
  h2 {
    margin: 0;
    font-weight: 400;
    font-size: 14px;
    a {
      color: white;
    }
  }
  .header-right {
    display: flex;
    align-items: center;
  }
}
.editors {
  background-color: #f9f9f9;
  height: calc(100% - 80px);
  display: flex;
  align-items: center;
  justify-content: center;
}
.input {
  border: none;
  outline: none;
  resize: none;
  font-size: 1rem;
}
.result {
  position: relative;
  height: 100%;
  background-color: white;
  overflow: auto;
}
.code {
  margin: 0;
  height: 100%;
}
.error {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background-color: red;
  color: white;
  padding: 0;
  overflow: auto;
  padding: 10px;
  font-size: 16px;
  white-space: pre;
}
.form-control {
  height: 26px;
}
.save-button {
  background: #eee;
  border: none;
  border-radius: 3px;
  margin-left: 5px;
}
</style>
