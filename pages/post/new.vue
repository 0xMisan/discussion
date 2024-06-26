<template>
  <UCard class="w-full mt-2">
    <template #header class="py-1">
      <div class="text-center text-sm">发表新帖子</div>
    </template>
    <div class="flex flex-col my-2 ">
      <UForm ref="form" :schema="createPostSchema" :state="state" :validate-on="['submit']" class="space-y-4"
        @submit="onSubmit" autocomplete="off">
        <UFormGroup label="标题" name="title">
          <UInput v-model="state.title" autocomplete="off" />
        </UFormGroup>
        <UFormGroup label="标签" name="tags">
          <USelectMenu value-attribute="id" option-attribute="desc" v-model="state.tagId" :options="tags">
          </USelectMenu>
        </UFormGroup>
        <UFormGroup label="正文" name="content">
          <client-only>
            <MdEditor :theme="mode as any" @onUploadImg="onUploadImg" style="max-height: 400px;" v-model="state.content"
              :preview="false" :toolbars="toolbars" editor-id="newPost">
              <template #defToolbars>
                <XEmoji />
                <XYoutubeDialog />
              </template>
            </MdEditor>
          </client-only>
        </UFormGroup>
        <div>
          <UButton type="submit" :loading="pending">
            发表(Ctrl+Enter提交)
          </UButton>
        </div>
      </UForm>
    </div>
  </UCard>
</template>


<script lang="ts" setup>
import type { FormSubmitEvent } from '#ui/types';
import { MdEditor, type ToolbarNames } from 'md-editor-v3';
import 'md-editor-v3/lib/style.css';
import { toast } from 'vue-sonner';
import { z } from 'zod';
import { createPostSchema, type PostDTO } from '~/types';
type Schema = z.output<typeof createPostSchema>
import { useColorMode } from '@vueuse/core'
const mode = useColorMode()

const route = useRoute()

const toolbars: ToolbarNames[] = [
  0, 1,
  'bold',
  'underline',
  'strikeThrough',
  'quote',
  'unorderedList',
  'orderedList',
  'codeRow',
  'code',
  'link',
  'image',
  'table',
  'preview',
];

const tagRes = await useFetch('/api/go/list', {
  method: 'POST',
  key: "tagLists"
})
const tags = computed(() => {
  const items = tagRes.data.value?.tags.map(item => { return { name: item.name, id: item.id, desc: item.enName + ' / ' + item.desc } })
  return items
})


const state = reactive<Schema>({
  pid: "",
  title: "",
  content: "",
  tagId: 0
})

const loadPost = async () => {
  const query = route.query
  const pid = (query.pid) as string || ''
  if (!pid) return
  const res = (await $fetch('/api/post/' + pid, {
    method: 'POST',
    body: JSON.stringify({})
  })) as any as { post: PostDTO }
  Object.assign(state, res.post)
}

await loadPost()
watch(() => route.fullPath, loadPost)


const pending = ref(false)

async function onSubmit(event: FormSubmitEvent<Schema>) {
  pending.value = true
  const result = await $fetch('/api/post/new', {
    method: 'POST',
    body: JSON.stringify(event.data)
  })
  if (result.success && 'pid' in result) {
    userCardChanged.emit()
    setTimeout(() => {
      navigateTo(`/post/${result.pid}`)
    }, 200)
  } else if ('message' in result) {
    toast.error('发表失败,' + (result.message))
  }
  pending.value = false
}

const form = ref<HTMLFormElement>()

const handleKeyDown = async (event: KeyboardEvent) => {
  if (event.ctrlKey && event.key === 'Enter') {
    form.value?.submit()
  }
};

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
});

useHead({
  title: `发表帖子`,
  meta: [
    { name: "keywords", content: "极简论坛" },
    { name: "description", content: "极简论坛" },
  ],
})
</script>

<style scoped></style>