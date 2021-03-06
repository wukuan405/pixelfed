<template>
<div>
	<transition name="fade">
		<div class="card notification-card shadow-none border">
			<div class="card-body loader text-center" style="height: 200px;">
				<div class="spinner-border" role="status">
					<span class="sr-only">Loading...</span>
				</div>
			</div>
			<div v-if="notifications.length > 0" class="card-body px-0 py-0 contents" style="max-height: 240px; overflow-y: scroll;">
				<div v-if="profile.locked" class="media align-items-center mt-n2 px-3 py-2 border-bottom border-lighter bg-light cursor-pointer" @click="redirect('/account/follow-requests')">
					<div class="media-body font-weight-light pt-2 small d-flex align-items-center justify-content-between">
						<p class="mb-0 text-lighter"><i class="fas fa-cog text-light"></i></p>
						<p class="text-center pt-1 mb-1 text-dark font-weight-bold"><strong>{{followRequests.count}}</strong> Follow Requests</p>
						<p class="mb-0 text-lighter"><i class="fas fa-chevron-right"></i></p>
					</div>
				</div>
				<div v-if="notifications.length > 0" class="media align-items-center px-3 py-2 border-bottom border-light" v-for="(n, index) in notifications">
					<img class="mr-2 rounded-circle" style="border:1px solid #ccc" :src="n.account.avatar" alt="" width="32px" height="32px" onerror="this.onerror=null;this.src='/storage/avatars/default.png';">
					<div class="media-body font-weight-light small">
						<div v-if="n.type == 'favourite'">
							<p class="my-0">
								<a :href="n.account.url" class="font-weight-bold text-dark word-break" :title="n.account.username">{{truncate(n.account.username)}}</a> liked your 
								<span v-if="n.status.hasOwnProperty('media_attachments')">
									<a class="font-weight-bold" v-bind:href="n.status.url" :id="'fvn-' + n.id">post</a>.
									<b-popover :target="'fvn-' + n.id" title="" triggers="hover" placement="top" boundary="window">
										<img :src="notificationPreview(n)" width="100px" height="100px" style="object-fit: cover;">
									</b-popover>
								</span>
								<span v-else>
									<a class="font-weight-bold" v-bind:href="n.status.url">post</a>.
								</span>
							</p>
						</div>
						<div v-else-if="n.type == 'comment'">
							<p class="my-0">
								<a :href="n.account.url" class="font-weight-bold text-dark word-break" :title="n.account.username">{{truncate(n.account.username)}}</a> commented on your <a class="font-weight-bold" v-bind:href="n.status.url">post</a>.
							</p>
						</div>
						<div v-else-if="n.type == 'mention'">
							<p class="my-0">
								<a :href="n.account.url" class="font-weight-bold text-dark word-break" :title="n.account.username">{{truncate(n.account.username)}}</a> <a class="font-weight-bold" v-bind:href="mentionUrl(n.status)">mentioned</a> you.
							</p>
						</div>
						<div v-else-if="n.type == 'follow'">
							<p class="my-0">
								<a :href="n.account.url" class="font-weight-bold text-dark word-break" :title="n.account.username">{{truncate(n.account.username)}}</a> followed you.
							</p>
						</div>
						<div v-else-if="n.type == 'share'">
							<p class="my-0">
								<a :href="n.account.url" class="font-weight-bold text-dark word-break" :title="n.account.username">{{truncate(n.account.username)}}</a> shared your <a class="font-weight-bold" v-bind:href="n.status.url">post</a>.
							</p>
						</div>
						<div v-else-if="n.type == 'modlog'">
							<p class="my-0">
								<a :href="n.account.url" class="font-weight-bold text-dark word-break" :title="n.account.username">{{truncate(n.account.username)}}</a> updated a <a class="font-weight-bold" v-bind:href="n.modlog.url">modlog</a>.
							</p>
						</div>
						<div v-else-if="n.type == 'tagged'">
							<p class="my-0">
								<a :href="n.account.url" class="font-weight-bold text-dark word-break" :title="n.account.username">{{truncate(n.account.username)}}</a> tagged you in a <a class="font-weight-bold" v-bind:href="n.tagged.post_url">post</a>.
							</p>
						</div>
						<div v-else>
							<p class="my-0">
								We cannot display this notification at this time.
							</p>
						</div>
					</div>
					<div class="small text-muted font-weight-bold" :title="n.created_at">{{timeAgo(n.created_at)}}</div>
				</div>
				<div v-if="notifications.length">
					<infinite-loading @infinite="infiniteNotifications">
						<div slot="no-results" class="font-weight-bold"></div>
						<div slot="no-more" class="font-weight-bold"></div>
					</infinite-loading>
				</div>
				<div v-if="notifications.length == 0" class="text-lighter text-center py-3">
					<p class="mb-0"><i class="fas fa-inbox fa-3x"></i></p>
					<p class="mb-0 small font-weight-bold">0 Notifications!</p>
				</div>
			</div>
		</div>
	</transition>
