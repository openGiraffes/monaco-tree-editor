# monaco-tree-editor

[English](https://github.com/AlphaFoxz/monaco-tree-editor/blob/main/README.md) | 简体中文

## 这个库做了什么

- [x] 提供 VSCode 风格的文件树
- [x] vue 事件采用异步 api
- [x] 提供一个浮动的全局消息 hook。

## 前置要求

- [vue3 v3.3.4+](https://cn.vuejs.org/) 或者 [nuxt3 v3.0+](https://nuxt.com.cn/)
- [monaco-editor v0.44.0+](https://microsoft.github.io/monaco-editor/)
- 【推荐】[pnpm 包管理器](https://www.pnpm.cn/)

## 在线预览地址

[在 Stackblitz 中运行](https://stackblitz.com/~/github.com/AlphaFoxz/monaco-tree-editor)

## 如何安装

### 1.执行命令

```shell
pnpm add monaco-tree-editor
#or
npm i monaco-tree-editor
```

### 2.复制必要的静态文件

{root}/node_modules/monaco-tree-editor/`monaco-tree-editor-statics` => {root}/public/`monaco-tree-editor-statics`

## 示例代码

### 模拟服务端

`mock-server.ts`

```typescript
import { type Files } from 'monaco-tree-editor'

const fileSeparator = '\\'
let responseFiles: Files = {
  'F:\\test_project\\test.html': {
    isFile: true,
    content: '<html><body><h1>Hello World!</h1></body></html>',
  },
  'F:\\test_project\\components': {
    isFolder: true,
  },
  'F:\\test_project\\index.ts': {
    isFile: true,
    content: 'console.log("hello world")',
  },
  'F:\\test_project\\api\\TestApi.ts': {
    isFile: true,
    content: 'console.log("hello world")',
  },
  'F:\\test_project\\dto\\TestDto.ts': {
    isFile: true,
    content: 'console.log("hello world")',
  },
}
// 模拟延迟，测试健壮性
export const delay = async (maxMs = 3000) => {
  return new Promise<void>((resolve) => {
    setTimeout(() => {
      resolve()
    }, Math.random() * maxMs)
  })
}
export const fetchFiles = async () => {
  await delay(1000)
  return await JSON.parse(JSON.stringify(responseFiles))
}
export const createOrSaveFile = async (path: string, content: string) => {
  await delay()
  if (responseFiles[path]) {
    if (!responseFiles[path].isFile) {
      throw new Error(`save file:[ ${path} ] is not a file!`)
    }
    responseFiles[path].content = content
  } else {
    responseFiles[path] = {
      isFile: true,
      content,
    }
  }
}
export const newFile = async (path: string) => {
  await delay()
  if (responseFiles[path]) {
    throw new Error(`new file: [ ${path} ] already exists!`)
  }
  responseFiles[path] = {
    isFile: true,
    content: '',
  }
}
export const newFolder = async (path: string) => {
  await delay()
  if (responseFiles[path]) {
    throw new Error(`new folder: [ ${path} ] already exists!`)
  }
  responseFiles[path] = {
    isFolder: true,
  }
}
export const rename = async (path: string, newPath: string) => {
  await delay()
  if (!responseFiles[path]) {
    throw new Error(`rename: source file/folder name [ ${path} ] not exists!`)
  } else if (responseFiles[newPath]) {
    throw new Error(`rename: target file/folder name [ ${newPath} ] already exists!`)
  }
  responseFiles[newPath] = responseFiles[path]
  if (path !== newPath) {
    delete responseFiles[path]
  }
  return true
}
export const deleteFile = async (path: string) => {
  await delay()
  if (!responseFiles[path]) {
    throw new Error(`delete: file name [ ${path} ] not exists!`)
  }
  delete responseFiles[path]
  return true
}
```

### 基础用法

```typescript
import { Editor as MonacoTreeEditor, useMonaco, type Files } from 'monaco-tree-editor'
import 'moanco-tree-editor/index.css'
import { ref } from 'vue'
import * as monaco from 'monaco-editor'
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker'
import jsonWorker from 'monaco-editor/esm/vs/language/json/json.worker?worker'
import tsWorker from 'monaco-editor/esm/vs/language/typescript/ts.worker?worker'
import htmlWorker from 'monaco-editor/esm/vs/language/html/html.worker?worker'
import cssWorker from 'monaco-editor/esm/vs/language/css/css.worker?worker'
import * as server from './mock-server'

// ================ 初始化 ================
window.MonacoEnvironment = {
  getWorker: function (_moduleId, label: string) {
    if (label === 'json') {
      return new jsonWorker()
    } else if (label === 'ts' || label === 'typescript') {
      return new tsWorker()
    } else if (label === 'html' || label === 'handlebars' || label === 'razor') {
      return new htmlWorker()
    } else if (label === 'css' || label === 'scss' || label === 'less') {
      return new cssWorker()
    }
    return new editorWorker()
  },
  globalAPI: true,
}
let monacoStore
// 模拟延迟，测试健壮性
server.delay().then(() => {
  monacoStore = useMonaco(monaco)
})

// ================ 回调函数 =================
/*
不论服务端的文件名是什么，组件内会统一去掉最长公共前缀，而在回调方法中的路径则会拼接回原路径
例如
const serverFiles = {
  'F:\\test_project\\index.ts': {...},
  'F:\\test_project\\components\\template.ts': {...}
}
在组件内会转换为
const serverFiles = {
  '/index.ts': {...},
  '/components/template.ts': {...},
}
回调中则会拼接回来
const handleSaveFile = (path: string, resolve: () => void, reject: (msg?: string) => void) => {
  console.log(path) // 这里会打印 'F:\\test_project\\index.ts'
}
*/
const files = ref<Files>()
const handleReload = (resolve: () => void, reject: (msg?: string) => void) => {
  server
    .fetchFiles()
    .then((response) => {
      files.value = response
      resolve()
    })
    .catch((e: Error) => {
      reject(e.message)
    })
}
const handleSaveFile = (path: string, content: string, resolve: () => void, reject: (msg?: string) => void) => {
  server
    .createOrSaveFile(path, content)
    .then((_response) => {
      resolve()
    })
    .catch((e: Error) => {
      reject(e.message)
    })
}
const handleDeleteFile = (path: string, resolve: () => void, reject: (msg?: string) => void) => {
  server
    .deleteFile(path)
    .then((_response) => {
      resolve()
    })
    .catch((e: Error) => {
      reject(e.message)
    })
}
const handleDeleteFolder = (path: string, resolve: () => void, reject: (msg?: string) => void) => {
  reject('不允许进行删除文件夹的操作！')
}
const handleNewFile = (path: string, resolve: Function, reject: Function) => {
  server
    .newFile(path)
    .then((_response) => {
      resolve()
    })
    .catch((e: Error) => {
      reject(e.message)
    })
}
const handleNewFolder = (path: string, resolve: Function, reject: Function) => {
  server
    .newFolder(path)
    .then((_response) => {
      resolve()
    })
    .catch((e: Error) => {
      reject(e.message)
    })
}
const handleRename = (path: string, newPath: string, resolve: () => void, reject: (msg?: string) => void) => {
  server
    .rename(path, newPath)
    .then((_response) => {
      resolve()
    })
    .catch((e: Error) => {
      reject(e.message)
    })
}
```

```vue
<template>
  <MonacoTreeEditor
    :font-size="14"
    :files="files"
    :sider-min-width="240"
    filelist-title="文件列表"
    @reload="handleReload"
    @new-file="handleNewFile"
    @new-folder="handleNewFolder"
    @save-file="handleSaveFile"
    @delete-file="handleDeleteFile"
    @delete-folder="handleDeleteFolder"
    @rename-file="handleRename"
    @rename-folder="handleRename"
  ></MonacoTreeEditor>
</template>
```

### 消息推送

```typescript
import { useMessage } from 'monaco-tree-editor'
import { onMounted } from 'vue'

const messageStore = useMessage()
onMounted(() => {
  const id = messageStore.action.info({
    content: '测试中..',
    loading: true,
  })
  setTimeout(() => {
    messageStore.action.close(id)
    messageStore.action.success({
      content: 'Hello Editor',
      closeable: true,
      timeoutMs: 15000,
      textTip: '测试成功!',
    })
  }, 5000)
})
```

### ~~快捷键~~

- 不推荐，这个 api 马上会被弃用
- 在未来的版本中，会有另一个新的交互式快捷键设置功能被内置

```typescript
import { useHotkey } from 'monaco-tree-editor'

const hotkeyStore = useHotkey()
// 焦点在根组件的时候触发
hotkeyStore.listen('root', (event: KeyboardEvent) => {})
// 焦点在编辑器内的时候触发
hotkeyStore.listen('editor', (event: KeyboardEvent) => {
  if (event.ctrlKey && !event.shiftKey && !event.altKey && (event.key === 's' || event.key === 'S')) {
    // do something...
  }
})
```

### 自定义菜单

```typescript
import { ref } from 'vue'
// ================ 自定义菜单 =================
/**
 * 自定义fileMenu和folderMenu将插入到树形结构的右键菜单中
 */
const fileMenu = ref([
  { label: 'Custom Selection 1', value: 'any type that not null' },
  { label: 'Custom Selection 2', value: 2 },
  { label: '自定义文件选项 3', value: { id: 3, decription: 'value可以是任意非空值' } },
])
const folderMenu = ref([{ label: 'backup', value: 'backupFolder' }])
/*
 * 点击左下角的设置图标后，将展示自定义菜单
 */
const settingsMenu = ref([
  {
    label: 'exit',
    handler: () => {
      alert('exit')
    },
  },
])
const handleContextMenuSelect = (path: string, item: { label: string | ComputedRef<string>; value: string }) => {
  console.warn('path: ' + path + '\nitem: ' + item)
}
```

```vue
<template>
  <MonacoTreeEditor
    :file-menu="fileMenu"
    :folder-menu="folderMenu"
    :settings-menu="settingsMenu"
    @contextmenu-select="handleContextMenuSelect"
  ></MonacoTreeEditor>
</template>
```

### 国际化 I18n

language 目前有 2 个可选值，`en-US`和`zh-CN`。

```vue
<!--
en-US: English (Default)
zh-CN: 简体中文
-->
<MonacoTreeEditor language="en-US"></MonacoTreeEditor>
```

### 主题

theme 目前有 2 个可选值，`dark`和`light`。

```vue
<!--
dark: 深色主题
light: 浅色主题
-->
<MonacoTreeEditor theme="dark"></MonacoTreeEditor>
```

### 自定义拖拽事件

```typescript
/*
 * 比如，当用户拖拽文件到编辑器中时，将文件导入到编辑器中
 */
const handleDragInEditor = (srcPath: string, targetPath: string, type: 'file' | 'folder') => {
  if (!targetPath.endsWith('.ts') && !srcPath.endsWith('.js')) {
    return
  }
  const editor = monacoStore.action.getEditor()
  const lineIndex = editor.getPosition()?.lineNumber!
  let str = 'import "' + _relativePathFrom(srcPath, targetPath) + '"'
  editor.executeEdits('drop', [{ range: new monaco.Range(lineIndex, 0, lineIndex, 0), text: str }])
}

function _longestCommonPrefix(strs: string[]): string {
  if (!strs.length) return ''
  let [a, ...b] = strs
  let result = ''
  for (let i = 0; i < a.length; i++) {
    let flag = b.every((item) => item[i] === a[i])
    if (flag) result += a[i]
    else break
  }
  return result
}

//计算相对路径
const _relativePathFrom = (returnPath: string, fromPath: string): string => {
  const prefix = _longestCommonPrefix([returnPath, fromPath])
  returnPath = returnPath.replace(prefix, '').replace(/\\/g, '/')
  fromPath = fromPath.replace(prefix, '').replace(/\\/g, '/')
  const fromPathArr = fromPath.split('/')
  let relativePath = ''
  if (fromPathArr.length === 1) {
    relativePath = './'
  } else {
    for (let i = fromPathArr.length - 2; i >= 0; i--) {
      relativePath += '../'
    }
  }
  return (relativePath += returnPath)
}
```

```vue
<template>
  <MonacoTreeEditor @drag-in-editor="handleDragInEditor"></MonacoTreeEditor>
</template>
```

## TODO 已知问题与待优化

[monaco-tree-editor/issues](https://github.com/AlphaFoxz/monaco-tree-editor/issues)
