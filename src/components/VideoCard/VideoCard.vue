<script lang="ts" setup>
import Button from '~/components/Button.vue'
import { useApiClient } from '~/composables/api'
import { settings } from '~/logic'
import type { VideoPreviewResult } from '~/models/video/videoPreview'
import { calcCurrentTime, calcTimeSince, numFormatter } from '~/utils/dataFormatter'
import { getCSRF, removeHttpFromUrl } from '~/utils/main'

import Tooltip from '../Tooltip.vue'

interface Props {
  id: number
  duration?: number
  durationStr?: string
  title: string
  desc?: string
  cover: string
  author?: string
  authorFace?: string
  /** If you set the `authorUrl`, clicking the author's name or avatar will navigate to this url  */
  authorUrl?: string
  mid?: number
  view?: number
  viewStr?: string
  danmaku?: number
  danmakuStr?: string
  publishedTimestamp?: number
  capsuleText?: string
  bvid?: string
  aid?: number
  uri?: string
  /** If you want to show preview video, you should set the cid value */
  cid?: number
  epid?: number
  followed?: boolean
  horizontal?: boolean
  tag?: string
  rank?: number
  topRightContent?: boolean
  showPreview?: boolean
  moreBtn?: boolean
  moreBtnActive?: boolean
  removed?: boolean
  type?: 'horizontal' | 'vertical' | 'bangumi'
}

const props = withDefaults(defineProps<Props>(), {
  topRightContent: true,
  type: 'horizontal',
})

const emit = defineEmits<{
  (e: 'moreClick', event: MouseEvent): MouseEvent
  (e: 'undo'): void
  (e: 'tellUsWhy'): void
}>()

const api = useApiClient()

// Used to click and control herf attribute
const isClick = ref<boolean>(false)

const videoUrl = computed(() => {
  if (!isClick.value)
    return undefined
  if (props.bvid || props.aid)
    return `https://www.bilibili.com/video/${props.bvid ?? `av${props.aid}`}`
  else if (props.epid)
    return `https://www.bilibili.com/bangumi/play/ep${props.epid}`
  else if (props.uri)
    return props.uri
  else
    return ''
})

const authorJumpUrl = computed(() => {
  if (props.authorUrl)
    return props.authorUrl
  else if (props.mid)
    return `//space.bilibili.com/${props.mid}`
  else
    return ''
})

const wValue = computed((): string => {
  if (props.horizontal)
    return 'xl:280px lg:250px md:200px 200px'
  else
    return 'w-full'
})

const isInWatchLater = ref<boolean>(false)
const isHover = ref<boolean>(false)
const contentVisibility = ref<'auto' | 'visible'>('auto')
const mouseEnterTimeOut = ref()
const mouseLeaveTimeOut = ref()
const previewVideoUrl = ref<string>('')

watch(() => isHover.value, (newValue) => {
  if (props.showPreview && settings.value.enableVideoPreview) {
    if (newValue && !previewVideoUrl.value && props.cid) {
      api.video.getVideoPreview({
        bvid: props.bvid,
        cid: props.cid,
      }).then((res: VideoPreviewResult) => {
        if (res.code === 0)
          previewVideoUrl.value = res.data.durl[0].url
      })
    }
  }
})

function toggleWatchLater() {
  if (!isInWatchLater.value) {
    api.watchlater.saveToWatchLater({
      aid: props.id,
      csrf: getCSRF(),
    })
      .then((res) => {
        if (res.code === 0)
          isInWatchLater.value = true
      })
  }
  else {
    api.watchlater.removeFromWatchLater({
      aid: props.id,
      csrf: getCSRF(),
    })
      .then((res) => {
        if (res.code === 0)
          isInWatchLater.value = false
      })
  }
}

function handleMouseEnter() {
  if (settings.value.hoverVideoCardDelayed) {
    mouseEnterTimeOut.value = setTimeout(() => {
      isHover.value = true
      clearTimeout(mouseLeaveTimeOut.value)
      contentVisibility.value = 'visible'
    }, 1200)
  }
  else {
    isHover.value = true
    clearTimeout(mouseLeaveTimeOut.value)
    contentVisibility.value = 'visible'
  }
}

