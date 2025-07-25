<template>
  <template v-if="state.index">
    <Transition name="fade">
      <div
        class="bg"
        v-show="state.preview"
      >
        <Transition name="fade">
          <div
            class="green"
            v-show="setting.green"
          ></div>
        </Transition>
        <div
          class="green-btn"
          title="切换背景"
          @click.stop="toggleGreenScreen"
          :class="{ highlight: setting.green }"
        >
          <Icon name="green" />
        </div>
      </div>
    </Transition>
    <Transition
      :name="setting.green ? 'preview-delay' : 'preview'"
      appear
    >
      <MessageBox
        v-show="state.preview"
        class="message-preview"
        :index="messageIndex"
        :title="title"
        :info="info"
        :playing="autoPlay.flag"
        preview
        @click.stop="autoPlay.flag ? stopPlay() : undefined"
        ref="boxRef"
      >
        <MessageItem
          v-for="(element, index) in dataList"
          :key="`preview-${index}`"
          :item="element"
          :index="0"
          preview
        />
        <div ref="endOfMessageList"></div>
        <template #bottom>
          <Transition
            name="option"
            @before-enter="beforeOptionShow"
            @after-enter="afterOptionShow"
            @after-leave="afterOptionHide"
          >
            <div
              class="option-box"
              @click.stop
              v-if="autoPlay.flag && autoPlay.option.length > 0"
            >
              <div
                class="option"
                v-for="(item, key) in autoPlay.option"
                :key="key"
                @click.stop="handleOptionClick(item)"
              >
                {{ item.text }}
              </div>
            </div>
          </Transition>
        </template>
      </MessageBox>
    </Transition>
  </template>
</template>

<script lang="ts" setup>
import { emitter } from '@/assets/scripts/event'
import { popupManager } from '@/assets/scripts/popup'
import { autoPlay } from '@/store/autoPlay'
import { currentMessage, messageIndex } from '@/store/message'
import { KEY, setting, state } from '@/store/setting'
import { screenshot } from '@/utils/screenshot'
import Icon from './Common/Icon.vue'
import { info, scrollToBottom, title } from './Message/Message'
import MessageBox from './Message/MessageBox.vue'
import MessageItem from './Message/MessageItem.vue'

const boxRef = ref<InstanceType<typeof MessageBox>>()
const endOfMessageList = ref()

// Preload all mp3 files from src/assets/sound
const sounds = import.meta.glob('/src/assets/sound/*.mp3', { eager: true, as: 'url' })

const audioCache = new Map<string, HTMLAudioElement>()

function playSound(key: string): void {
  const path = `/src/assets/sound/${key}.mp3`
  const src = sounds[path]

  if (!src) {
    console.warn(`Sound "${key}" not found at ${path}`)
    return
  }

  let audio = audioCache.get(key)
  if (!audio) {
    audio = new Audio(src)
    audioCache.set(key, audio)
  }

  audio.currentTime = 0
  audio.play().catch((err) => {
    console.error(`Failed to play sound "${key}":`, err)
  })
}

// 要显示的数据
const dataList = computed(() => {
  if (!currentMessage.value) return []
  if (autoPlay.flag || autoPlay.list.length > 0) return autoPlay.list

  const list: (Message & { default?: [boolean] })[] = []
  currentMessage.value.list.forEach((item) => {
    if (item.option) {
      if (!list[list.length - 1] || !list[list.length - 1].default) {
        list.push({
          ...item,
          default: [item.option[0]],
          option: undefined
        })
      } else {
        if (!list[list.length - 1].default?.[0] && item.option[0]) {
          list[list.length - 1].text = item.text
          list[list.length - 1].default = [item.option[0]]
        }
      }
    } else {
      list.push(item)
    }
  })
  return list
})

let optionIndex = 0
let autoPlayIndex = -1

const reset = () => {
  optionIndex = 0
  autoPlayIndex = -1
  autoPlay.list = []
  autoPlay.option = []
}

let timer: number
emitter.on('autoplay', () => {
  if (autoPlay.flag) return

  state.preview = true
  reset()
  autoPlay.flag = true

  window.setTimeout(() => {
    playSound('message-start')
  }, 500)
  timer = window.setTimeout(() => {
    nextTick(() => {
      next(0, true)
    })
  }, 1000)
})

let shouldRefresh: boolean = false
let selectOption: Message | undefined

const refresh = () => {
  scrollToBottom(boxRef.value?.listDom, false)
  if (shouldRefresh) {
    requestAnimationFrame(refresh)
  }
}

const beforeOptionShow = () => {
  shouldRefresh = true
  selectOption = undefined
  requestAnimationFrame(refresh)
}

const afterOptionShow = () => {
  shouldRefresh = false
}

const afterOptionHide = () => {
  if (!autoPlay.flag) return
  if (selectOption) {
    autoPlay.list.push({ ...selectOption, option: undefined })
  }
  nextTick(() => {
    selectOption = undefined
    autoPlayIndex = optionIndex - 1
    scrollToBottom(boxRef.value?.listDom)

    clearTimeout(timer)
    timer = window.setTimeout(() => {
      next(optionIndex, true)
    }, 1500)
  })
}

const handleOptionClick = (item: Message) => {
  autoPlay.option = []
  selectOption = item
}

