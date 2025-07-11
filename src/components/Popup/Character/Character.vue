<template>
  <Popup
    :index="props.index"
    show
  >
    <div
      class="character-select"
      @contextmenu.prevent.stop="close"
    >
      <div
        class="box"
        @click.stop=""
      >
        <div class="tabs">
          <div
            class="tab"
            :class="{ highlight: page === 0 }"
            @click="changePage(0)"
          >
            游戏角色
          </div>
          <div
            class="tab"
            :class="{ highlight: page === 1 }"
            @click="changePage(1)"
          >
            其它角色
          </div>
          <div
            class="tab"
            :class="{ highlight: page === 2 }"
            @click="changePage(2)"
          >
            自定义角色
          </div>
        </div>
        <input />
        <div class="list">
          <div
            class="character-list"
            data-type="game"
            ref="game"
          >
            <CharacterCard
              class="character game-char"
              :custom="false"
              :name="setting.name"
              :avatar="userData.avatar"
              :level="5"
              title="开拓者"
              @click="handlecharacterClick('开拓者', '')"
            />
            <CharacterCard
              v-for="(item, key) in character.game"
              :key="`character-${key}`"
              class="character game-char"
              :name="key as string"
              :info="item.info"
              :avatar="item.avatar"
              @click="handlecharacterClick(String(key), item.name)"
            />
          </div>
          <div
            class="character-list"
            data-type="other"
            ref="other"
          >
            <CharacterCard
              v-for="(item, key) in character.other"
              :key="`other-character-${key}`"
              class="character"
              :custom="true"
              :name="item.name"
              :info="item.info"
              :avatar="item.avatar"
              :level="item.gold ? 5 : undefined"
              @click="handlecharacterClick(String(key), item.name)"
            />
          </div>

          <div
            class="character-list"
            data-type="custom"
            ref="custom"
          >
            <CharacterCard
              v-for="(item, key) in character.custom"
              :key="`custom-character-${key}`"
              class="character"
              :custom="true"
              :name="item.name"
              :info="item.info"
              :avatar="item.avatar"
              @click="handlecharacterClick(String(key), item.name)"
              @contextmenu.prevent.stop="handleDelClick(String(key), item.name)"
            >
              <div
                class="del"
                @click.stop="handleDelClick(String(key), item.name)"
              >
                <Icon
                  name="delete"
                  width="25"
                  height="25"
                />
              </div>
            </CharacterCard>
            <div
              class="add"
              @click="addCustom"
            >
              <Icon name="add" />
            </div>
          </div>
        </div>
      </div>
    </div>
  </Popup>
</template>

<script lang="ts" setup>
import { getAvatarBase64 } from '@/assets/scripts/avatar'
import { popupManager } from '@/assets/scripts/popup'
import CharacterCard from '@/components/Character/CharacterCard.vue'
import Icon from '@/components/Common/Icon.vue'
import { data as avatarData } from '@/components/Popup/Avatar/data'
import { userData } from '@/store/avatar'
import { character } from '@/store/character'
import { input } from '@/store/input'
import { message } from '@/store/message'
import { setting } from '@/store/setting'

import { Popup } from '@/components/Common/Popup'

import { data } from './data'

const game = ref<HTMLElement | null>(null)
const other = ref<HTMLElement | null>(null)
const custom = ref<HTMLElement | null>(null)

const props = defineProps<{
  name: string
  index: number
}>()

const emits = defineEmits<{
  (event: 'close', name: string): void
}>()

const close = () => {
  emits('close', props.name)
}

const page = ref(0)

const intersectionRatio: [number, number, number] = [0, 0, 0]

const intersectionObserver = new IntersectionObserver((entries) => {
  entries.forEach((item) => {
    switch ((item.target as HTMLElement).dataset.type) {
      case 'game':
        intersectionRatio[0] = item.intersectionRatio
        break
      case 'other':
        intersectionRatio[1] = item.intersectionRatio
        break
      case 'custom':
        intersectionRatio[2] = item.intersectionRatio
        break
    }
  })

  if (intersectionRatio[2] > 0) {
    page.value = 2
    return
  }
  if (intersectionRatio[1] > 0) {
    page.value = 1
    return
  }
  page.value = 0
})