function handelMouseLeave() {
  isHover.value = false
  clearTimeout(mouseEnterTimeOut.value)
  clearTimeout(mouseLeaveTimeOut.value)
  mouseLeaveTimeOut.value = setTimeout(() => {
    contentVisibility.value = 'auto'
  }, 300)
}

function switchClickState(flag: boolean) {
  if (flag) {
    isClick.value = flag
  }
  else {
    setTimeout(() => {
      isClick.value = flag
    })
  }
}

function handleMoreBtnClick(event: MouseEvent) {
  emit('moreClick', event)
}

function handleUndo() {
  emit('undo')
}
</script>

<template>
  <div
    relative
  >
    <!-- By directly using predefined unocss width properties, it is possible to dynamically set the width attribute -->
    <div hidden w="xl:280px lg:250px md:200px 200px" />
    <div hidden w="full" />

    <template v-if="removed">
      <div
        :style="{ contentVisibility }"
        w-full
        pos="absolute top-0 left-0" aspect-video
      >
        <div :w="wValue" h-fit relative>
          <img
            :src="`${removeHttpFromUrl(cover)}@672w_378h_1c`" alt=""
            w-full h-fit object-cover pos="absolute top-0 left-0" aspect-video
            z--1 rounded="$bew-radius"
          >

          <div
            pos="absolute top-0 left-0" w-full h-fit aspect-video flex="~ col gap-2 items-center justify-center"
            bg="$bew-fill-4" backdrop-blur-20px mix-blend-luminosity transform-gpu rounded="$bew-radius"
          >
            <p mb-2 color-white text-lg>
              {{ $t('home.video_removed') }}
            </p>
            <Button
              color="rgba(255,255,255,.35)" text-color="white" size="small"
              @click="handleUndo"
            >
              <template #left>
                <div i-mingcute-back-line text-lg />
              </template>
              {{ $t('common.undo') }}
            </Button>
          </div>
        </div>
      </div>
    </template>
    <div
      v-else
      class="video-card group"
      w="full" pos="absolute top-0 left-0"
      rounded="$bew-radius" duration-300 ease-in-out
      bg="hover:$bew-fill-2 active:$bew-fill-3" hover:ring="8 $bew-fill-2" active:ring="8 $bew-fill-3"
      :style="{ contentVisibility }"
    >
      <a
        :style="{ display: horizontal ? 'flex' : 'block', gap: horizontal ? '1.5rem' : '0' }"
        :href="videoUrl" target="_blank" rel="noopener noreferrer"
        @mouseenter="handleMouseEnter"
        @mouseleave="handelMouseLeave"
        @mousedown="switchClickState(true)"
        @mouseup="switchClickState(false)"
        @dragend="switchClickState(false)"
      >
        <!-- Cover -->
        <div
          class="group/cover"
          shrink-0
          :w="wValue" h-fit relative bg="$bew-fill-4" rounded="$bew-radius"
          cursor-pointer
          duration-300 ease-in-out
          group-hover:z-2
        >
          <!-- Video preview -->
          <Transition v-if="showPreview && settings.enableVideoPreview" name="fade">
            <video
              v-if="previewVideoUrl && isHover"
              autoplay muted
              :controls="settings.enableVideoCtrlBarOnVideoCard"
              :style="{ pointerEvents: settings.enableVideoCtrlBarOnVideoCard ? 'auto' : 'none' }"
              pos="absolute top-0 left-0" w-full aspect-video rounded="$bew-radius" bg-black
              @mouseenter="handleMouseEnter"
            >
              <source :src="previewVideoUrl" type="video/mp4">
            </video>
          </Transition>
          <!-- <video  /> -->
          <!-- style="--un-shadow: 0 0 0 4px var(--bew-theme-color)" -->
          <!-- group-hover:transform="translate--4px" -->
          <!-- Ranking Number -->
          <div v-if="rank" absolute p-2 group-hover:opacity-0 duration-300>
            <div
              v-if="Number(rank) <= 3"
              bg="$bew-theme-color" text-center lh-30px h-30px w-30px
              text-white rounded="1/2" shadow="$bew-shadow-1"
              border="1 $bew-theme-color"
              text="2xl" fw-bold
            >
              {{ rank }}
            </div>
            <div
              v-else
              bg="$bew-elevated-solid-1" text-center lh-30px h-30px w-30px
              rounded="1/2" shadow="$bew-shadow-1"
              border="1 $bew-border-color"
            >
              {{ rank }}
            </div>
          </div>

          <!-- Video Duration -->
          <div
            v-if="duration || durationStr"
            pos="absolute bottom-0 right-0"
            z="2"
            p="x-2 y-1"
            m="1"
            rounded="$bew-radius"
            text="!white xs"
            bg="black opacity-60"
            class="group-hover:opacity-0"
            duration-300
          >
            {{ duration ? calcCurrentTime(duration) : durationStr }}
          </div>

          <div
            class="opacity-0 group-hover/cover:opacity-100"
            transform="scale-70 group-hover/cover:scale-100"
            duration-300
            pos="absolute top-0 left-0" z-2
          >
            <slot name="coverTopLeft" />
          </div>

          <button
            v-if="topRightContent"
            pos="absolute top-0 right-0" z="2"
            p="x-2 y-1" m="1"
            rounded="$bew-radius"
            text="!white xl"
            bg="black opacity-60"
            class="opacity-0 group-hover/cover:opacity-100"
            transform="scale-70 group-hover/cover:scale-100"
            duration-300
            @click.prevent="toggleWatchLater"
          >
            <Tooltip v-if="!isInWatchLater" :content="$t('common.save_to_watch_later')" placement="bottom" type="dark">
              <div i-mingcute:carplay-line />
            </Tooltip>
            <Tooltip v-else :content="$t('common.added')" placement="bottom" type="dark">
              <div i-line-md:confirm />
            </Tooltip>
          </button>

          <!-- Video cover -->
          <img
            :src="`${removeHttpFromUrl(cover)}@672w_378h_1c`"
            loading="lazy"
            w="full" max-w-full align-middle aspect-video
            bg="cover center"
            rounded="$bew-radius"
          >
        </div>

        <!-- Other Information -->
        <div
          :style="{
            width: horizontal ? '100%' : 'unset',
            marginTop: horizontal ? '0' : '1rem',
          }"
          flex="~"
        >
          <!-- Author Avatar -->
          <div v-if="!horizontal" flex>
            <a
              v-if="authorFace"
              :href="authorJumpUrl" target="_blank" rel="noopener noreferrer"
              m="r-4" w="36px" h="36px" rounded="1/2"
              object="center cover" bg="$bew-fill-4" cursor="pointer"
              position-relative
              @click.stop=""
            >
              <img
                rounded="1/2"
                :src="`${removeHttpFromUrl(authorFace)}@50w_50h_1c`"
                width="36"
                height="36"
                loading="lazy"
              >
              <div
                v-if="followed"
                pos="absolute bottom--2px right--2px"
                w-14px h-14px
                bg="$bew-theme-color"
                rounded="1/2"
                grid place-items-center
              >
                <div color-white text-sm class="i-mingcute:check-fill w-10px h-10px" />
              </div>
            </a>
          </div>
          <div class="group/desc" flex="~ col" w="full" align="items-start">
            <div flex="~ gap-1 justify-between items-start" w="full" pos="relative">
              <h3
                class="keep-two-lines"
                text="lg overflow-ellipsis $bew-text-1"
                cursor="pointer"
              >
                <a :href="videoUrl" target="_blank" :title="title" rel="noopener noreferrer">
                  {{ title }}
                </a>
              </h3>

              <div
                v-if="moreBtn"
                class="opacity-0 group-hover/desc:opacity-100"
                :class="{ 'more-active': moreBtnActive }"
                shrink-0 w-30px h-30px m="t--3px r--8px" translate-x--8px
                grid place-items-center
                pointer="auto" rounded="50%" duration-300
                @click.prevent="handleMoreBtnClick"
              >
                <div i-mingcute:more-2-line text="lg" />
              </div>
            </div>
            <div text="base $bew-text-2" w-fit m="t-2" flex="~ items-center wrap">
              <!-- Author Avatar -->
              <span flex="inline items-center">
                <div v-if="horizontal" flex mb-2>
                  <a
                    v-if="authorFace"
                    :href="authorJumpUrl" target="_blank" rel="noopener noreferrer"
                    m="r-2" w="30px" h="30px" rounded="1/2"
                    object="center cover" bg="$bew-fill-4" cursor="pointer" relative
                    @click.stop=""
                  >
                    <img
                      :src="`${removeHttpFromUrl(authorFace)}@50w_50h_1c`"
                      width="30"
                      height="30"
                      loading="lazy"
                      object-cover rounded="1/2"
                    >
                    <div
                      v-if="followed"
                      pos="absolute bottom--2px right--2px"
                      w-14px h-14px
                      bg="$bew-theme-color"
                      rounded="1/2"
                      grid place-items-center
                    >
                      <div color-white text-sm class="i-mingcute:check-fill w-10px h-10px" />
                    </div>
                  </a>
                </div>

                <a
                  v-if="author"
                  class="channel-name"
                  un-text="hover:$bew-text-1"
                  cursor-pointer mr-4
                  :href="authorJumpUrl" target="_blank" rel="noopener noreferrer"
                  @click.stop=""
                >
                  <span>{{ author }}</span>
                </a>
              </span>
            </div>

            <div flex="~ items-center gap-1 wrap">
              <!-- View & Danmaku Count -->
              <div
                text="$bew-text-2" rounded="$bew-radius"
                inline-block
              >
                <span v-if="view || viewStr">
                  {{ view ? $t('common.view', { count: numFormatter(view) }, view) : `${viewStr}${$t('common.viewWithoutNum')}` }}
                </span>
                <template v-if="danmaku || danmakuStr">
                  <span text-xs font-light mx-4px>•</span>
                  <span>{{ danmaku ? $t('common.danmaku', { count: numFormatter(danmaku) }, danmaku) : `${danmakuStr}${$t('common.danmakuWithoutNum')}` }}</span>
                </template>
                <br>
              </div>
            </div>
            <div mt-2 flex="~ gap-1">
              <!-- Tag -->
              <span
                v-if="tag"
                text="$bew-theme-color sm" lh-6 p="x-2" rounded="$bew-radius" bg="$bew-theme-color-20"
              >
                {{ tag }}
              </span>
              <span
                v-if="publishedTimestamp || capsuleText"
                bg="$bew-fill-1" p="x-2" rounded="$bew-radius" text="sm $bew-text-3" lh-6
                mr-1
              >
                {{ publishedTimestamp ? calcTimeSince(publishedTimestamp * 1000) : capsuleText?.trim() }}
              </span>
              <!-- Video type -->
              <span text="$bew-text-2" grid="~ place-items-center">
                <div v-if="type === 'vertical'" i-mingcute:cellphone-2-line />
                <div v-else-if="type === 'bangumi'" i-mingcute:movie-line />
              </span>
            </div>
          </div>
        </div>
      </a>
    </div>

    <!-- skeleton -->
    <template v-if="!horizontal">
      <div
        block mb-4 pointer-events-none select-none invisible
      >
        <!-- Cover -->
        <div w-full shrink-0 aspect-video h-fit rounded="$bew-radius" />
        <!-- Other Information -->
        <div
          mt-4 flex="~ gap-4"
        >
          <div
            block w="36px" h="36px" rounded="1/2" shrink-0
          />
          <div w-full>
            <div grid gap-2>
              <div w-full h-5 />
              <div w="3/4" h-5 />
            </div>
            <div grid gap-2 mt-4>
              <div w="40%" h-4 />
              <div w="80%" h-4 />
            </div>
            <div mt-2 flex>
              <div text="transparent sm" inline-block p="x-2" h-7 rounded-4>
                hello world
              </div>
            </div>
          </div>
        </div>
      </div>
    </template>
    <template v-else>
      <div
        flex="~ gap-6"
        mb-4 pointer-events-none select-none invisible
      >
        <!-- Cover -->
        <div
          :w="wValue"
          shrink-0 aspect-video h-fit rounded="$bew-radius"
        />
        <!-- Other Information -->
        <div
          w-full flex="~ gap-4"
        >
          <div w-full>
            <div grid gap-2>
              <div w-full h-5 />
              <div w="3/4" h-5 />
            </div>
            <div grid gap-2 mt-4>
              <div w="70%" h-4 />
            </div>
            <div mt-4 flex>
              <div text="transparent sm" inline-block p="x-2" h-7 rounded-4>
                hello world
              </div>
            </div>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<style lang="scss" scoped>
// .video-card.is-dislike {
//   > *:not(#dislike-control) {
//     --at-apply: invisible pointer-events-none duration-0 transition-none;
//   }
// }

.more-active {
  --at-apply: opacity-100 bg-$bew-fill-3;
}
</style>
