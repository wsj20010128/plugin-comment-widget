<script lang="ts" setup>
import ReplyItem from "./ReplyItem.vue";
import {
  VTag,
  VAvatar,
  VEmpty,
  VSpace,
  VButton,
  VLoading,
} from "@halo-dev/components";
import Form from "./Form.vue";
import type { CommentVo, ReplyVo } from "@halo-dev/api-client";
import { computed, provide, ref, watch, inject, type Ref } from "vue";
import { apiClient } from "@/utils/api-client";
import { formatDatetime, timeAgo } from "@/utils/date";
import MdiCardsHeart from "~icons/mdi/cards-heart";
import MdiCardsHeartOutline from "~icons/mdi/cards-heart-outline";
import MdiCommentQuoteOutline from "~icons/mdi/comment-quote-outline";
import MdiCommentQuote from "~icons/mdi/comment-quote";

const props = withDefaults(
  defineProps<{
    comment?: CommentVo;
  }>(),
  {
    comment: undefined,
  }
);

const emit = defineEmits<{
  (event: "reload"): void;
}>();

const showReplies = ref(false);
const showForm = ref(false);

const replies = ref<ReplyVo[]>([] as ReplyVo[]);
const loading = ref(false);
const hoveredReply = ref<ReplyVo>();

provide<Ref<ReplyVo | undefined>>("hoveredReply", hoveredReply);

const isAuthor = computed(() => {
  if (!props.comment) {
    return false;
  }
  return false;
  // TODO get post info
  // const { comment, subject, owner } = props.comment;
  // if (comment.spec.owner.kind !== "User") {
  //   return false;
  // }
  // if (!subject) {
  //   return false;
  // }
  // const { spec } = subject as Post;
  // return owner?.name === spec.owner;
});

const website = computed(() => {
  if (!props.comment) {
    return "";
  }
  const { annotations } = props.comment.spec.owner;
  return annotations?.website;
});

const handleFetchReplies = async (mute?: boolean) => {
  try {
    if (!mute) {
      loading.value = true;
    }
    const { data } = await apiClient.comment.listCommentReplies({
      name: props.comment?.metadata.name as string,
    });
    replies.value = data.items;
  } catch (error) {
    console.error("Failed to fetch comment replies", error);
  } finally {
    loading.value = false;
  }
};

watch(
  () => showReplies.value,
  () => {
    if (showReplies.value) {
      handleFetchReplies();
    } else {
      replies.value.length = 0;
    }
  }
);

const onReplyCreated = () => {
  showForm.value = false;
  showReplies.value = true;
  handleFetchReplies();
};

// upvote
const upvotedComments = inject<Ref<string[]>>("upvotedComments", ref([]));
const handleUpvote = async () => {
  if (!props.comment) {
    return;
  }

  if (upvotedComments.value.includes(props.comment.metadata.name)) {
    return;
  }

  await apiClient.tracker.upvote({
    voteRequest: {
      name: props.comment.metadata.name,
      plural: "comments",
      group: "content.halo.run",
    },
  });

  upvotedComments.value.push(props.comment.metadata.name);

  emit("reload");
};
</script>

<template>
  <div :id="`comment-${comment?.metadata.name}`" class="comment-item py-4">
    <div class="flex flex-row gap-3">
      <div class="comment-avatar">
        <VAvatar
          :src="comment?.owner?.avatar"
          :alt="comment?.owner?.displayName"
          size="sm"
          circle
        />
      </div>
      <div class="flex-1">
        <div class="comment-informations flex items-center">
          <div class="flex flex-auto items-center gap-3">
            <div class="text-sm font-medium dark:text-slate-50">
              <a
                v-if="website"
                class="hover:text-gray-600 dark:hover:text-slate-300"
                :href="website"
                target="_blank"
              >
                {{ comment?.owner.displayName }}
              </a>
              <span v-else>
                {{ comment?.owner.displayName }}
              </span>
            </div>
            <span
              class="text-xs text-gray-500 dark:text-slate-400"
              :title="formatDatetime(comment?.spec.creationTime)"
            >
              {{ timeAgo(comment?.spec.creationTime) }}
            </span>
            <span
              v-if="!comment?.spec.approved"
              class="text-xs text-gray-500 dark:text-slate-400"
            >
              审核中
            </span>
            <VTag
              v-if="isAuthor"
              rounded
              class="dark:!border-slate-600 dark:!bg-slate-700 dark:!text-slate-50"
            >
              Author
            </VTag>
          </div>
        </div>
        <div class="comment-content mt-2">
          <pre
            class="whitespace-pre-wrap break-words text-sm text-gray-800 dark:text-slate-200"
            >{{ comment?.spec.content }}</pre
          >
        </div>
        <div class="comment-actions mt-2 flex flex-auto items-center gap-1.5">
          <div
            class="inline-flex cursor-pointer select-none items-center gap-1 text-xs text-gray-600 hover:text-gray-900 dark:text-slate-500 dark:hover:text-slate-400"
            @click="handleUpvote()"
          >
            <MdiCardsHeartOutline
              v-if="!upvotedComments.includes(comment?.metadata.name as string)"
              class="h-3.5 w-3.5 hover:text-red-600 hover:dark:text-red-400"
            />
            <MdiCardsHeart
              v-else
              class="h-3.5 w-3.5 text-red-600 dark:text-red-400"
            />
            <span>
              {{ comment?.stats.upvote }}
            </span>
          </div>
          <span class="text-gray-600">·</span>
          <div
            class="inline-flex cursor-pointer select-none items-center gap-1 text-xs text-gray-600 hover:text-gray-900 dark:text-slate-500 dark:hover:text-slate-400"
            @click="showReplies = !showReplies"
          >
            <MdiCommentQuoteOutline v-if="!showReplies" class="h-3.5 w-3.5" />
            <MdiCommentQuote v-else class="h-3.5 w-3.5" />
            <span> {{ comment?.status?.visibleReplyCount || 0 }} </span>
          </div>
          <span class="text-gray-600">·</span>
          <span
            class="cursor-pointer select-none text-xs text-gray-600 hover:text-gray-900 dark:text-slate-500 dark:hover:text-slate-400"
            @click="showForm = !showForm"
          >
            加入回复
          </span>
        </div>

        <Form
          v-if="showForm"
          class="mt-2"
          :comment="comment"
          @created="onReplyCreated"
        />

        <div v-if="showReplies" class="comment-replies mt-2">
          <div
            class="flex flex-col divide-y divide-gray-100 dark:divide-slate-700"
          >
            <VLoading v-if="loading" class="dark:text-slate-100" />
            <Transition
              v-else-if="!replies.length && !showForm"
              appear
              name="fade"
            >
              <VEmpty
                title="暂无回复"
                message="你可以尝试点击刷新或者添加新回复"
              >
                <template #actions>
                  <VSpace>
                    <VButton type="default" @click="handleFetchReplies">
                      刷新
                    </VButton>
                    <VButton type="primary" @click="showForm = true">
                      回复
                    </VButton>
                  </VSpace>
                </template>
              </VEmpty>
            </Transition>
            <TransitionGroup v-else appear name="fade" tag="div">
              <ReplyItem
                v-for="(reply, index) in replies"
                :key="index"
                :class="{ '!pt-2': index === 1 }"
                :comment="comment"
                :reply="reply"
                :replies="replies"
                @reload="handleFetchReplies(true)"
              ></ReplyItem>
            </TransitionGroup>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