const next = (i: number, loading: boolean) => {
  if (!autoPlay.flag) return
  if (!currentMessage.value) return

  if (!currentMessage.value.list[i]) {
    autoPlay.flag = false
    return
  }

  if (currentMessage.value.list[i].option) {
    optionIndex = i + 1
    autoPlay.option.push(currentMessage.value.list[i])
    if (currentMessage.value.list[optionIndex] && currentMessage.value.list[optionIndex].option) {
      next(optionIndex, true)
    }
    return
  }
  autoPlayIndex = i

  if (currentMessage.value.list[i].key === '开拓者' || currentMessage.value.list[i].notice) {
    loading = false
    autoPlay.list.push(currentMessage.value.list[i])
  } else {
    if (loading) {
      autoPlay.list.push({
        ...currentMessage.value.list[i],
        loading: true
      })
    } else {
      autoPlay.list[autoPlay.list.length - 1].loading = false
    }
  }

  nextTick(() => {
    if (!currentMessage.value) return

    endOfMessageList.value?.scrollIntoView({ behavior: 'smooth' })

    if (loading) {
      const time = Math.max(currentMessage.value.list[i].text.length * 100, 1000)

      clearTimeout(timer)
      timer = window.setTimeout(() => {
        playSound('message-send')
        next(i, false)
      }, time)
    } else {
      if (currentMessage.value.list[i + 1]) {
        let time = currentMessage.value.list[i + 1].interval
        if (time === undefined || time < 1000) {
          time = currentMessage.value.list[i + 1].key === '开拓者' ? 2000 : 1500
        }
        clearTimeout(timer)
        timer = window.setTimeout(() => {
          next(i + 1, true)
        }, time)
      } else {
        autoPlay.flag = false
      }
    }
  })
}

// 取消播放
const stopPlay = () => {
  clearTimeout(timer)
  if (!currentMessage.value) return

  autoPlay.flag = false
  const list: (Message & { default?: [boolean] })[] = []
  for (let j = autoPlayIndex + 1; j < currentMessage.value.list.length; j++) {
    if (currentMessage.value.list[j].option) {
      if (!list[list.length - 1] || !list[list.length - 1].default) {
        list.push({
          ...currentMessage.value.list[j],
          default: [!!currentMessage.value.list[j].option?.[0]],
          option: undefined
        })
      } else {
        if (!list[list.length - 1].default?.[0] && currentMessage.value.list[j].option?.[0]) {
          list[list.length - 1].text = currentMessage.value.list[j].text
          list[list.length - 1].default = [!!currentMessage.value.list[j].option?.[0]]
        }
      }
    } else {
      list.push(currentMessage.value.list[j])
    }
  }
  autoPlay.list = [...autoPlay.list, ...list]
  scrollToBottom(boxRef.value?.listDom)
}
emitter.on('stopplay', stopPlay)

emitter.on('screenshot', () => {
  if (popupManager.isLoading()) return
  reset()

  state.preview = true
  popupManager.open('loading')
  nextTick(() => {
    if (boxRef.value?.boxDom && boxRef.value?.listDom && state.preview) {
      screenshot(
        boxRef.value.boxDom,
        {
          name: title.value,
          height: boxRef.value.listDom.scrollHeight + 185,
          download: setting.download,
          data: {
            raw: JSON.stringify(toRaw(currentMessage.value)),
            filename: KEY.RAW_NAME
          }
        },
        { pixelRatio: setting.quality }
      )
        .catch(() => {
          popupManager.open('confirm', {
            title: '图片保存异常',
            text: ['可能是浏览器拦截了新窗口'],
            tip: '请尝试在设置中切换下载模式'
          })
        })
        .finally(() => {
          window.setTimeout(() => {
            popupManager.close('loading')
          }, 1000)
        })
    }
  })
})

const toggleGreenScreen = () => {
  setting.green = !setting.green
}

onUnmounted(() => {
  emitter.off('autoplay')
  emitter.off('stopplay')
  emitter.off('screenshot')
})
</script>

<style lang="stylus" scoped>
@import './Message/Message.styl'
@import './Common/Window.styl'

.bg
  position absolute
  z-index 9
  width 100%
  height 100%
  background rgba(0, 0, 0, 0.6)
  box-shadow 0 0 20px 0px rgba(0, 0, 0, 0.5)
  backdrop-filter blur(20px)
  -webkit-backdrop-filter blur(20px)

  .green
    position fixed
    top 0
    width 100%
    height 105%
    background-color green
    transition 0.2s

  .green-btn
    position absolute
    right 80px
    bottom -75px
    display flex
    justify-content center
    align-items center
    border-radius 15px
    background #666
    cursor pointer
    transition background 0.2s
    user-select none

    &:hover
      &:before
        position absolute
        top -10px
        right -10px
        bottom -10px
        left -10px
        border 5px solid #666
        border-radius 10px
        content ''

  .highlight
    background none

.message-preview
  position absolute
  top 12%
  left 900px
  z-index 10
  width 1400px
  height 80%
  message()
  scale 1.2

  :deep(*)
    cursor auto !important

  .option-box
    z-index 9
    overflow-y auto
    padding 20px 50px
    height 90px * 3 + 20px * 4
    border-top var(--menu-border-hover)
    background var(--message-menu-background-color)
    scrollbar-width none

    &::-webkit-scrollbar
      width 0
      height 0

    .option
      display flex
      justify-content center
      align-items center
      option()
      margin 20px 0
      cursor pointer !important
      user-select none

.option-enter-active
  transition height 0.25s, opacity 0.75s

.option-leave-active
  transition all 0.15s

.option-enter-from
.option-leave-to
  height 0 !important
  opacity 0 !important

.preview-enter-active
  transition all 0.3s

.preview-delay-enter-active
  transition all 0.3s
  transition-delay 0.5s

.preview-enter-from
.preview-delay-enter-from
  opacity 0
  transform scaleY(0)
</style>
