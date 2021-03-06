<template>
	<aside ref="userlist" class="userlist" @mouseleave="removeHoverUser">
		<div class="count">
			<input
				ref="input"
				:value="userSearchInput"
				:placeholder="
					channel.users.length + ' user' + (channel.users.length === 1 ? '' : 's')
				"
				type="search"
				class="search"
				aria-label="Search among the user list"
				tabindex="-1"
				@input="setUserSearchInput"
				@keydown.up="navigateUserList($event, -1)"
				@keydown.down="navigateUserList($event, 1)"
				@keydown.page-up="navigateUserList($event, -10)"
				@keydown.page-down="navigateUserList($event, 10)"
				@keydown.enter="selectUser"
			/>
		</div>
		<div class="names">
			<div
				v-for="(users, mode) in groupedUsers"
				:key="mode"
				:class="['user-mode', getModeClass(mode)]"
			>
				<template v-if="userSearchInput.length > 0">
					<Username
						v-for="user in users"
						:key="user.original.nick + '-search'"
						:on-hover="hoverUser"
						:active="user.original === activeUser"
						:user="user.original"
						v-html="user.string"
					/>
				</template>
				<template v-else>
					<Username
						v-for="user in users"
						:key="user.nick"
						:on-hover="hoverUser"
						:active="user === activeUser"
						:user="user"
					/>
				</template>
			</div>
		</div>
	</aside>
</template>

<script>
import {filter as fuzzyFilter} from "fuzzy";
import Username from "./Username.vue";

const modes = {
	"~": "owner",
	"&": "admin",
	"!": "admin",
	"@": "op",
	"%": "half-op",
	"+": "voice",
	"": "normal",
};

export default {
	name: "ChatUserList",
	components: {
		Username,
	},
	props: {
		channel: Object,
	},
	data() {
		return {
			userSearchInput: "",
			activeUser: null,
		};
	},
	computed: {
		// filteredUsers is computed, to avoid unnecessary filtering
		// as it is shared between filtering and keybindings.
		filteredUsers() {
			if (!this.userSearchInput) {
				return;
			}

			return fuzzyFilter(this.userSearchInput, this.channel.users, {
				pre: "<b>",
				post: "</b>",
				extract: (u) => u.nick,
			});
		},
		groupedUsers() {
			const groups = {};

			if (this.userSearchInput) {
				const result = this.filteredUsers;

				for (const user of result) {
					const mode = user.original.modes[0] || "";

					if (!groups[mode]) {
						groups[mode] = [];
					}

					// Prepend user mode to search result
					user.string = mode + user.string;

					groups[mode].push(user);
				}
			} else {
				for (const user of this.channel.users) {
					const mode = user.modes[0] || "";

					if (!groups[mode]) {
						groups[mode] = [user];
					} else {
						groups[mode].push(user);
					}
				}
			}

			return groups;
		},
	},
	methods: {
		setUserSearchInput(e) {
			this.userSearchInput = e.target.value;
		},
		getModeClass(mode) {
			return modes[mode];
		},
		selectUser() {
			// Simulate a click on the active user to open the context menu.
			// Coordinates are provided to position the menu correctly.
			if (!this.activeUser) {
				return;
			}

			const el = this.$refs.userlist.querySelector(".active");
			const rect = el.getBoundingClientRect();
			const ev = new MouseEvent("click", {
				view: window,
				bubbles: true,
				cancelable: true,
				clientX: rect.left,
				clientY: rect.top + rect.height,
			});
			el.dispatchEvent(ev);
		},
		hoverUser(user) {
			this.activeUser = user;
		},
		removeHoverUser() {
			this.activeUser = null;
		},
		navigateUserList(event, direction) {
			// Prevent propagation to stop global keybind handler from capturing pagedown/pageup
			// and redirecting it to the message list container for scrolling
			event.stopImmediatePropagation();
			event.preventDefault();

			let users = this.channel.users;

			// Only using filteredUsers when we have to avoids filtering when it's not needed
			if (this.userSearchInput) {
				users = this.filteredUsers.map((result) => result.original);
			}

			// Bail out if there's no users to select
			if (!users.length) {
				this.activeUser = null;
				return;
			}

			let currentIndex = users.indexOf(this.activeUser);

			// If there's no active user select the first or last one depending on direction
			if (!this.activeUser || currentIndex === -1) {
				this.activeUser = direction ? users[0] : users[users.length - 1];
				this.scrollToActiveUser();
				return;
			}

			currentIndex += direction;

			// Wrap around the list if necessary. Normaly each loop iterates once at most,
			// but might iterate more often if pgup or pgdown are used in a very short user list
			while (currentIndex < 0) {
				currentIndex += users.length;
			}

			while (currentIndex > users.length - 1) {
				currentIndex -= users.length;
			}

			this.activeUser = users[currentIndex];
			this.scrollToActiveUser();
		},
		scrollToActiveUser() {
			// Scroll the list if needed after the active class is applied
			this.$nextTick(() => {
				const el = this.$refs.userlist.querySelector(".active");
				el.scrollIntoView({block: "nearest", inline: "nearest"});
			});
		},
	},
};
</script>
