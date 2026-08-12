## Complete Recent Discord Quest

> [!NOTE]
> This does not works in browser for quests which require you to play a game! Use the [desktop app](https://discord.com/download) to complete those.

How to use this script:
1. Accept a quest under the Quests tab
2. Press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd> to open DevTools
3. Go to the `Console` tab
4. Paste the following code and hit enter:
<details>
	<summary>Click to expand</summary>
	
```js
delete window.$;
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

function findExport(pred) {
	for (let id in wpRequire.c) {
		let mod = wpRequire.c[id];
		if (!mod?.exports) continue;
		for (let key in mod.exports) {
			try {
				if (mod.exports[key] && pred(mod.exports[key])) return mod.exports[key];
			} catch (e) {}
		}
	}
	return null;
}

let ApplicationStreamingStore = findExport(x => x?.__proto__?.getStreamerActiveStreamMetadata);
let RunningGameStore = findExport(x => x?.getRunningGames);
let QuestsStore = findExport(x => x?.__proto__?.getQuest);
let ChannelStore = findExport(x => x?.__proto__?.getAllThreadsForParent);
let GuildChannelStore = findExport(x => x?.getSFWDefaultChannel);
let FluxDispatcher = findExport(x => x?.__proto__?.flushWaitQueue);
let TokenStore = findExport(x => typeof x?.getToken === "function");
let UserStore = findExport(x => typeof x?.getCurrentUser === "function" && typeof x?.__proto__?.getUser === "function");

let token = TokenStore?.getToken();
if (!token) {
	console.log("Couldn't find your token!");
}

let api = {
	get: async ({ url }) => {
		let res = await fetch(`https://discord.com/api/v9${url}`, {
			headers: { Authorization: token },
		});
		return { body: await res.json() };
	},
	post: async ({ url, body }) => {
		let res = await fetch(`https://discord.com/api/v9${url}`, {
			method: "POST",
			headers: { Authorization: token, "Content-Type": "application/json" },
			body: JSON.stringify(body),
		});
		return { body: await res.json() };
	},
};

const supportedTasks = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"];
let isApp = typeof DiscordNative !== "undefined";

// --- helpers added for the 2026 quest config changes ---

// Discord moved the app off config.application in July 2026. It now lives
// per-task at taskConfig.tasks[<KEY>].applications[0], with the old
// config.application.id kept as a legacy fallback for older clients.
function appIdFor(taskConfig, taskName, legacyAppId) {
	return taskConfig?.tasks?.[taskName]?.applications?.[0]?.id ?? legacyAppId ?? null;
}

// Task keys can drift (e.g. PLAY_ON_DESKTOP_V2). Exact keys first, then
// family/prefix matching so variants still get routed to the right handler.
function pickTaskName(taskConfig) {
	const keys = Object.keys(taskConfig?.tasks ?? {});
	for (const k of supportedTasks) if (taskConfig.tasks[k] != null) return k;
	const order = [
		(k) => k === "ACHIEVEMENT_IN_ACTIVITY",
		(k) => k === "PLAY_ACTIVITY",
		(k) => k.startsWith("STREAM"),
		(k) => k.includes("VIDEO"),
		(k) => k.startsWith("PLAY"),
		(k) => k.includes("ACTIVITY"),
	];
	for (const m of order) {
		const k = keys.find(m);
		if (k) return k;
	}
	return null;
}

function questHasSupportedTask(x) {
	const cfg = x.config?.taskConfig ?? x.config?.taskConfigV2;
	return pickTaskName(cfg) != null;
}

// userStatus.progress can be a Map in store payloads and a plain object over
// REST; read defensively.
function readProgress(userStatus, key) {
	const p = userStatus?.progress;
	const entry = p instanceof Map ? p.get(key) : p?.[key];
	return entry?.value ?? 0;
}