</div>
</template>

<style type="text/css" scoped></style>

<script type="text/javascript">
	export default {
		data() {
			return {
				notifications: {},
				notificationCursor: 2,
				notificationMaxId: 0,
				profile: {
					locked: false
				},
				followRequests: null
			};
		},

		mounted() {
			let self = this;
			this.fetchNotifications();
			setTimeout(function() {
				self.profile = window._sharedData.curUser;
				self.fetchFollowRequests();
			}, 500);
		},

		updated() {
		},

		methods: {
			fetchNotifications() {
				axios.get('/api/pixelfed/v1/notifications?pg=true')
				.then(res => {
					let data = res.data;
					let ids = res.data.map(n => n.id);
					this.notificationMaxId = Math.min(...ids);
					this.notifications = data;
					$('.notification-card .loader').addClass('d-none');
					$('.notification-card .contents').removeClass('d-none');
					//this.notificationPoll();
				});
			},

			infiniteNotifications($state) {
				if(this.notificationCursor > 5) {
					$state.complete();
					return;
				}
				axios.get('/api/pixelfed/v1/notifications', {
					params: {
						page: this.notificationCursor
					}
				}).then(res => {
					if(res.data.length) {
						let data = res.data.filter(n => {
							if(n.type == 'share' && !status) {
								return false;
							}
							if(_.find(this.notifications, {id: n.id})) {
								return false;
							}
							return true;
						});
						this.notifications.push(...data);
						this.notificationCursor++;
						$state.loaded();
					} else {
						$state.complete();
					}
				});
			},

			truncate(text) {
				if(text.length <= 15) {
					return text;
				}

				return text.slice(0,15) + '...'
			},

			timeAgo(ts) {
				return window.App.util.format.timeAgo(ts);
			},

			mentionUrl(status) {
				let username = status.account.username;
				let id = status.id;
				return '/p/' + username + '/' + id;
			},

			notificationPoll() {
				let interval = this.notifications.length > 5 ? 15000 : 120000;
				let self = this;
				setInterval(function() {
					axios.get('/api/pixelfed/v1/notifications')
					.then(res => {
						let data = res.data.filter(n => {
							if(n.type == 'share' || self.notificationMaxId >= n.id) {
								return false;
							}
							return true;
						});
						if(data.length) {
							let ids = data.map(n => n.id);
							self.notificationMaxId = Math.max(...ids);

							self.notifications.unshift(...data);
							let beep = new Audio('/static/beep.mp3');
							beep.volume = 0.7;
							beep.play();
							$('.notification-card .far.fa-bell').addClass('fas text-danger').removeClass('far text-muted');
						}
					});
				}, interval);
			},

			refreshNotifications() {
				let self = this;
				axios.get('/api/pixelfed/v1/notifications')
				.then(res => {
					let data = res.data.filter(n => {
						if(n.type == 'share' || self.notificationMaxId >= n.id) {
							return false;
						}
						return true;
					});
					if(data.length > 0) {
						let ids = data.map(n => n.id);
						let max = Math.max(ids);
						if(max <= self.notificationMaxId) {
							return;
						} else {
							self.notificationMaxId = max;
							self.notifications = data;
							let beep = new Audio('/static/beep.mp3');
							beep.volume = 0.7;
							beep.play();
						}
					}
				});
			},

			fetchFollowRequests() {
				if(window._sharedData.curUser.locked == true) {
					axios.get('/account/follow-requests.json')
					.then(res => {
						this.followRequests = res.data;
					})
				}
			},

			redirect(url) {
				window.location.href = url;
			},

			notificationPreview(n) {
				if(!n.status.hasOwnProperty('media_attachments') || !n.status.media_attachments.length) {
					return '/storage/no-preview.png';
				}
				return n.status.media_attachments[0].preview_url;
			}
		}
	}
</script>