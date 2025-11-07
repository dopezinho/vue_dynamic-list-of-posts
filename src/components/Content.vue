<script setup>
import { onMounted, ref, reactive } from 'vue'
import { getPosts, createPost, deletePost, updatePost } from '@/api/posts'
import { SidebarMode } from '@/utils/SidebarModes'
import AppHeader from './AppHeader.vue'
import PostsList from './PostsList.vue'
import Sidebar from './Sidebar.vue'
import PostPreview from './PostPreview.vue'
import PostForm from './PostForm.vue'

const user = defineModel('user', { type: Object })

const currentPost = reactive({
  id: null as number | null,
  title: '',
  body: '',
})

/** Main list loading & error (only for initial load) */
const listLoading = ref(false)
const loadError = ref('')

/** Sidebar-local states */
const sidebarMode = ref(SidebarMode.Closed)
const sidebarError = ref('')
const creating = ref(false)
const updating = ref(false)
const deleting = ref(false)

const posts = ref([])

onMounted(async () => {
  listLoading.value = true
  loadError.value = ''
  try {
    posts.value = await getPosts()
  } catch (error) {
    // Show a visible error to the user
    loadError.value = 'Failed to load posts. Please try again.'
    // optional: console.error(error)
  } finally {
    listLoading.value = false
  }
})

const handleViewPost = ({ id, title, body }) => {
  sidebarError.value = ''
  if (currentPost.id === id) {
    currentPost.id = null
    currentPost.title = ''
    currentPost.body = ''
    sidebarMode.value = SidebarMode.Closed
  } else {
    currentPost.id = id
    currentPost.title = title
    currentPost.body = body
    sidebarMode.value = SidebarMode.View_Post
  }
}

const openCreatePostForm = () => {
  sidebarError.value = ''
  currentPost.id = null
  currentPost.title = ''
  currentPost.body = ''
  sidebarMode.value = SidebarMode.Create_Post
}

const openEditPostForm = () => {
  sidebarError.value = ''
  sidebarMode.value = SidebarMode.Edit_Post
}

const cancelForm = () => {
  sidebarError.value = ''
  if (currentPost.id) {
    sidebarMode.value = SidebarMode.View_Post
  } else {
    sidebarMode.value = SidebarMode.Closed
    currentPost.title = ''
    currentPost.body = ''
  }
}

const handleCreatePost = async (title: string, body: string) => {
  creating.value = true
  sidebarError.value = ''
  try {
    const newPost = await createPost(title, body)
    posts.value.push(newPost)

    currentPost.id = newPost.id
    currentPost.title = newPost.title
    currentPost.body = newPost.body
    cancelForm()
  } catch (error) {
    sidebarError.value = 'Could not create the post. Please try again.'
  } finally {
    creating.value = false
  }
}

const handleUpdatePost = async (title: string, body: string) => {
  if (!currentPost.id) return
  updating.value = true
  sidebarError.value = ''
  try {
    const updatedPost = await updatePost(currentPost.id, title, body)
    posts.value = posts.value.map((post) =>
      post.id === updatedPost.id
        ? { ...post, title: updatedPost.title, body: updatedPost.body }
        : post,
    )
    currentPost.title = updatedPost.title
    currentPost.body = updatedPost.body
    cancelForm()
  } catch (error) {
    sidebarError.value = 'Could not update the post. Please try again.'
  } finally {
    updating.value = false
  }
}

const handleDeletePost = async (id: number) => {
  deleting.value = true
  sidebarError.value = ''
  try {
    await deletePost(id)
    posts.value = posts.value.filter((post) => post.id !== id)
    currentPost.id = null
    cancelForm()
  } catch (error) {
    sidebarError.value = 'Could not delete the post. Please try again.'
  } finally {
    deleting.value = false
  }
}
</script>

<template>
  <AppHeader v-model:user="user" />

  <main class="section">
    <div class="container">
      <!-- Top-level error notification for initial load -->
      <div v-if="loadError" class="notification is-danger" role="alert" style="margin-bottom: 1rem">
        <button class="delete" @click="loadError = ''" aria-label="dismiss"></button>
        {{ loadError }}
      </div>

      <div class="tile is-ancestor">
        <PostsList
          :posts="posts"
          :isFormOpen="sidebarMode === SidebarMode.Create_Post"
          :current-post-id="currentPost.id"
          v-model:isLoading="listLoading"
          @viewPost="handleViewPost"
          @createPost="openCreatePostForm"
        />

        <Sidebar :isOpen="sidebarMode !== SidebarMode.Closed">
          <!-- Sidebar-local error notification -->
          <div
            v-if="sidebarError"
            class="notification is-warning"
            role="alert"
            style="margin-bottom: 0.75rem"
          >
            <button class="delete" @click="sidebarError = ''" aria-label="dismiss"></button>
            {{ sidebarError }}
          </div>

          <PostPreview
            v-if="sidebarMode === SidebarMode.View_Post"
            v-model:currentPost="currentPost"
            :deleting="deleting"
            @edit="openEditPostForm"
            @delete="handleDeletePost"
          />

          <PostForm
            v-if="sidebarMode === SidebarMode.Create_Post"
            mode="create"
            v-model:currentPost="currentPost"
            :submitting="creating"
            @submit="handleCreatePost"
            @cancel="cancelForm"
          />

          <PostForm
            v-if="sidebarMode === SidebarMode.Edit_Post"
            mode="edit"
            v-model:currentPost="currentPost"
            :submitting="updating"
            @submit="handleUpdatePost"
            @cancel="cancelForm"
          />
        </Sidebar>
      </div>
    </div>
  </main>
</template>

<style>
h2 {
  font-weight: 600 !important;
}
.tile.is-ancestor:last-child {
  margin-bottom: -0.75rem;
}
.tile.is-parent {
  padding: 0.75rem;
}
.tile {
  align-items: stretch;
  display: block;
  flex-basis: 0;
  flex-grow: 1;
  flex-shrink: 1;
  min-height: -webkit-min-content;
  min-height: -moz-min-content;
  min-height: min-content;
}
@media screen and (min-width: 769px), print {
  .tile:not(.is-child) {
    display: flex;
  }
}
</style>
