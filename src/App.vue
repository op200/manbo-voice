<script setup lang="ts">
import { LogoGithub } from '@vicons/ionicons5';
import { NA, NAlert, NButton, NConfigProvider, NDropdown, NH1, NIcon, NInput, NInputGroup, NP, NProgress, NSelect, NSpace, NUpload, NUploadDragger, darkTheme, dateZhCN, lightTheme, useOsTheme, zhCN, type UploadFileInfo, NForm } from 'naive-ui';
import { computed, h, ref } from 'vue';

const home_link = 'https://github.com/op200/manbo-voice'

const currentLang = ref({ 'lang': zhCN, 'dateLang': dateZhCN })

const text_to_pages_default_len = 100

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

const pages = ref<string[]>([])
const res_audio_url_list = ref<string[]>([])
const is_in_get_res = ref<boolean>(false)

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
  is_in_get_res.value = true
  for (const [i, text] of pages.value.entries()) {
    const url = `https://api.milorapart.top/apis/mbAIsc?format=${audio_format.value}&text=${text}`
    console.debug(api_submit.name, url, text, text.length)
    let is_not_break: boolean = true
    while (is_not_break) {
      try {
        const res = await fetch(url)
        const data: Api_res = await res.json()
        if (data.code !== 200) {
          console.error("faild", url, data)
        }
        else {
          console.debug("success", url, data)
          res_audio_url_list.value.push(data.url)
          is_not_break = false
        }
      } catch (err) {
        console.error("catch error", err)
      }
    }
  }
  is_in_get_res.value = false
}
</script>

<template>
  <div id="top-box">
    <n-config-provider :theme="theme" :locale="currentLang.lang" :date-locale="currentLang.dateLang">
      <div id="pack-box">
        <n-space vertical>


          <n-dropdown trigger="hover" placement="top" :options="[
            {
              label: () => h(NA, { href: home_link }, { default: () => 'Github' }),
              icon: () => h(NIcon, null, { default: () => h(LogoGithub) }),
            },
          ]">
            <n-h1 style="text-align: center;cursor: help;">文本转曼波配音</n-h1>
          </n-dropdown>

          <n-alert v-show="new RegExp(url_ill_regex.source, url_ill_regex.flags).test(text_val)" title="存在非法字符"
            type="warning">
            存在非法字符
          </n-alert>

          <n-upload :disabled="is_in_get_res" multiple directory-dnd @update:file-list="(fileList: UploadFileInfo[]) => {
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

          <n-input :disabled="is_in_get_res" id="input-area" v-model:value="text_val" type="textarea"
            placeholder="输入文本" />

          <n-input-group>
            <n-select :disabled="is_in_get_res" v-model:value="audio_format" :options="[
              {
                label: 'WAV',
                value: 'wav',
              },
              {
                label: 'MP3',
                value: 'mp3',
              },
            ]" />
            <n-button :disabled="is_in_get_res" @click="() => {
              pages = text_to_pages(text_val, text_to_pages_default_len)
              api_submit()
            }">提交</n-button>
          </n-input-group>

          <n-space vertical v-show="is_in_get_res || res_audio_url_list.length">
            <n-progress type="line" :percentage="res_audio_url_list.length / pages.length * 100"
              indicator-placement="inside" :processing="pages.length !== res_audio_url_list.length" />
            <n-space :justify="'center'">
              <n-dropdown trigger="hover" :options="res_audio_url_list.map(url => ({
                label: () => h(NA, { href: url }, { default: () => url.replace(/^https:\/\/api\.milorapart\.top\/voice\//, '') })
              }))">
                <n-button :disabled="pages.length !== res_audio_url_list.length" @click="async () => {
                  for (let i = 0; i < res_audio_url_list.length; i++) {
                    const url = res_audio_url_list[i] as string
                    await downloadFileWithDelay(url,
                      i.toString().padStart(res_audio_url_list.length.toString().length, '0')
                    )
                  }
                }
                ">{{ res_audio_url_list.length }} / {{ pages.length }}
                  全部下载</n-button>
              </n-dropdown>
            </n-space>
          </n-space>

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
