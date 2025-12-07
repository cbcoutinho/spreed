<!--
  - SPDX-FileCopyrightText: 2025 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<script setup lang="ts">

import { t } from '@nextcloud/l10n'
import { computed, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { useStore } from 'vuex'
import NcActionButton from '@nextcloud/vue/components/NcActionButton'
import NcListItem from '@nextcloud/vue/components/NcListItem'
import IconClose from 'vue-material-design-icons/Close.vue'
import IconPinOff from 'vue-material-design-icons/PinOffOutline.vue'
import AvatarWrapper from '../../AvatarWrapper/AvatarWrapper.vue'
import { AVATAR } from '../../..//constants.ts'
import { useGetToken } from '../../../composables/useGetToken.ts'
import { EventBus } from '../../../services/EventBus.ts'
import { useSharedItemsStore } from '../../../stores/sharedItems.ts'

const store = useStore()
const route = useRoute()
const token = useGetToken()
const sharedItemsStore = useSharedItemsStore()
const conversation = computed(() => store.getters.conversation(token.value))

// Array of all pinned messages
const pinnedMessages = computed(() => {
    if (!sharedItemsStore.sharedItems(token.value).pinned) {
        return []
    }
    return Object.values(sharedItemsStore.sharedItems(token.value).pinned) || []
})

// The pinned message to be displayed (the latest one that is not hidden)
const pinnedMessage = computed(() => {
    if (!pinnedMessages.value.length) {
        return null
    }
	return pinnedMessages.value.find((item) => +item.id === conversation.value.lastPinnedId
        && item.id !== conversation.value.hiddenPinnedId)
})

const isModerator = computed(() => store.getters.isModerator)
const isInThread = computed(() => pinnedMessage.value?.threadId !== pinnedMessage.value?.id)
const to = computed(() => ({
	name: 'conversation',
	hash: `#message_${pinnedMessage.value?.id}`,
	params: { token: conversation.value.token },
	query: { threadId: isInThread.value ? pinnedMessage.value?.threadId : undefined },
}))

/**
 * Handle click on pinned message
 */
function handlePinClick() {
	if (route.hash === '#message_' + pinnedMessage.value?.id) {
		// Already on this message route, just trigger highlight
		EventBus.emit('focus-message', { messageId: pinnedMessage.value?.id })
	}
}

/**
 * Handle hiding the pinned message
 */
function handleHidePinnedMessage() {
    sharedItemsStore.handleHidePinnedMessage(token.value, pinnedMessage.value!.id)
}

onMounted(() => {
	if (!sharedItemsStore.sharedItems(token.value).pinned) {
        // This is only needed on relaod for each conversation
        // Afterwards, pinned messages are added/removed via system messages
		sharedItemsStore.fetchPinnedMessages(token.value)
	}
})
</script>

<template>
	<div v-if="pinnedMessage" class="pinned-message">
		<NcListItem
			:name="pinnedMessage.actorDisplayName"
			:title="pinnedMessage.message"
			:active="false"
			:to="to"
			@click="handlePinClick">
			<template #icon>
				<AvatarWrapper
					:id="pinnedMessage.actorId"
					:name="pinnedMessage.actorDisplayName"
					:source="pinnedMessage.actorType"
					disable-menu
					:token="token"
					:size="AVATAR.SIZE.SMALL" />
			</template>
			<template #subname>
				{{ pinnedMessage.message }}
			</template>
			<template #actions>
				<NcActionButton
					close-after-click
					:title="t('spreed', 'Discard pin')"
					@click.stop="handleHidePinnedMessage">
					<template #icon>
						<IconClose :size="20" />
					</template>
					{{ t('spreed', 'Discard pin') }}
				</NcActionButton>
				<NcActionButton
					v-if="isModerator"
					close-after-click
					:title="t('spreed', 'Unpin for all')"
					@click.stop="sharedItemsStore.handleUnpinMessage(token, pinnedMessage.id)">
					<template #icon>
						<IconPinOff :size="20" />
					</template>
					{{ t('spreed', 'Unpin for all') }}
				</NcActionButton>
			</template>
		</NcListItem>
	</div>
</template>

<style scoped lang="scss">
.pinned-message {
	position: absolute;
	top: 4px;
	inset-inline-start: 7%; // (100% - 86%) / 2
	width: 86%;
	z-index: 3;
	background-color: var(--color-background-plain-text);
	border-radius: var(--border-radius-container);
	border: 2px solid var(--color-border);
}

:deep(.list-item__wrapper) {
    padding: 0 !important;
}
</style>
