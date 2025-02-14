<template>
<div class="gafaadew"
	@dragover.stop="onDragover"
	@dragenter="onDragenter"
	@dragleave="onDragleave"
	@drop.stop="onDrop"
>
	<header>
		<button class="cancel _button" @click="cancel"><fa :icon="faTimes"/></button>
		<div>
			<span class="text-count" :class="{ over: trimmedLength(text) > 2000 }">{{ 2000 - trimmedLength(text) }}</span>
			<button class="submit _buttonPrimary" :disabled="!canPost" @click="post">{{ submitText }}</button>
		</div>
	</header>
	<div class="form">
		<x-note-preview class="preview" v-if="reply" :note="reply"/>
		<x-note-preview class="preview" v-if="renote" :note="renote"/>
		<div class="with-quote" v-if="quoteId"><fa icon="quote-left"/> {{ $t('@.post-form.quote-attached') }}<button @click="quoteId = null"><fa icon="times"/></button></div>
		<div v-if="visibility === 'specified'" class="to-specified">
			{{ $t('recipient') }}
			<div class="visibleUsers">
				<span v-for="u in visibleUsers">
					<dp-acct :user="u"/>
					<button class="_button" @click="removeVisibleUser(u)"><fa :icon="faTimes"/></button>
				</span>
				<button @click="addVisibleUser" class="_button"><fa :icon="faPlus"/></button>
			</div>
		</div>
		<input v-show="useCw" ref="cw" v-model="cw" :placeholder="$t('annotation')" v-autocomplete="{ model: 'cw' }">
		<textarea v-model="text" ref="text" :disabled="posting" :placeholder="placeholder" v-autocomplete="{ model: 'text' }" @keydown="onKeydown" @paste="onPaste"></textarea>
		<x-post-form-attaches class="attaches" :files="files"/>
		<x-poll-editor v-if="poll" ref="poll" @destroyed="poll = false" @updated="onPollUpdate()"/>
		<x-uploader ref="uploader" @uploaded="attachMedia" @change="onChangeUploadings"/>
		<footer>
			<button class="_button" @click="chooseFile"><fa :icon="faUpload"/></button>
			<button class="_button" @click="poll = !poll"><fa :icon="faChartPie"/></button>
			<button class="_button" @click="useCw = !useCw"><fa :icon="faEyeSlash"/></button>
			<button class="_button" @click="setVisibility" ref="visibilityButton">
				<span v-if="visibility === 'public'"><fa :icon="faGlobe"/></span>
				<span v-if="visibility === 'home'"><fa :icon="faHome"/></span>
				<span v-if="visibility === 'followers'"><fa :icon="faUnlock"/></span>
				<span v-if="visibility === 'specified'"><fa :icon="faEnvelope"/></span>
			</button>
		</footer>
		<input ref="file" class="file _button" type="file" multiple="multiple" @change="onChangeFile"/>
	</div>
</div>
</template>

<script lang="ts">
import Vue from 'vue';
import { faTimes, faUpload, faChartPie, faGlobe, faHome, faUnlock, faEnvelope, faPlus } from '@fortawesome/free-solid-svg-icons';
import { faEyeSlash, faLaugh } from '@fortawesome/free-regular-svg-icons';
import insertTextAtCursor from 'insert-text-at-cursor';
import { length } from 'stringz';
import { toASCII } from 'punycode';
import i18n from '../i18n';
import DpVisibilityChooser from './visibility-chooser.vue';
import DpUserSelect from './user-select.vue';
import XNotePreview from './note-preview.vue';
import { parse } from '../../mfm/parse';
import { host, url } from '../config';
import { erase, unique } from '../../prelude/array';
import extractMentions from '../../misc/extract-mentions';
import { formatTimeString } from '../../misc/format-time-string';

