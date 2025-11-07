<script setup>
import { onMounted, ref, watch } from 'vue'
import { getComments, addComment, deleteComment } from '@/api/comments'
import Loader from './Loader.vue'
import Comment from './Comment.vue'
import NewCommentForm from './NewCommentForm.vue'

const currentPost = defineModel('currentPost', {
  type: Object,
  default: () => ({ id: null, title: '', body: '' }),
})

const comments = ref([])
const newCommentFormIsShown = ref(false)

/** Loading & error for fetching comments list */
const isLoading = ref(false)
const commentsError = ref('') // <-- visible error for list load

/** Parent-owned comment form state (persist name/email) */
const commentName = ref('')
const commentEmail = ref('')
const commentBody = ref('')

/** Submit state & error for form */
const isSubmittingComment = ref(false)
const submitError = ref('')

const onShowPost = async () => {
  if (!currentPost.value?.id) return

  // reset list errors and show loader
  commentsError.value = ''
  isLoading.value = true
  newCommentFormIsShown.value = false

  try {
    comments.value = await getComments(currentPost.value.id)
  } catch (error) {
    // user-facing error for failed load
    comments.value = []
    commentsError.value = 'Failed to load comments. Please try again.'
  } finally {
    isLoading.value = false
  }
}

const emit = defineEmits(['delete', 'edit'])

const showCommentForm = (show) => {
  newCommentFormIsShown.value = show
  submitError.value = ''
}

const createComment = async (name, email, body) => {
  // keep form open; set local submit loading and clear previous error
  isSubmittingComment.value = true
  submitError.value = ''

  try {
    const newComment = await addComment(currentPost.value.id, name, email, body)
    comments.value.push(newComment)

    // ✅ keep name & email; clear only body on success
    commentBody.value = ''
    // (Optionally keep form open for multiple comments)
  } catch (error) {
    // show user-facing submit error; keep form open for retry
    submitError.value = 'Could not add the comment. Please try again.'
  } finally {
    isSubmittingComment.value = false
  }
}

const removeComment = async (id) => {
  const original = [...comments.value]
  comments.value = comments.value.filter((c) => c.id !== id)
  try {
    await deleteComment(id)
  } catch (error) {
    comments.value = original
    // (Optional) add a toast/notification here if desired
  }
}

onMounted(onShowPost)
watch(() => currentPost.value, onShowPost, { deep: true })
</script>

<template>
  <!-- Comments load error (user-facing) -->
  <div
    v-if="commentsError && !isLoading"
    class="notification is-danger"
    data-cy="CommentsError"
    role="alert"
    style="margin-bottom: 0.75rem"
  >
    <button class="delete" @click="commentsError = ''" aria-label="dismiss"></button>
    {{ commentsError }}
  </div>

  <Loader v-if="isLoading" />

  <div class="block" v-if="!isLoading">
    <div class="is-flex is-justify-content-space-between is-align-items-center">
      <h2>#{{ currentPost.id }}: {{ currentPost.title }}</h2>

      <div class="is-flex">
        <span class="icon is-small is-right is-clickable" @click="emit('edit')">
          <i class="fas fa-pen-to-square"></i>
        </span>

        <span
          class="icon is-small is-right has-text-danger is-clickable ml-3"
          @click="emit('delete', currentPost.id)"
        >
          <i class="fas fa-trash"></i>
        </span>
      </div>
    </div>

    <p data-cy="PostBody">{{ currentPost.body }}</p>

    <div class="block" v-if="comments.length === 0 && !newCommentFormIsShown && !commentsError">
      <p class="title is-4">No comments yet</p>
    </div>

    <template v-if="comments.length > 0 && !newCommentFormIsShown">
      <Comment
        :key="comment.id"
        v-for="comment in comments"
        :comment="comment"
        @delete="removeComment"
      />
    </template>

    <!-- NewCommentForm with parent-managed state and submit loading -->
    <NewCommentForm
      v-if="newCommentFormIsShown"
      v-model:name="commentName"
      v-model:email="commentEmail"
      v-model:body="commentBody"
      :submitting="isSubmittingComment"     <!-- controls is-loading on button -->
      :error="submitError"                  <!-- show user-facing submit error -->
      @submit="createComment"
      @cancel="showCommentForm(false)"
    />

    <button
      v-if="!newCommentFormIsShown"
      type="button"
      class="button is-link"
      @click="showCommentForm(true)"
    >
      Write a comment
    </button>
  </div>
</template>

<style></style>
