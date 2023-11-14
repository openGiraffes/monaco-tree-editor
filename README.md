# monaco-tree-editor

这个我的第一个 npm 库。**_抱歉。这个组件现在还不能用。_** 我会尽可能快地搞定它，并提供一个预发布版本。

It's my first npm repo. **_Sorry. It still not work now._** I'll try fast that i can to make it and publish a rc-version

# 这个库做了什么 What this repo did?

- [x] 提供 VSCode 风格的文件树 Provide FileTree with VSCode style.
- [x] 回调函数采用异步处理 async callback functions
- [ ] 支持自定义的热键处理 hook（开发中） A hook for Hotkey (developing)
- [x] 提供一个浮动的全局消息 hook。 A hook for global float message box.

## 前置要求 Prerequisites

- vue3 (推荐版本 recommend v.3.3.4+)

## 如何安装 How to install

```shell
pnpm add monaco-tree-editor
```

## 示例代码 Demo Code (Beta)

```vue
<script setup lang="ts">
import Editor from 'monaco-tree-editor/src/components/monaco-tree-editor/Index.vue'
import { useMessage } from 'monaco-tree-editor/src/components/monaco-tree-editor/message-store'
import { useHotkey } from 'monaco-tree-editor/src/components/monaco-tree-editor/hotkey-store'
import { useMonaco } from 'monaco-tree-editor/src/components/monaco-tree-editor/monaco-store'
import { type Files } from 'monaco-tree-editor/src/components/monaco-tree-editor/define'
import { onMounted, ref } from 'vue'

// ================ 调整大小 resize ================
const editorRef = ref()
onMounted(() => {
  window.onresize = () => {
    setTimeout(() => {
      editorRef.value.resize()
    }, 30)
  }
})

// ================ 推送消息 push message ================
const messageStore = useMessage()
onMounted(() => {
  const id = messageStore.info({
    content: 'testing..',
    loading: true,
  })
  setTimeout(() => {
    messageStore.close(id)
    messageStore.success({
      content: 'Hello Editor',
      closeable: true,
      timeoutMs: 15000,
      textTip: 'testing successed!',
    })
  }, 5000)
})

// ================ 原生功能 original modules of monaco-editor ================
const monacoStore = useMonaco()
monacoStore.monaco.languages.register
onMounted(() => {
  monacoStore.getEditor().setValue
})

// ================ 快捷键 hotkey ==================
const hotkeyStore = useHotkey()
hotkeyStore.listen('root', (event: KeyboardEvent) => {})
hotkeyStore.listen('editor', (event: KeyboardEvent) => {
  if (event.ctrlKey && !event.shiftKey && !event.altKey && event.key === 's') {
    // do something...
  }
})

// ================ 模拟服务端 mock server ================
namespace server {
  let fileSeparator = '\\'
  let responseFiles: Files = {
    'F:\\test_project\\components': {
      isDirectory: true,
    },
    'F:\\test_project\\index.ts': {
      isFile: true,
      content: 'console.log("hello world")',
    },
  }
  export const fetchFiles = async () => {
    return await JSON.parse(JSON.stringify(responseFiles))
  }
  export const createOrSaveFile = async (path: string, content: string) => {
    if (responseFiles[path]) {
      if (!responseFiles[path].isFile) {
        throw new Error(`[ ${path} ] is not a file!`)
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
    if (responseFiles[path]) {
      throw new Error(`[ ${path} ] already exists!`)
    }
    responseFiles[path] = {
      isFile: true,
      content: '',
    }
  }
  export const newFolder = async (path: string) => {
    if (responseFiles[path]) {
      throw new Error(`[ ${path} ] already exists!`)
    }
    responseFiles[path] = {
      isDirectory: true,
    }
  }
  export const rename = async (path: string, newName: string) => {
    if (!responseFiles) {
      throw new Error(`[ ${path} ] not exists!`)
    }
    let tmp = path.split(fileSeparator)
    tmp[tmp.length - 1] = newName
    responseFiles[tmp.join(fileSeparator)] = responseFiles[path]
    delete responseFiles[path]
    return true
  }
  export const deleteFile = async (path: string) => {
    delete responseFiles[path]
    return true
  }
}

// ================ 加载文件 load files ================
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

// ================ 回调函数 callback =================
/*
不论服务端的文件名是什么，组件内会统一取最长公共前缀，而在回调方法中的路径则会拼接回原路径
Whaterver the server's file name is,
the component will take the longest common prefix,
and the path in the callback method will be concatenated with the original path
例如 For example:
const serverFiles = {
  'F:\\test_project\\index.ts': {...},
  'F:\\test_project\\components\\template.ts': {...}
}
在组件内会转换为 In component, it will be converted to:
const serverFiles = {
  '/index.ts': {...},
  '/components/template.ts': {...},
}
回调中会拼接回来 In your callback functions:
const handleSaveFile = (path: string, resolve: () => void, reject: (msg?: string) => void) => {
  console.log(path) // will print 'F:\\test_project\\index.ts'
}
*/
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
  reject('Operation of delete folder is not supported!')
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
const handleRename = (path: string, name: string, resolve: () => void, reject: (msg?: string) => void) => {
  server
    .rename(path, name)
    .then((_response) => {
      resolve()
    })
    .catch((e: Error) => {
      reject(e.message)
    })
}
</script>

<template>
  <Editor
    :files="files"
    @reload="handleReload"
    @new-file="handleNewFile"
    @new-folder="handleNewFolder"
    @save-file="handleSaveFile"
    @delete-file="handleDeleteFile"
    @delete-folder="handleDeleteFolder"
    @rename-file="handleRename"
    @rename-folder="handleRename"
    ref="editorRef"
  ></Editor>
</template>
```
