<template>
	<NcAppNavigationItem
		:title="title"
		:icon="icon"
		:menu-open.sync="actionsOpen"
		:to="{ name: 'note', params: { noteId: note.id.toString() } }"
		:class="{ actionsOpen }"
		:loading="loading.note"
		:editable="!note.readonly"
		:edit-label="t('notes', 'Rename')"
		:edit-placeholder="t('notes', 'Note\'s title')"
		@update:title="onRename"
	>
		<template #actions>
			<NcActionButton :icon="actionFavoriteIcon" @click="onToggleFavorite">
				{{ actionFavoriteText }}
			</NcActionButton>
			<NcActionButton icon="icon-details" @click="onToggleSidebar">
				<SidebarIcon slot="icon" :size="20" />
				{{ t('notes', 'Details') }}
			</NcActionButton>
			<NcActionButton v-if="!note.readonly" :icon="actionDeleteIcon" @click="onDeleteNote">
				{{ t('notes', 'Delete note') }}
			</NcActionButton>
			<NcActionSeparator />
			<NcActionButton icon="icon-files-dark" @click="onCategorySelected">
				{{ actionCategoryText }}
			</NcActionButton>
		</template>
	</NcAppNavigationItem>
</template>

<script>
import {
	NcActionButton,
	NcActionSeparator,
	NcAppNavigationItem,
} from '@nextcloud/vue'
import SidebarIcon from 'vue-material-design-icons/PageLayoutSidebarRight.vue'
import { showError } from '@nextcloud/dialogs'
import store from '../store.js'

import { setFavorite, setTitle, fetchNote, deleteNote } from '../NotesService.js'
import { categoryLabel, routeIsNewNote } from '../Util.js'

export default {
	name: 'NavigationNoteItem',

	components: {
		NcActionButton,
		NcActionSeparator,
		NcAppNavigationItem,
		SidebarIcon,
	},

	props: {
		note: {
			type: Object,
			required: true,
		},
	},

	data() {
		return {
			loading: {
				note: false,
				favorite: false,
				delete: false,
			},
			actionsOpen: false,
		}
	},

	computed: {
		icon() {
			let icon = ''
			if (this.note.error) {
				icon = 'nav-icon icon-error-color'
			} else if (this.note.favorite) {
				icon = 'nav-icon icon-starred'
			}
			return icon
		},

		title() {
			return this.note.title + (this.note.unsaved ? ' *' : '')
		},

		actionFavoriteText() {
			return this.note.favorite ? this.t('notes', 'Remove from favorites') : this.t('notes', 'Add to favorites')
		},

		actionFavoriteIcon() {
			let icon = this.note.favorite ? 'icon-star-dark' : 'icon-starred'
			if (this.loading.favorite) {
				icon += ' loading'
			}
			return icon
		},

		actionCategoryText() {
			return categoryLabel(this.note.category)
		},

		actionDeleteIcon() {
			return 'icon-delete' + (this.loading.delete ? ' loading' : '')
		},
	},

	methods: {
		onToggleFavorite() {
			this.loading.favorite = true
			setFavorite(this.note.id, !this.note.favorite)
				.catch(() => {
				})
				.then(() => {
					this.loading.favorite = false
					this.actionsOpen = false
				})
		},

		onCategorySelected() {
			this.actionsOpen = false
			this.$emit('category-selected', this.note.category)
		},

		onToggleSidebar() {
			this.actionsOpen = false
			store.commit('setSidebarOpen', !store.state.app.sidebarOpen)
		},

		onRename(newTitle) {
			this.loading.note = true
			setTitle(this.note.id, newTitle)
				.catch(() => {
				})
				.finally(() => {
					this.loading.note = false
				})
			if (routeIsNewNote(this.$route)) {
				this.$router.replace({
					name: 'note',
					params: { noteId: this.note.id.toString() },
				})
			}
		},

		async onDeleteNote() {
			this.loading.delete = true
			try {
				const note = await fetchNote(this.note.id)
				if (note.errorType) {
					throw new Error('Note has errors')
				}
				await deleteNote(this.note.id, () => {
					this.$emit('note-deleted', note)
					this.loading.delete = false
					this.actionsOpen = false
				})
			} catch (e) {
				showError(this.t('notes', 'Error during preparing note for deletion.'))
				this.loading.delete = false
				this.actionsOpen = false
			}
		},
	},
}
</script>
