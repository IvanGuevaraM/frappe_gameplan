<template>
  <div
    class="rounded-md py-6 transition-shadow"
    :class="{
      ring: !comment.loading && highlightedComment == comment.name,
    }"
    :data-id="comment.name"
  >
    <UserInfo :email="comment.owner" v-slot="{ user }">
      <div class="mb-2 flex items-center text-base text-gray-900">
        <UserProfileLink class="mr-3" :user="user.name">
          <UserAvatar :user="user.name" />
        </UserProfileLink>
        <div class="md:flex md:items-center">
          <UserProfileLink
            class="font-medium hover:text-blue-600"
            :user="user.name"
          >
            {{ user.full_name }}
            <span class="hidden md:inline">&nbsp;&middot;&nbsp;</span>
          </UserProfileLink>
          <div>
            <time
              class="text-gray-600"
              :datetime="comment.creation"
              :title="$dayjs(comment.creation)"
            >
              {{ $dayjs(comment.creation).fromNow() }}
            </time>
            <span
              v-if="comment.modified > comment.creation"
              class="text-gray-600"
              :title="$dayjs(comment.modified)"
            >
              &nbsp;&middot; Edited
            </span>
            <span v-if="comment.loading" class="italic text-gray-600">
              &nbsp;&middot; Sending...
            </span>
            <div v-if="comment.error">
              &nbsp;&middot;
              <span class="text-red-600"> Error</span>
            </div>
          </div>
        </div>
        <Dropdown
          v-show="!comment.editing"
          class="ml-auto"
          placement="right"
          :button="{
            icon: 'more-horizontal',
            appearance: 'minimal',
            label: 'Comment Options',
          }"
          :options="[
            {
              label: 'Edit',
              icon: 'edit',
              handler: () => (comment.editing = true),
              condition: () =>
                $isSessionUser(comment.owner) &&
                !comment.deleted_at &&
                !readOnlyMode,
            },
            {
              label: 'Copy link',
              icon: 'link',
              handler: () => copyLink(comment),
            },
            {
              label: 'Delete',
              icon: 'trash',
              handler: () => {
                $dialog({
                  title: 'Delete comment',
                  message: 'Are you sure you want to delete this comment?',
                  actions: [
                    {
                      label: 'Delete',
                      appearance: 'danger',
                      handler: ({ close }) => {
                        return comments.setValue
                          .submit({
                            name: comment.name,
                            deleted_at: $dayjs().format('YYYY-MM-DD HH:mm:ss'),
                          })
                          .then(close)
                      },
                    },
                    {
                      label: 'Cancel',
                    },
                  ],
                })
              },
              condition: () =>
                $isSessionUser(comment.owner) &&
                comment.deleted_at == null &&
                !readOnlyMode,
            },
          ]"
        />
      </div>
      <div class="flex-1">
        <div
          :class="{
            'w-full rounded-lg border bg-white p-4 focus-within:border-gray-400':
              comment.editing,
          }"
          @keydown.ctrl.enter.capture.stop="editComment(comment)"
          @keydown.meta.enter.capture.stop="editComment(comment)"
        >
          <CommentEditor
            v-if="comment.deleted_at == null"
            :value="comment.content"
            @change="comment.content = $event"
            :editable="comment.editing || false"
            :submitButtonProps="{
              onClick: () => editComment(comment),
              loading: comment.loading,
            }"
            :discardButtonProps="{
              onClick: () => {
                comment.editing = false
                comments.fetchOne.submit(comment.name)
              },
            }"
          />
          <span class="text-base italic text-gray-600" v-else>
            This message is deleted
          </span>
          <div
            class="mt-3"
            v-if="!comment.deleted_at && !comment.editing && comment.reactions"
          >
            <Reactions
              doctype="GP Comment"
              :name="comment.name"
              v-model:reactions="comment.reactions"
              :read-only-mode="readOnlyMode"
            />
          </div>
        </div>
      </div>
    </UserInfo>
  </div>
</template>
<script>
import { Dropdown } from 'frappe-ui'
import { copyToClipboard } from '@/utils'
import UserProfileLink from './UserProfileLink.vue'
import CommentEditor from './CommentEditor.vue'
import Reactions from './Reactions.vue'

export default {
  name: 'Comment',
  props: {
    comment: {
      type: Object,
      required: true,
    },
    readOnlyMode: {
      type: Boolean,
      default: false,
    },
    highlightedComment: {
      type: [String, Number],
      default: null,
    },
    comments: {
      type: Object,
    },
  },
  components: { UserProfileLink, Dropdown, CommentEditor, Reactions },
  methods: {
    editComment(comment) {
      comment.loading = true
      comment.editing = false
      this.comments.setValue.submit(
        {
          name: comment.name,
          content: comment.content,
        },
        {
          onSuccess() {
            comment.loading = false
          },
          onError(error) {
            comment.loading = false
            comment.error = error
          },
        }
      )
    },
    copyLink(comment) {
      let location = window.location
      let url = `${location.origin}${location.pathname}?comment=${comment.name}`
      copyToClipboard(url)
    },
  },
}
</script>