export default Vue.extend({
	i18n,

	components: {
		XNotePreview,
		XUploader: () => import('./uploader.vue').then(m => m.default),
		XPostFormAttaches: () => import('./post-form-attaches.vue').then(m => m.default),
		XPollEditor: () => import('./poll-editor.vue').then(m => m.default)
	},

	props: {
		reply: {
			type: Object,
			required: false
		},
		renote: {
			type: Object,
			required: false
		},
		mention: {
			type: Object,
			required: false
		},
		specified: {
			type: Object,
			required: false
		},
		initialText: {
			type: String,
			required: false
		},
		initialNote: {
			type: Object,
			required: false
		},
		instant: {
			type: Boolean,
			required: false,
			default: false
		}
	},

	data() {
		return {
			posting: false,
			text: '',
			files: [],
			uploadings: [],
			poll: false,
			pollChoices: [],
			pollMultiple: false,
			pollExpiration: [],
			useCw: false,
			cw: null,
			visibility: 'public',
			visibleUsers: [],
			autocomplete: null,
			draghover: false,
			quoteId: null,
			recentHashtags: JSON.parse(localStorage.getItem('hashtags') || '[]'),
			faTimes, faUpload, faChartPie, faGlobe, faHome, faUnlock, faEnvelope, faEyeSlash, faLaugh, faPlus
		};
	},

	computed: {
		draftId(): string {
			return this.renote
				? `renote:${this.renote.id}`
				: this.reply
					? `reply:${this.reply.id}`
					: 'note';
		},

		placeholder(): string {
			return this.renote
				? this.$t('_postForm.quotePlaceholder')
				: this.reply
					? this.$t('_postForm.replyPlaceholder')
					: this.$t('_postForm.placeholder');
		},

		submitText(): string {
			return this.renote
				? this.$t('renote')
				: this.reply
					? this.$t('reply')
					: this.$t('_postForm.post');
		},

		canPost(): boolean {
			return !this.posting &&
				(1 <= this.text.length || 1 <= this.files.length || this.poll || this.renote) &&
				(length(this.text.trim()) <= 2000) &&
				(!this.poll || this.pollChoices.length >= 2);
		}
	},

	mounted() {
		if (this.initialText) {
			this.text = this.initialText;
		}

		if (this.mention) {
			this.text = this.mention.host ? `@${this.mention.username}@${toASCII(this.mention.host)}` : `@${this.mention.username}`;
			this.text += ' ';
		}

		if (this.reply && this.reply.user.host != null) {
			this.text = `@${this.reply.user.username}@${toASCII(this.reply.user.host)} `;
		}

		if (this.reply && this.reply.text != null) {
			const ast = parse(this.reply.text);

			for (const x of extractMentions(ast)) {
				const mention = x.host ? `@${x.username}@${toASCII(x.host)}` : `@${x.username}`;

				// 自分は除外
				if (this.$store.state.i.username == x.username && x.host == null) continue;
				if (this.$store.state.i.username == x.username && x.host == host) continue;

				// 重複は除外
				if (this.text.indexOf(`${mention} `) != -1) continue;

				this.text += `${mention} `;
			}
		}

		// デフォルト公開範囲
		this.applyVisibility(this.$store.state.settings.rememberNoteVisibility ? (this.$store.state.device.visibility || this.$store.state.settings.defaultNoteVisibility) : this.$store.state.settings.defaultNoteVisibility);

		// 公開以外へのリプライ時は元の公開範囲を引き継ぐ
		if (this.reply && ['home', 'followers', 'specified'].includes(this.reply.visibility)) {
			this.visibility = this.reply.visibility;
			if (this.reply.visibility === 'specified') {
				this.$root.api('users/show', {
					userIds: this.reply.visibleUserIds.filter(uid => uid !== this.$store.state.i.id && uid !== this.reply.userId)
				}).then(users => {
					this.visibleUsers.push(...users);
				});

				if (this.reply.userId !== this.$store.state.i.id) {
					this.$root.api('users/show', { userId: this.reply.userId }).then(user => {
						this.visibleUsers.push(user);
					});
				}
			}
		}

		if (this.specified) {
			this.visibility = 'specified';
			this.visibleUsers.push(this.specified);
		}

		// keep cw when reply
		if (this.$store.state.settings.keepCw && this.reply && this.reply.cw) {
			this.useCw = true;
			this.cw = this.reply.cw;
		}

		this.focus();

		this.$nextTick(() => {
			this.focus();
		});

		this.$nextTick(() => {
			// 書きかけの投稿を復元
			if (!this.instant && !this.mention) {
				const draft = JSON.parse(localStorage.getItem('drafts') || '{}')[this.draftId];
				if (draft) {
					this.text = draft.data.text;
					this.files = (draft.data.files || []).filter(e => e);
					if (draft.data.poll) {
						this.poll = true;
						this.$nextTick(() => {
							(this.$refs.poll as any).set(draft.data.poll);
						});
					}
					this.$emit('change-attached-files', this.files);
				}
			}

			// 削除して編集
			if (this.initialNote) {
				const init = this.initialNote;
				this.text = init.text ? init.text : '';
				this.files = init.files;
				this.cw = init.cw;
				this.useCw = init.cw != null;
				if (init.poll) {
					this.poll = true;
					this.$nextTick(() => {
						(this.$refs.poll as any).set({
							choices: init.poll.choices.map(c => c.text),
							multiple: init.poll.multiple
						});
					});
				}
				this.visibility = init.visibility;
				this.quoteId = init.renote ? init.renote.id : null;
			}

			this.$nextTick(() => this.watch());
		});
	},

	methods: {
		watch() {
			this.$watch('text', () => this.saveDraft());
			this.$watch('poll', () => this.saveDraft());
			this.$watch('files', () => this.saveDraft());
		},

		trimmedLength(text: string) {
			return length(text.trim());
		},

		addTag(tag: string) {
			insertTextAtCursor(this.$refs.text, ` #${tag} `);
		},

		focus() {
			(this.$refs.text as any).focus();
		},

		chooseFile() {
			(this.$refs.file as any).click();
		},

		attachMedia(driveFile) {
			this.files.push(driveFile);
		},

		detachMedia(id) {
			this.files = this.files.filter(x => x.id != id);
		},

		updateMedia(file) {
			Vue.set(this.files, this.files.findIndex(x => x.id === file.id), file);
		},

		onChangeFile() {
			for (const x of Array.from((this.$refs.file as any).files)) this.upload(x);
		},

		upload(file: File, name?: string) {
			(this.$refs.uploader as any).upload(file, this.$store.state.settings.uploadFolder, name);
		},

		onChangeUploadings(uploads) {
			this.$emit('change-uploadings', uploads);
		},

		onPollUpdate() {
			const got = this.$refs.poll.get();
			this.pollChoices = got.choices;
			this.pollMultiple = got.multiple;
			this.pollExpiration = [got.expiration, got.expiresAt || got.expiredAfter];
			this.saveDraft();
		},

		setVisibility() {
			const w = this.$root.new(DpVisibilityChooser, {
				source: this.$refs.visibilityButton,
				currentVisibility: this.visibility
			});
			w.$once('chosen', v => {
				this.applyVisibility(v);
			});
		},

		applyVisibility(v: string) {
			this.visibility = v;
		},

		addVisibleUser() {
			const vm = this.$root.new(DpUserSelect, {});
			vm.$once('selected', user => {
				this.visibleUsers.push(user);
			});
		},

		removeVisibleUser(user) {
			this.visibleUsers = erase(user, this.visibleUsers);
		},

		clear() {
			this.text = '';
			this.files = [];
			this.poll = false;
			this.quoteId = null;
			this.$emit('change-attached-files', this.files);
		},

		onKeydown(e) {
			if ((e.which == 10 || e.which == 13) && (e.ctrlKey || e.metaKey) && this.canPost) this.post();
		},

		async onPaste(e: ClipboardEvent) {
			for (const { item, i } of Array.from(e.clipboardData.items).map((item, i) => ({item, i}))) {
				if (item.kind == 'file') {
					const file = item.getAsFile();
					const lio = file.name.lastIndexOf('.');
					const ext = lio >= 0 ? file.name.slice(lio) : '';
					const formatted = `${formatTimeString(new Date(file.lastModified), this.$store.state.settings.pastedFileName).replace(/{{number}}/g, `${i + 1}`)}${ext}`;
					this.upload(file, formatted);
				}
			}

			const paste = e.clipboardData.getData('text');

			if (!this.renote && !this.quoteId && paste.startsWith(url + '/notes/')) {
				e.preventDefault();

				this.$root.dialog({
					type: 'info',
					text: this.$t('@.post-form.quote-question'),
					showCancelButton: true
				}).then(({ canceled }) => {
					if (canceled) {
						insertTextAtCursor(this.$refs.text, paste);
						return;
					}

					this.quoteId = paste.substr(url.length).match(/^\/notes\/(.+?)\/?$/)[1];
				});
			}
		},

		onDragover(e) {
			if (!e.dataTransfer.items[0]) return;
			const isFile = e.dataTransfer.items[0].kind == 'file';
			const isDriveFile = e.dataTransfer.types[0] == 'mk_drive_file';
			if (isFile || isDriveFile) {
				e.preventDefault();
				this.draghover = true;
				e.dataTransfer.dropEffect = e.dataTransfer.effectAllowed == 'all' ? 'copy' : 'move';
			}
		},

		onDragenter(e) {
			this.draghover = true;
		},

		onDragleave(e) {
			this.draghover = false;
		},

		onDrop(e): void {
			this.draghover = false;

			// ファイルだったら
			if (e.dataTransfer.files.length > 0) {
				e.preventDefault();
				for (const x of Array.from(e.dataTransfer.files)) this.upload(x);
				return;
			}

			//#region ドライブのファイル
			const driveFile = e.dataTransfer.getData('mk_drive_file');
			if (driveFile != null && driveFile != '') {
				const file = JSON.parse(driveFile);
				this.files.push(file);
				this.$emit('change-attached-files', this.files);
				e.preventDefault();
			}
			//#endregion
		},

		async emoji() {
			/*
			const Picker = await import('../../desktop/views/components/emoji-picker-dialog.vue').then(m => m.default);
			const button = this.$refs.emoji;
			const rect = button.getBoundingClientRect();
			const vm = this.$root.new(Picker, {
				x: button.offsetWidth + rect.left + window.pageXOffset,
				y: rect.top + window.pageYOffset
			});
			vm.$once('chosen', emoji => {
				insertTextAtCursor(this.$refs.text, emoji);
			});
			this.$once('hook:beforeDestroy', () => {
				vm.close();
			});*/
		},

		saveDraft() {
			if (this.instant) return;

			const data = JSON.parse(localStorage.getItem('drafts') || '{}');

			data[this.draftId] = {
				updatedAt: new Date(),
				data: {
					text: this.text,
					files: this.files,
					poll: this.poll && this.$refs.poll ? (this.$refs.poll as any).get() : undefined
				}
			};

			localStorage.setItem('drafts', JSON.stringify(data));
		},

		deleteDraft() {
			const data = JSON.parse(localStorage.getItem('drafts') || '{}');

			delete data[this.draftId];

			localStorage.setItem('drafts', JSON.stringify(data));
		},

		post() {
			this.posting = true;
			this.$root.api('notes/create', {
				text: this.text == '' ? undefined : this.text,
				fileIds: this.files.length > 0 ? this.files.map(f => f.id) : undefined,
				replyId: this.reply ? this.reply.id : undefined,
				renoteId: this.renote ? this.renote.id : this.quoteId ? this.quoteId : undefined,
				poll: this.poll ? (this.$refs.poll as any).get() : undefined,
				cw: this.useCw ? this.cw || '' : undefined,
				visibility: this.visibility,
				visibleUserIds: this.visibility == 'specified' ? this.visibleUsers.map(u => u.id) : undefined,
			}).then(data => {
				this.clear();
				this.deleteDraft();
				this.$emit('posted');
			}).catch(err => {
			}).then(() => {
				this.posting = false;
			});

			if (this.text && this.text != '') {
				const hashtags = parse(this.text).filter(x => x.node.type === 'hashtag').map(x => x.node.props.hashtag);
				const history = JSON.parse(localStorage.getItem('hashtags') || '[]') as string[];
				localStorage.setItem('hashtags', JSON.stringify(unique(hashtags.concat(history))));
			}
		},

		cancel() {
			this.$emit('cancel');
		}
	}
});
</script>

