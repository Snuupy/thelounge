<template>
	<form id="form" method="post" action="" @submit.prevent="onSubmit">
		<span id="nick">{{ network.nick }}</span>
		<textarea
			id="input"
			ref="input"
			:value="channel.pendingMessage"
			:placeholder="getInputPlaceholder(channel)"
			:aria-label="getInputPlaceholder(channel)"
			class="mousetrap"
			@input="setPendingMessage"
			@keypress.enter.exact.prevent="onSubmit"
		/>
		<span
			v-if="this.$root.isFileUploadEnabled"
			id="upload-tooltip"
			class="tooltipped tooltipped-w tooltipped-no-touch"
			aria-label="Upload file"
			@click="openFileUpload"
		>
			<input id="upload-input" ref="uploadInput" type="file" multiple />
			<button
				id="upload"
				type="button"
				aria-label="Upload file"
				:disabled="!this.$root.isConnected"
			/>
		</span>
		<span
			id="submit-tooltip"
			class="tooltipped tooltipped-w tooltipped-no-touch"
			aria-label="Send message"
		>
			<button
				id="submit"
				type="submit"
				aria-label="Send message"
				:disabled="!this.$root.isConnected"
			/>
		</span>
	</form>
</template>

<script>
const commands = require("../js/commands/index");
const socket = require("../js/socket");
const upload = require("../js/upload");
const Mousetrap = require("mousetrap");
const {wrapCursor} = require("undate");

const formattingHotkeys = {
	k: "\x03",
	b: "\x02",
	u: "\x1F",
	i: "\x1D",
	o: "\x0F",
	s: "\x1e",
	m: "\x11",
};

// Autocomplete bracket and quote characters like in a modern IDE
// For example, select `text`, press `[` key, and it becomes `[text]`
const bracketWraps = {
	'"': '"',
	"'": "'",
	"(": ")",
	"<": ">",
	"[": "]",
	"{": "}",
	"*": "*",
	"`": "`",
	"~": "~",
	_: "_",
};

export default {
	name: "ChatInput",
	props: {
		network: Object,
		channel: Object,
	},
	watch: {
		"channel.pendingMessage"() {
			this.setInputSize();
		},
	},
	mounted() {
		if (this.$root.settings.autocomplete) {
			require("../js/autocompletion").enable(this.$refs.input);
		}

		const inputTrap = Mousetrap(this.$refs.input);

		for (const hotkey in formattingHotkeys) {
			inputTrap.bind("mod+" + hotkey, function(e) {
				// Key is lowercased because keybinds also get processed if caps lock is on
				const modifier = formattingHotkeys[e.key.toLowerCase()];

				wrapCursor(
					e.target,
					modifier,
					e.target.selectionStart === e.target.selectionEnd ? "" : modifier
				);

				return false;
			});
		}

		inputTrap.bind(Object.keys(bracketWraps), function(e) {
			if (e.target.selectionStart !== e.target.selectionEnd) {
				wrapCursor(e.target, e.key, bracketWraps[e.key]);

				return false;
			}
		});

		inputTrap.bind(["up", "down"], (e, key) => {
			if (this.$root.isAutoCompleting || e.target.selectionStart !== e.target.selectionEnd) {
				return;
			}

			const {channel} = this;

			if (channel.inputHistoryPosition === 0) {
				channel.inputHistory[channel.inputHistoryPosition] = channel.pendingMessage;
			}

			if (key === "up") {
				if (channel.inputHistoryPosition < channel.inputHistory.length - 1) {
					channel.inputHistoryPosition++;
				}
			} else if (channel.inputHistoryPosition > 0) {
				channel.inputHistoryPosition--;
			}

			channel.pendingMessage = channel.inputHistory[channel.inputHistoryPosition];
			this.$refs.input.value = channel.pendingMessage;
			this.setInputSize();

			return false;
		});

		if (this.$root.isFileUploadEnabled) {
			upload.initialize();
		}
	},
	destroyed() {
		require("../js/autocompletion").disable();
		upload.abort();
	},
	methods: {
		setPendingMessage(e) {
			this.channel.pendingMessage = e.target.value;
			this.channel.inputHistoryPosition = 0;
			this.setInputSize();
		},
		setInputSize() {
			this.$nextTick(() => {
				const style = window.getComputedStyle(this.$refs.input);
				const lineHeight = parseFloat(style.lineHeight, 10) || 1;

				// Start by resetting height before computing as scrollHeight does not
				// decrease when deleting characters
				this.$refs.input.style.height = "";

				// Use scrollHeight to calculate how many lines there are in input, and ceil the value
				// because some browsers tend to incorrently round the values when using high density
				// displays or using page zoom feature
				this.$refs.input.style.height =
					Math.ceil(this.$refs.input.scrollHeight / lineHeight) * lineHeight + "px";
			});
		},
		getInputPlaceholder(channel) {
			if (channel.type === "channel" || channel.type === "query") {
				return `Write to ${channel.name}`;
			}

			return "";
		},
		onSubmit() {
			// Triggering click event opens the virtual keyboard on mobile
			// This can only be called from another interactive event (e.g. button click)
			this.$refs.input.click();
			this.$refs.input.focus();

			if (!this.$root.isConnected) {
				return false;
			}

			const target = this.channel.id;
			const text = this.channel.pendingMessage;

			if (text.length === 0) {
				return false;
			}

			this.channel.inputHistoryPosition = 0;
			this.channel.pendingMessage = "";
			this.$refs.input.value = "";
			this.setInputSize();

			// Store new message in history if last message isn't already equal
			if (this.channel.inputHistory[1] !== text) {
				this.channel.inputHistory.splice(1, 0, text);
			}

			// Limit input history to a 100 entries
			if (this.channel.inputHistory.length > 100) {
				this.channel.inputHistory.pop();
			}

			if (text[0] === "/") {
				const args = text.substr(1).split(" ");
				const cmd = args.shift().toLowerCase();

				if (
					Object.prototype.hasOwnProperty.call(commands, cmd) &&
					commands[cmd].input(args)
				) {
					return false;
				}
			}

			socket.emit("input", {target, text});
		},
		openFileUpload() {
			this.$refs.uploadInput.click();
		},
	},
};
</script>
