<template>
  <transition name="fade">
    <div class="em-editor" v-show="value.show">
      <div class="em-editor__editor">
        <div ref="codeEditor"></div>
      </div>
      <div class="panel-info" style="overflow-y: scroll!important;padding-top: 10%!important;padding-bottom: 10%!important;">
        <em-spots :size="10"></em-spots>
        <div class="wrapper">
          <h2>{{isEdit ? $t('p.detail.editor.title[0]') : $t('p.detail.editor.title[1]')}}</h2>
          <div class="em-editor__form">
            <Form label-position="top">
              <Form-item label="ServiceName">
                <i-input v-model="temp.serviceName">
                  <!--<span slot="prepend">/</span>-->
                </i-input>
              </Form-item>
              <Form-item label="protoName">
                <i-input v-model="temp.protoName">
                  <!--<span slot="prepend">/</span>-->
                </i-input>
              </Form-item>
              <Form-item label="Method">
                <i-select v-model="temp.method">
                  <Option v-for="item in methods" :value="item.value" :key="item.value">{{ item.label }}</Option>
                </i-select>
              </Form-item>
              <Form-item label="URL">
                <i-input v-model="temp.url">
                  <span slot="prepend">/</span>
                </i-input>
              </Form-item>

              <!--<FormItem-->
                <!--v-for="(item, index) in temp.params"-->
                <!--v-if="(temp.method!=='get' && temp.method!=='delete') && item.status"-->
                <!--:key="index"-->
                <!--:label="'Item ' + item.index"-->
                <!--:prop="'items.' + index + '.value'"-->
                <!--:rules="{required: true, message: 'Params ' + item.index +' can not be empty', trigger: 'blur'}">-->
                <!--<Row>-->
                  <!--<Col span="9">-->
                  <!--<Input type="text" v-model="item.key" placeholder="Enter key..."></Input>-->
                  <!--</Col>-->
                  <!--<Col span="9">-->
                  <!--<Input type="text" v-model="item.value" placeholder="Enter value..."></Input>-->
                  <!--</Col>-->
                  <!--<Col span="4" offset="1">-->
                  <!--<Button type="ghost" @click="handleRemove(index)">Delete</Button>-->
                  <!--</Col>-->
                <!--</Row>-->
              <!--</FormItem>-->
              <!--<FormItem  v-if="(temp.method!=='get' && temp.method!=='delete')">-->
                <!--<Row>-->
                  <!--<Col span="12">-->
                  <!--<Button type="dashed" long @click="handleAdd" icon="plus-round">Add params</Button>-->
                  <!--</Col>-->
                <!--</Row>-->
              <!--</FormItem>-->

              <Form-item :label="$t('p.detail.editor.params')">
                <i-input type="textarea" rows="5" v-model="temp.params"></i-input>
              </Form-item>
              <Form-item :label="$t('p.detail.columns[0]')">
                <i-input type="textarea" rows="5" v-model="temp.description"></i-input>
              </Form-item>
              <Form-item :label="$t('p.detail.editor.autoClose')" v-if="isEdit">
                <i-switch v-model="autoClose"></i-switch>
              </Form-item>
              <Form-item>
                <Button type="primary" long @click="submit">{{isEdit ? $t('p.detail.editor.action[0]') : $t('p.detail.editor.action[1]')}}</Button>
              </Form-item>
            </Form>
          </div>
          <div class="em-editor__control">
            <div class="em-proj-detail__switcher">
              <ul>
                <li @click="format">{{$t('p.detail.editor.control[0]')}}</li>
                <li @click="preview" v-if="isEdit">{{$t('p.detail.editor.control[1]')}}</li>
                <li @click="close">{{$t('p.detail.editor.control[2]')}}</li>
              </ul>
            </div>
          </div>
          <div class="em-editor__control">
            <div class="em-proj-detail__switcher">
              <ul>
                <li @click="getEditorContent">复制JSON</li>
              </ul>
            </div>
          </div>
          <div class="wrapper" v-if="editorContent!==''">
            <div class="em-editor__form">
              <Form label-position="top">
                <!--<Form-item>-->
                  <!--<Button type="primary" long @click="getEditorContent">复制配置</Button>-->
                <!--</Form-item>-->
                <Form-item>
                  <i-input type="textarea" rows="10" v-model="editorContent"></i-input>
                </Form-item>
              </Form>
            </div>
        </div>
      </div>
    </div>
  </transition>
</template>

<script>
import * as api from '../../api'
import jsBeautify from 'js-beautify/js/lib/beautify'
let ace

if (typeof window !== 'undefined') {
  ace = require('brace')
  require('brace/mode/javascript')
  require('brace/theme/monokai')
  require('brace/ext/language_tools')
  require('brace/ext/searchbox')
  require('./snippets')
}

