<script setup lang="ts">
import { NAlert, NButton, NConfigProvider, NH1, NInput, NInputGroup, NList, NListItem, NP, NSelect, NSpace, NUpload, NUploadDragger, darkTheme, dateZhCN, lightTheme, useOsTheme, zhCN, type UploadFileInfo, NA } from 'naive-ui';
import { computed, ref } from 'vue';

const currentLang = ref({ 'lang': zhCN, 'dateLang': dateZhCN })


const osTheme = useOsTheme()
const theme = computed(() => (osTheme.value === 'dark' ? darkTheme : lightTheme))

async function downloadFileWithDelay(url: string, filename: string): Promise<void> {
  return new Promise(async (resolve) => {
    try {
      // 1. 获取文件数据
      const response = await fetch(url)
      const blob = await response.blob()

      // 2. 创建Blob URL
      const blobUrl = URL.createObjectURL(blob)

      // 3. 创建下载链接
      const a = document.createElement('a')
      a.href = blobUrl
      a.download = filename + '.' + audio_format.value // 添加文件扩展名
      a.style.display = 'none'
      document.body.appendChild(a)
      a.click()
      document.body.removeChild(a)

      // 4. 清理
      setTimeout(() => {
        URL.revokeObjectURL(blobUrl)
        resolve()
      }, 100)

    } catch (error) {
      console.error('下载失败:', error)
      resolve() // 即使出错也继续下一个
    }
  })
}

const text_val = ref<string>("")

type Audio_format = "mp3" | "wav"
const audio_format = ref<Audio_format>("wav")

const res_audio_url_list = ref<string[]>([])

const url_ill_regex = /[!#$&*+/=@\[\]]/g

function text_to_pages(text: string, one_page_size: number): string[] {
  const res: string[] = []

  let current_text = ""
  let next_text_slice = ""
  for (
    const v of text
      .replace(/\n+/g, '\n') // 换行符 复数个 -> 单个
      .replace(url_ill_regex, '') // 移除非法字符
      .replace(/,/g, '，')
      .replace(/\?/g, '？')
      .replace(/:/g, '：')
      .replace(/;/g, '；')
      .replace(/\(/g, '（')
      .replace(/\)/g, '）')
      .split("\n")
  ) {
    next_text_slice = v + "。"

    if (current_text.length + next_text_slice.length > one_page_size) {
      res.push(current_text)
      current_text = ""
    }

    current_text += next_text_slice
  }
  res.push(current_text)

  return res
}

interface Api_res {
  /** 状态码，200表示成功 */
  code: number;
  /** 消息描述 */
  msg: string;
  /** 生成的音频文件URL */
  url: string;
  /** API 来源信息 */
  api_source: string;
}
async function api_submit() {
  for (const text of text_to_pages(text_val.value, 100)) {
    const url = `https://api.milorapart.top/apis/mbAIsc?format=${audio_format.value}&text=${text}`
    console.debug(api_submit.name, url, text, text.length)
    const res = await fetch(url)
    res.json()
      .then((data: Api_res) => {
        if (data.code !== 200) {
          console.error("faild", url, data)
        }
        else {
          console.debug("success", url, data)
          res_audio_url_list.value.push(data.url)
        }
      })
      .catch(err => console.error("catch error", err))
  }
}
</script>

<template>
  <div id="top-box">
    <n-config-provider :theme="theme" :locale="currentLang.lang" :date-locale="currentLang.dateLang">
      <div id="pack-box">
        <n-space vertical>

          <n-h1 style="text-align: center;">文本转曼波配音</n-h1>

          <n-alert v-show="new RegExp(url_ill_regex.source, url_ill_regex.flags).test(text_val)" title="存在非法字符"
            type="warning">
            存在非法字符
          </n-alert>

          <n-upload multiple directory-dnd @update:file-list="(fileList: UploadFileInfo[]) => {
            text_val = ''
            fileList.forEach(file_info => {
              const f = file_info.file
              if (f)
                f.text().then(v => text_val += v)
            })
          }">
            <n-upload-dragger>
              <n-p>上传文本文件自动拼接输入</n-p>
            </n-upload-dragger>
          </n-upload>

          <n-input id="input-area" v-model:value="text_val" type="textarea" placeholder="输入文本" />

          <n-input-group>
            <n-select v-model:value="audio_format" :options="[
              {
                label: 'WAV',
                value: 'wav',
              },
              {
                label: 'MP3',
                value: 'mp3',
              },
            ]" />
            <n-button @click="api_submit">提交</n-button>
          </n-input-group>

          <n-list>
            <n-list-item v-for="url, i in res_audio_url_list">
              <n-a :href="url">{{ url }}</n-a>
            </n-list-item>
          </n-list>

          <n-button v-show="res_audio_url_list.length" @click="async () => {
            for (let i = 0; i < res_audio_url_list.length; i++) {
              const url = res_audio_url_list[i] as string
              await downloadFileWithDelay(url,
                i.toString().padStart(res_audio_url_list.length.toString().length, '0')
              )
            }
          }
          ">全部下载</n-button>

        </n-space>
      </div>
    </n-config-provider>
  </div>
</template>

<style scoped>
#top-box {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background-color: v-bind('theme.common.bodyColor');
}

#pack-box {
  height: 100vh;
  width: 40vw;
  min-width: 20rem;
  margin: 0 auto;
  padding: 0;

  display: flex;
  flex-direction: column;
  justify-content: center;
}

#input-area {
  & * {
    max-height: 50vh;
  }
}
</style>