// Stream keys are `call:<dmChannelId>:<ownerId>` for DMs and
// `guild:<guildId>:<channelId>:<ownerId>` for guild voice channels. The old
// `call:<channel>:1` put a random 4-digit number where a user snowflake belongs.
function buildStreamKey() {
	try {
		const ownerId = UserStore?.getCurrentUser?.()?.id ?? getUserIdFromToken();
		if (!ownerId) return null;

		const dm = ChannelStore?.getSortedPrivateChannels?.()?.[0]?.id;
		if (dm) return `call:${dm}:${ownerId}`;

		for (const g of Object.values(GuildChannelStore?.getAllGuilds?.() ?? {})) {
			const vc = g?.VOCAL?.[0]?.channel ?? g?.VOCAL?.[0];
			const guildId = vc?.guild_id ?? g?.id;
			if (vc?.id && guildId) return `guild:${guildId}:${vc.id}:${ownerId}`;
		}
		return null;
	} catch (e) {
		console.log("Stream key lookup failed:", e);
		return null;
	}
}

function getUserIdFromToken() {
	try {
		const b64 = token.split(".")[0].replace(/-/g, "+").replace(/_/g, "/");
		const padded = b64.padEnd(Math.ceil(b64.length / 4) * 4, "=");
		return atob(padded);
	} catch (e) {
		return null;
	}
}

async function getQuests() {
	if (QuestsStore?.quests) {
		let quests = [...QuestsStore.quests.values()].filter(
			(x) =>
				x.userStatus?.enrolledAt &&
				!x.userStatus?.completedAt &&
				new Date(x.config.expiresAt).getTime() > Date.now() &&
				questHasSupportedTask(x)
		);
		if (quests.length > 0) return quests;
	}

	console.log("QuestsStore failed or empty, trying API...");

	let res = await fetch("https://discord.com/api/v9/quests/@me", {
		headers: { Authorization: token },
	});
	let allQuests = await res.json();

	return allQuests.filter(
		(x) =>
			x.userStatus?.enrolledAt &&
			!x.userStatus?.completedAt &&
			new Date(x.config.expiresAt).getTime() > Date.now() &&
			questHasSupportedTask(x)
	);
}