<style lang="scss" scoped>
@import '../theme';

.gafaadew {
	background: var(--bg);
	border-radius: var(--radius);
	box-shadow: 0 0 2px rgba(#000, 0.1);

	> header {
		z-index: 1000;
		height: 66px;

		@media (max-width: 500px) {
			height: 50px;
		}

		> .cancel {
			padding: 0;
			font-size: 20px;
			width: 64px;
			line-height: 66px;

			@media (max-width: 500px) {
				width: 50px;
				line-height: 50px;
			}
		}

		> div {
			position: absolute;
			top: 0;
			right: 0;

			> .text-count {
				line-height: 66px;

				@media (max-width: 500px) {
					line-height: 50px;
				}
			}

			> .submit {
				margin: 16px;
				padding: 0 16px;
				line-height: 34px;
				vertical-align: bottom;
				border-radius: 4px;

				@media (max-width: 500px) {
					margin: 8px;
				}

				&:disabled {
					opacity: 0.7;
				}
			}
		}
	}

	> .form {
		max-width: 500px;
		margin: 0 auto;

		> .preview {
			padding: 16px;
		}

		> .with-quote {
			margin: 0 0 8px 0;
			color: $primary;

			> button {
				padding: 4px 8px;
				color: var(--primaryAlpha04);

				&:hover {
					color: var(--primaryAlpha06);
				}

				&:active {
					color: var(--primaryDarken30);
				}
			}
		}

		> .to-specified {
			padding: 6px 24px;
			margin-bottom: 8px;
			overflow: auto;
			white-space: nowrap;

			@media (max-width: 500px) {
				padding: 6px 16px;
			}

			> .visibleUsers {
				display: inline;
				top: -1px;
				font-size: 14px;

				> span {
					margin-right: 14px;
					padding: 8px 0 8px 8px;
					border-radius: 8px;
					background: rgba(0, 0, 0, 0.1);

					@media (prefers-color-scheme: dark) {
						background: rgba(255, 255, 255, 0.1);
					}

					> button {
						padding: 4px 8px;
					}
				}
			}
		}

		> input {
			z-index: 1;
		}

		> input,
		> textarea {
			display: block;
			box-sizing: border-box;
			padding: 0 24px;
			margin: 0;
			width: 100%;
			font-size: 16px;
			border: none;
			border-radius: 0;
			background: transparent;
			color: var(--fg);
			font-family: initial;

			@media (max-width: 500px) {
				padding: 0 16px;
			}

			&:focus {
				outline: none;
			}

			&:disabled {
				opacity: 0.5;
			}
		}

		> textarea {
			max-width: 100%;
			min-width: 100%;
			min-height: 90px;

			@media (max-width: 500px) {
				min-height: 80px;
			}
		}

		> .dp-uploader {
			margin: 8px 0 0 0;
			padding: 8px;
		}

		> .file {
			display: none;
		}

		> footer {
			> * {
				display: inline-block;
				padding: 0;
				margin: 0;
				font-size: 16px;
				width: 64px;
				height: 64px;

				@media (max-width: 500px) {
					width: 48px;
					height: 48px;
				}

				&:hover {
					background: rgba(0, 0, 0, 0.05);

					@media (prefers-color-scheme: dark) {
						background: rgba(255, 255, 255, 0.05);
					}
				}
			}
		}
	}
}
</style>
