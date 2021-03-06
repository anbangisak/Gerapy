<template>
  <el-row>
    <el-col :span="5">
      <div class="panel" id="tree">
        <panel-title :title="projectName">
        </panel-title>
        <div class="panel-body">
          <el-tree :data="tree" :props="defaultProps" @node-click="handleNodeClick"></el-tree>
        </div>
      </div>
    </el-col>
    <el-col :span="19">
      <div class="panel m-l-md">
        <panel-title title="代码">
          <el-button @click="renameFile" size="mini" type="primary" v-if="activeFile">
            <i class="fa fa-edit"></i>
            重命名
          </el-button>
          <el-button @click="createFile" size="mini" type="info" v-if="activeFile">
            <i class="fa fa-plus"></i>
            新建
          </el-button>
          <el-button @click="deleteFile" size="mini" type="danger" v-if="activeFile">
            <i class="fa fa-close"></i>
            删除
          </el-button>
        </panel-title>
        <div class="panel-body">
          <codemirror v-model="code" :options="editorOptions" @change="onEditorCodeChange">
          </codemirror>
        </div>
      </div>
    </el-col>
  </el-row>
</template>
<script type="text/javascript">
  import {panelTitle, bottomToolBar} from 'components'
  import {codemirror, CodeMirror} from 'vue-codemirror'

  // Key Map
  import 'codemirror/mode/clike/clike'
  import 'codemirror/addon/edit/matchbrackets'
  import 'codemirror/addon/comment/comment'
  import 'codemirror/addon/dialog/dialog'
  import 'codemirror/addon/dialog/dialog.css'
  import 'codemirror/addon/search/searchcursor'
  import 'codemirror/addon/search/search'
  import 'codemirror/keymap/sublime'

  // CloseBrackets
  import 'codemirror/addon/edit/closebrackets'

  // Active Line
  import 'codemirror/addon/selection/active-line'

  // StyleSelectedText
  import 'codemirror/addon/selection/mark-selection'
  import 'codemirror/addon/search/searchcursor'

  // Hint
  import 'codemirror/addon/hint/show-hint'
  import 'codemirror/addon/hint/show-hint.css'
  import 'codemirror/addon/hint/anyword-hint'

  // foldGutter
  import 'codemirror/addon/fold/foldgutter.css'
  import 'codemirror/addon/fold/brace-fold'
  import 'codemirror/addon/fold/comment-fold'
  import 'codemirror/addon/fold/foldcode'
  import 'codemirror/addon/fold/foldgutter'
  import 'codemirror/addon/fold/indent-fold'
  import 'codemirror/addon/fold/markdown-fold'
  import 'codemirror/addon/fold/xml-fold'

  // Mode

  import 'codemirror/mode/css/css'
  import 'codemirror/mode/dockerfile/dockerfile'
  import 'codemirror/mode/htmlmixed/htmlmixed'
  import 'codemirror/mode/markdown/markdown'
  import 'codemirror/mode/php/php'
  import 'codemirror/mode/python/python'
  import 'codemirror/mode/sass/sass'
  import 'codemirror/mode/javascript/javascript'
  import 'codemirror/mode/sql/sql'
  import 'codemirror/mode/vue/vue'

  export default{
    data(){
      return {
        projectName: this.$route.params.name,
        clientData: null,
        //请求时的loading效果
        loadData: true,
        //批量选择数组
        batchSelect: [],
        tree: [],
        defaultProps: {
          children: 'children',
          label: 'label'
        },
        activeFile: null,
        activeNode: null,
        code: 'Please Choose File from Left to Edit.',
        modeMap: {
          'py': 'text/x-python',
          'js': 'text/javascript',
          'html': 'text/html',
          'vue': 'text/x-vue',
          'md': 'text/x-markdown',
          'scss': 'text/x-sass',
          'css': 'text/x-css',
          'php': 'application/x-httpd-php',
        },
        editorOptions: {
          tabSize: 4,
          mode: 'text/javascript',
          theme: 'monokai',
          lineNumbers: true,
          styleActiveLine: true,
          line: true,
          // sublime、emacs、vim三种键位模式
          keyMap: 'sublime',
          // 按键映射，比如Ctrl键映射autocomplete，autocomplete是hint代码提示事件
          extraKeys: {
            'Tab': 'autocomplete'
          },
          // 代码折叠
          foldGutter: true,
          gutters: [
            'CodeMirror-linenumbers', 'CodeMirror-foldgutter'
          ],
          // 选中文本自动高亮，及高亮方式
          styleSelectedText: true,
          highlightSelectionMatches: {
            showToken: /\w/,
            annotateScrollbar: true
          },
        }
      }
    },
    components: {
      panelTitle,
      bottomToolBar,
      codemirror
    },
    created(){
      this.getProjectTree(this.projectName)

    },
    methods: {
      getProjectTree(name) {
        // 获取目录树
        this.$fetch.apiProject.projectTree({
          name: name
        }).then(({data: tree}) => {
          this.tree = tree
        }).catch(() => {
          this.$message.error('加载项目失败')
        })
      },
      handleNodeClick(data) {
        this.activeNode = data
        this.changeCode(data)
        this.activeFile = this.activeNode.path + '/' + this.activeNode.label
      },
      // 呈现代码
      changeCode(data) {
        // 获取后缀
        let group = this.activeNode.label.split('.')
        let extension = group[group.length - 1]
        // 设置Mode
        this.editorOptions.mode = this.modeMap[extension]
        // 加载文件
        if (!data.children) {
          this.$fetch.apiProject.projectFileRead(
            data
          ).then(({data: code}) => {
            this.code = code
          }).catch(() => {
            this.$message.error('加载失败')
          })
        }
      },
      renameFile() {
        this.$prompt('修改文件', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          inputValue: this.activeNode.label,
          inputPattern: /[^\/\s]/,
          inputErrorMessage: '格式不正确'
        }).then(({value}) => {
          // 删除文件
          this.$fetch.apiProject.projectFileRename({
            path: this.activeNode.path,
            pre: this.activeNode.label,
            new: value
          }).then(() => {
            this.$message.success('修改成功')
            this.getProjectTree(this.projectName)
          }).catch(() => {
            this.$message.error('修改失败')
          })
        })
      },
      createFile() {
        this.$prompt('请输入文件名', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          inputValue: this.activeNode.label,
          inputPattern: /[^\/\s]/,
          inputErrorMessage: '格式不正确'
        }).then(({value}) => {
          // 删除文件
          this.$fetch.apiProject.projectFileCreate({
            path: this.activeNode.path,
            name: value
          }).then(() => {
            this.$message.success('创建成功')
            this.getProjectTree(this.projectName)
          }).catch(() => {
            this.$message.error('创建失败')
          })
        })
      },
      deleteFile() {
        this.$confirm('确定要删除吗?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          // 删除文件
          this.$fetch.apiProject.projectFileDelete({
            path: this.activeNode.path,
            label: this.activeNode.label
          }).then(() => {
            this.$message.success('删除成功')
            this.getProjectTree(this.projectName)
          }).catch(() => {
            this.$message.error('删除失败')
          })
        }).catch((error) => {
          console.log(error)
        })
      },
      onEditorCodeChange(code, callback) {
        //更新文件
        this.$fetch.apiProject.projectFileUpdate({
          code: code,
          path: this.activeNode['path'],
          label: this.activeNode['label']
        }).then(() => {
          if (callback) {
            callback()
          }
        }).catch(() => {
          this.$message.error('修改失败')
        })
      }
    }
  }
</script>

<style lang="scss" type="text/scss" rel="stylesheet/scss">
  .CodeMirror {
    font-size: 16px;
    min-height: 600px;
  }

  #tree {
    .panel-body {
      overflow: hidden;
    }
    .el-tree {
      overflow-x: scroll;
      min-height: 590px;
      border: none;
    }
  }

  .el-tree-node.is-current {
    background: #EEE;
  }

  .CodeMirror {
    z-index: 0;
  }
</style>