export default {
  props: {
    value: {}
  },
  data () {
    return {
      index: 1,
      codeEditor: null,
      autoClose: true,
      methods: [
        { label: 'get', value: 'get' },
        { label: 'post', value: 'post' },
        { label: 'put', value: 'put' },
        { label: 'delete', value: 'delete' },
        { label: 'patch', value: 'patch' }
      ],
      editorContent: '',
      temp: {
        url: '',
        mode: '',
        method: '',
        serviceName: '',
        protoName: '',
        params: '',
        // params: [
        //   {
        //     key: '',
        //     value: '',
        //     index: 1,
        //     status: 1
        //   }
        // ],
        description: ''
      }
    }
  },
  computed: {
    isEdit () {
      return !!this.value._id
    }
  },
  mounted () {
    this.codeEditor = ace.edit(this.$refs.codeEditor)
    this.codeEditor.getSession().setMode('ace/mode/javascript')
    this.codeEditor.setTheme('ace/theme/monokai')
    this.codeEditor.setOption('tabSize', 2)
    this.codeEditor.setOption('fontSize', 15)
    this.codeEditor.setOption('enableLiveAutocompletion', true)
    this.codeEditor.setOption('enableSnippets', true)
    this.codeEditor.clearSelection()
    this.codeEditor.getSession().setUseWorker(false)
    this.codeEditor.on('change', this.onChange)
  },
  watch: {
    'value.show': function (show) {
      document.body.style.overflow = show ? 'hidden' : 'auto'
      if (show) {
        if (this.isEdit) {
          this.autoClose = true
          this.temp.url = this.value.url.slice(1) // remove /
          this.temp.mode = this.value.mode
          this.temp.method = this.value.method
          this.temp.serviceName = this.value.serviceName
          this.temp.protoName = this.value.protoName
          this.temp.description = this.value.description
          this.temp.params = this.value.params
          this.codeEditor.setValue(this.temp.mode)
        } else {
          this.temp.url = ''
          this.temp.mode = '{"data": {}}'
          this.temp.method = 'get'
          this.temp.serviceName = ''
          this.temp.protoName = ''
          this.temp.description = ''
          this.temp.params = ''
          // this.temp.params = '[{key: "",value: "",index: 1,status: 1}]'
          this.codeEditor.setValue(this.temp.mode)
        }
        this.format()
      }
    }
  },
  methods: {
    // handleAdd () {
    //   this.index++
    //   this.temp.params.push({
    //     key: '',
    //     value: '',
    //     index: this.index,
    //     status: 1
    //   })
    // },
    // handleRemove (index) {
    //   this.temp.params[index].status = 0
    // },
    getEditorContent () {
      const context = this.codeEditor.getValue()
      let code = /^http(s)?/.test(context)
        ? context
        : jsBeautify.js_beautify(context, { indent_size: 2 })
      this.editorContent = code
    },
    convertUrl (url) {
      const newUrl = '/' + url
      return newUrl === '/'
        ? '/'
        : newUrl.replace(/\/\//g, '/').replace(/\/$/, '')
    },
    format () {
      const context = this.codeEditor.getValue()
      let code = /^http(s)?/.test(context)
        ? context
        : jsBeautify.js_beautify(context, { indent_size: 2 })
      this.codeEditor.setValue(code)
    },
    onChange () {
      this.temp.mode = this.codeEditor.getValue()
    },
    close () {
      this.value.show = false
      this.$emit('input', this.value)
    },
    submit () {
      // if (this.temp.method === 'get' || this.temp.method === 'delete') {
      //   this.temp.params = []
      // } else {
      //   this.temp.params = this.temp.params.filter((p) => p.status !== 0)
      // }
      console.log(this.temp)
      console.log(...this.temp)
      const testStr = /^[A-Za-z]+$/
      if (!testStr.test(this.temp.serviceName)) {
        this.$Message.error('ServiceName须为纯英文')
        return
      }
      const mockUrl = this.convertUrl(this.temp.url)

      try {
        const value = (new Function(`return ${this.temp.mode}`))() // eslint-disable-line
        if (!value) {
          this.$Message.error(this.$t('p.detail.editor.submit.error[0]'))
          return
        } else if (typeof value !== 'object') {
          throw new Error()
        }
      } catch (error) {
        if (!/^http(s)?:\/\//.test(this.temp.mode)) {
          this.$Message.error(error.message || this.$t('p.detail.editor.submit.error[1]'))
          return
        }
      }

      if (this.isEdit) {
        api.mock.update({
          data: {
            ...this.temp,
            id: this.value._id,
            url: mockUrl
          }
        }).then((res) => {
          if (res.data.success) {
            this.$Message.success(this.$t('p.detail.editor.submit.updateSuccess'))
            this.value.url = mockUrl
            this.value.params = this.temp.params
            this.value.mode = this.temp.mode
            this.value.method = this.temp.method
            this.value.serviceName = this.temp.serviceName
            this.value.protoName = this.temp.protoName
            this.value.description = this.temp.description
            if (this.autoClose) this.close()
          }
        })
      } else {
        console.log('create')
        console.log(...this.temp)
        this.$store.dispatch('mock/CREATE', {
          route: this.$route,
          ...this.temp,
          url: mockUrl
        }).then((res) => {
          if (res.data.success) {
            this.$Message.success(this.$t('p.detail.create.success'))
            this.close()
          }
        })
      }
    },
    preview () {
      window.open(this.$parent.baseUrl + this.value.url + '#!method=' + this.value.method)
    }
  }
}
</script>