onMounted(() => {
  if (game.value) intersectionObserver.observe(game.value)
  if (other.value) intersectionObserver.observe(other.value)
  if (custom.value) intersectionObserver.observe(custom.value)
})

const changePage = (page: number) => {
  if (page === 0) {
    game.value?.scrollIntoView({
      behavior: 'smooth'
    })
    return
  }

  if (page === 1) {
    other.value?.scrollIntoView({
      behavior: 'smooth'
    })
    return
  }

  if (page === 2) {
    custom.value?.scrollIntoView({
      behavior: 'smooth'
    })
    return
  }
}

const handlecharacterClick = async (key: string, name: string) => {
  if (data.key) {
    if (data.key[0] === -1) {
      if (key !== '开拓者') {
        avatarData.index = key
        avatarData.name = name
      } else {
        avatarData.index = setting.avatar
        avatarData.name = setting.name
      }
    } else {
      message.list[data.key[0]].list[data.key[1]].key = key
      message.list[data.key[0]].list[data.key[1]].name = name
      message.list[data.key[0]].list[data.key[1]].avatar = await getAvatarBase64(key)
    }
  } else {
    input.character.key = key
    input.character.name = name
  }
  close()
}

const addCustom = async () => {
  const name = await popupManager.open('input', { title: '请输入角色名' })
  if (name === null) return

  const info =
    (await popupManager.open('input', {
      title: '请输入角色签名',
      tip: '非必要选项',
      required: false
    })) ?? ''
  const key = Date.now()

  popupManager
    .open('cropper', {
      aspectRatio: 1,
      maxWidth: 500
    })
    .then(({ base64 }) => {
      character.custom[key] = {
        name,
        avatar: base64,
        info,
        custom: true
      }
    })
}

const handleDelClick = (key: string, name: string) => {
  popupManager.open('confirm', {
    title: '删除角色',
    text: [`是否删除${name}？`],
    fn: () => {
      delete character.custom[key]
    }
  })
}
</script>

<style lang="stylus" scoped>
@import '../../Common/Window.styl'

$character-item-width = 200px

.character-select
  position absolute
  top 0
  left 0
  display flex
  justify-content center
  align-items center
  width 100%
  height 100%
  user-select none

  .box
    position relative
    display flex
    flex-direction column
    justify-content center
    align-items center
    padding 20px 65px
    width 90%
    height 95%
    background var(--box-background-color)
    cursor default
    message()

    .tabs
      display flex
      align-self flex-start
      width 100%

      .tab
        margin 0 20px 25px 5px
        font-weight bold
        font-size 60px
        opacity 0.5
        cursor pointer

        &:hover
          opacity 1

    input
      flex 1
      box-sizing border-box
      padding 0 50px 0 90px
      width 100%
      height 100px
      outline none
      border none
      background #e8e8e8
      color #121212
      text-align center
      font-size 48px
      transition box-shadow 0.2s

      &:focus
      &:hover
        box-shadow 5px 5px 15px #aaa

    .list
      overflow overlay
      margin 15px 0
      padding 0 30px 0 20px
      width 100%
      height 100%

      .character-list
        display flex
        flex-wrap wrap

        .character
          width $character-item-width

          &:hover
            .del
              opacity 1

        .add
          display flex
          flex-direction column
          justify-content center
          align-items center
          box-sizing border-box
          margin 10px
          width $character-item-width
          height 250px
          border 5px solid #afafaf
          color #afafaf
          cursor pointer

.highlight
  position relative
  opacity 1 !important

  &:after
    position absolute
    bottom -10px
    left 0
    width 100%
    height 5px
    background #121212
    content ''
    animation open 0.3s forwards

.del
  position absolute
  top 0
  right 0
  display flex
  justify-content center
  align-items center
  width 80px
  height 80px
  font-size 50px
  opacity 0
  cursor pointer

  :deep(path)
    fill #fff

  &:hover
    opacity 1
</style>