(async () => {
	let quests = await getQuests();

	if (quests.length === 0) {
		console.log("You don't have any uncompleted quests!");
		return;
	}

	console.log("Found quests:", quests.map((q) => q.config.messages.questName));

	let doJob = async function () {
		const quest = quests.pop();
		if (!quest) return;

		const pid = Math.floor(Math.random() * 30000) + 1000;
		const questName = quest.config.messages.questName;
		const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
		const taskName = pickTaskName(taskConfig);
		const secondsNeeded = taskConfig.tasks[taskName]?.target ?? 0;
		let secondsDone = readProgress(quest.userStatus, taskName);

		// FIXED: app id/name moved off config.application onto the task
		const applicationId = appIdFor(taskConfig, taskName, quest.config?.application?.id);
		const applicationName = quest.config?.application?.name ?? quest.config?.messages?.questName ?? "Unknown App";

		if (!taskName || !secondsNeeded) {
			console.log(`Skipping "${questName}": unsupported or missing task config.`);
			doJob();
			return;
		}

		if (!applicationId && taskName !== "WATCH_VIDEO" && taskName !== "WATCH_VIDEO_ON_MOBILE") {
			console.log(`Skipping "${questName}": no application id in its config, Discord couldn't match a spoofed process/stream to it.`);
			doJob();
			return;
		}

		if (taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
			const maxFuture = 10, speed = 7, interval = 1;
			const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime();
			let completed = false;
			console.log(`Spoofing video for ${questName}.`);

			while (true) {
				const maxAllowed = Math.floor((Date.now() - enrolledAt) / 1000) + maxFuture;
				const diff = maxAllowed - secondsDone;
				const timestamp = secondsDone + speed;
				if (diff >= speed) {
					const res = await api.post({ url: `/quests/${quest.id}/video-progress`, body: { timestamp: Math.min(secondsNeeded, timestamp + Math.random()) } });
					completed = res.body.completed_at != null;
					secondsDone = Math.min(secondsNeeded, timestamp);
				}
				if (timestamp >= secondsNeeded) break;
				await new Promise((resolve) => setTimeout(resolve, interval * 1000));
			}
			if (!completed) {
				await api.post({ url: `/quests/${quest.id}/video-progress`, body: { timestamp: secondsNeeded } });
			}
			console.log("Quest completed!");
			doJob();

		} else if (taskName.startsWith("PLAY") && taskName !== "PLAY_ACTIVITY") {
			if (!isApp) {
				console.log("This needs the desktop app for", questName);
				doJob();
				return;
			}

			let exeName = `${applicationName}.exe`;
			let appName = applicationName;

			try {
				let res = await api.get({ url: `/applications/public?application_ids=${applicationId}` });
				const appData = res.body?.[0];
				if (appData) {
					appName = appData.name || applicationName;
					const foundExe = appData.executables?.find((x) => x.os === "win32");
					if (foundExe) {
						exeName = foundExe.name.replace(">", "");
					}
				}
			} catch (e) {
				console.log("Failed to fetch application data, using fallback values.", e);
			}

			const fakeGame = {
				cmdLine: `C:\\Program Files\\${appName}\\${exeName}`,
				exeName,
				exePath: `c:/program files/${appName.toLowerCase()}/${exeName}`,
				hidden: false,
				isLauncher: false,
				id: applicationId,
				name: appName,
				pid: pid,
				pidPath: [pid],
				processName: appName,
				start: Date.now(),
			};

			let realGetRunningGames, realGetGameForPID;
			if (RunningGameStore && FluxDispatcher) {
				const realGames = RunningGameStore.getRunningGames();
				realGetRunningGames = RunningGameStore.getRunningGames;
				realGetGameForPID = RunningGameStore.getGameForPID;
				RunningGameStore.getRunningGames = () => [fakeGame];
				RunningGameStore.getGameForPID = (p) => [fakeGame].find((x) => x.pid === p);
				FluxDispatcher.dispatch({ type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: [fakeGame] });
			}

			console.log(`Spoofed game to ${appName}. Sending heartbeats every 20s for ~${Math.ceil((secondsNeeded - secondsDone) / 60)} minutes.`);

			const streamKey = buildStreamKey() ?? "call:0:0";

			while (true) {
				try {
					let hbRes = await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, application_id: String(applicationId ?? ""), terminal: false } });
					let progress = hbRes.body?.progress?.[taskName]?.value ?? secondsDone;
					secondsDone = progress;
					console.log(`Quest progress: ${Math.floor(progress)}/${secondsNeeded}`);

					if (progress >= secondsNeeded) {
						await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, application_id: String(applicationId ?? ""), terminal: true } });
						break;
					}
				} catch (e) {
					console.log("Heartbeat error:", e);
				}
				await new Promise((resolve) => setTimeout(resolve, 20 * 1000));
			}

			if (RunningGameStore && FluxDispatcher && realGetRunningGames) {
				RunningGameStore.getRunningGames = realGetRunningGames;
				RunningGameStore.getGameForPID = realGetGameForPID;
				FluxDispatcher.dispatch({ type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: [] });
			}

			console.log("Quest completed!");
			doJob();

		} else if (taskName.startsWith("STREAM")) {
			if (!isApp) {
				console.log("This needs the desktop app for", questName);
				doJob();
				return;
			}

			let realFunc;
			if (ApplicationStreamingStore) {
				realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata;
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
					id: applicationId,
					pid,
					sourceName: null,
				});
			}

			const streamKey = buildStreamKey() ?? "call:0:0";

			console.log(`Spoofed stream to ${applicationName}. Sending heartbeats every 20s for ~${Math.ceil((secondsNeeded - secondsDone) / 60)} minutes.`);
			console.log("Remember you need at least 1 other person in the vc!");

			while (true) {
				try {
					let hbRes = await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, application_id: String(applicationId ?? ""), terminal: false } });
					let progress = hbRes.body?.progress?.[taskName]?.value ?? secondsDone;
					secondsDone = progress;
					console.log(`Quest progress: ${Math.floor(progress)}/${secondsNeeded}`);

					if (progress >= secondsNeeded) {
						await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, application_id: String(applicationId ?? ""), terminal: true } });
						break;
					}
				} catch (e) {
					console.log("Heartbeat error:", e);
				}
				await new Promise((resolve) => setTimeout(resolve, 20 * 1000));
			}

			if (ApplicationStreamingStore && realFunc) {
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc;
			}

			console.log("Quest completed!");
			doJob();

		} else if (taskName === "PLAY_ACTIVITY") {
			const streamKey = buildStreamKey() ?? "call:0:0";

			console.log("Completing quest", questName);

			while (true) {
				try {
					const res = await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, application_id: String(applicationId ?? ""), terminal: false } });
					const progress = res.body?.progress?.[taskName]?.value ?? res.body?.progress?.PLAY_ACTIVITY?.value ?? secondsDone;
					console.log(`Quest progress: ${progress}/${secondsNeeded}`);

					if (progress >= secondsNeeded) {
						await api.post({ url: `/quests/${quest.id}/heartbeat`, body: { stream_key: streamKey, application_id: String(applicationId ?? ""), terminal: true } });
						break;
					}
				} catch (e) {
					console.log("Heartbeat error:", e);
				}
				await new Promise((resolve) => setTimeout(resolve, 20 * 1000));
			}

			console.log("Quest completed!");
			doJob();
		}
	};
	doJob();
})();
```
</details>

# Videos may not be supported anymore
### (If you're unable to paste into the console, you might have to type `allow pasting` and hit enter)

5. Follow the printed instructions depending on what type of quest you have
    - If your quest says to "play" the game or watch a video, you can just wait and do nothing
    - If your quest says to "stream" the game, join a vc with a friend or alt and stream any window
7. Wait a bit for it to complete the quest
8. You can now claim the reward!

You can track the progress by looking at the `Quest progress:` prints in the Console tab, or by looking at the progress bar in the quests tab.

## FAQ

**Q: Running the script does nothing besides printing "undefined", and makes chat messages not go through**

A: This is a random bug with opening devtools, where all http requests break for a few minutes. It's not the script's fault. Either wait and try again, or restart discord and try again.

**Q: Can I get banned for using this?**

A: There is always a risk, though so far nobody has been banned for this or other similar things like client mods. (Very unlikely for a ban to happen)


**Q: Ctrl + Shift + I doesn't work**

A: Either download the [ptb client](https://discord.com/api/downloads/distributions/app/installers/latest?channel=ptb&platform=win&arch=x64), or use [this](https://www.reddit.com/r/discordapp/comments/sc61n3/comment/hu4fw5x/) to enable DevTools on stable.


**Q: Ctrl + Shift + I takes a screenshot**

A: Disable the keybind in your AMD Radeon app.


**Q: I get a syntax error/unexpected token error**

A: Make sure your browser isn't auto-translating this website before copying the script. Turn off any translator extensions and try again.


**Q: I'm on Vesktop but it tells me that I'm using a browser**

A: Vesktop is not a true desktop client, it's a fancy browser wrapper. Download the actual desktop app instead.


**Q: I get a different error**

A: Make sure you're copy/pasting the script correctly and that you've have done all the steps.


**Q: Can I complete expired quests with this?**

A: No, there is no way to do that.


**Q: Can you make the script auto accept the quest/reward?**

A: No. Both of those actions may show a captcha, so automating them is not a good idea. Just do the two clicks yourself.


**Q: Can you make this a Vencord plugin?**

A: No. The script sometimes requires immediate updates for Discord's changes, and Vencord's update cycle and code review would be too slow for that. There are some Vencord forks which have implemented this script or their own quest completers if you really want one.


**Q: Can you upload the standalone script to a repo and make this gist's code a one line fetch()?**

A: No. Doing that would put you at risk because I (or someone in my account) could change the underlying code to be malicious at any time, then forcepush it away later, and you'd never know.
