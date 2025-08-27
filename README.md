# Riffy [![NPM version](https://img.shields.io/npm/v/riffy.svg?style=flat-square&color=informational)](https://npmjs.com/package/riffy)
Riffy is a modern, lightweight, and highly extensible Lavalink client for Node.js. It is meticulously crafted to simplify the development of high-performance music bots by providing a robust and intuitive interface for managing audio streams, queues, and player states. Riffy acts as a high-level abstraction over the Lavalink API, allowing you to focus on your bot's core logic without getting bogged down in low-level voice protocol details.
Table of Contents
 * Key Features
 * Getting Started
   * Prerequisites
   * Installation
 * Core Concepts
 * API & Usage
   * Initialization
   * Basic Commands (!play, !skip, !stop)
 * Configuration
 * Events
 * Troubleshooting
 * Community & Support
 * Our Team
 * License
Key Features
Riffy is engineered to be a comprehensive and flexible solution, offering a suite of features that simplify bot development:
 * Broad Lavalink Protocol Support: Maintains compatibility with both version 3 and version 4 of the Lavalink protocols, ensuring seamless integration with existing and future nodes.
 * Intuitive Autoplay: Provides intelligent autoplay support for major platforms, including YouTube, SoundCloud, and Spotify, ensuring continuous music playback.
 * Universal Discord Library Compatibility: Works effortlessly with any Node.js-based Discord library, such as discord.js and Eris.
 * Comprehensive Filter Support: Access and apply all available Lavalink filters to fine-tune audio output, including equalizer, bassboost, karaoke, and more.
 * Lightweight & Performant: A minimalist design ensures low resource usage and high performance, critical for handling numerous voice connections efficiently.
 * Extensible Architecture: Designed with extensibility in mind, allowing for custom functionality and integrations.
Getting Started
Prerequisites
Before using Riffy, you must have a running Lavalink node. You can find the latest version here or use a free, public node from the official Riffy resources.
Installation
Install Riffy from npm, along with your chosen Discord library.
npm install riffy discord.js

Core Concepts
Understanding these core components will help you build your bot more effectively:
 * Riffy Client: The main client instance that manages all connections to Lavalink nodes, handles events, and provides the primary API for interacting with players.
 * Node: Represents a single, connected Lavalink server. Riffy automatically manages the connection state and load balancing across all configured nodes.
 * Player: A local representation of a Lavalink player. It holds the queue, manages the player state (playing, paused, etc.), and controls the playback for a specific guild.
 * Queue: A data structure within each Player that stores the list of tracks to be played. Riffy's queue handles track addition, removal, and playback order.
API & Usage
Initialization
Initialize the Riffy client and pass your Discord client instance along with your node configurations.
// index.js

const { Client, GatewayDispatchEvents } = require("discord.js");
const { Riffy } = require("riffy");

const client = new Client({
    intents: [
        "Guilds",
        "GuildMessages",
        "GuildVoiceStates",
        "GuildMessageReactions",
        "MessageContent",
        "DirectMessages",
    ],
});

const nodes = [
    {
        host: "localhost",
        password: "youshallnotpass",
        port: 2333,
        secure: false,
    },
];

client.riffy = new Riffy(client, nodes, {
    send: (payload) => {
        const guild = client.guilds.cache.get(payload.d.guild_id);
        if (guild) guild.shard.send(payload);
    },
    defaultSearchPlatform: "ytmsearch",
    restVersion: "v4",
});

client.on("ready", () => {
    client.riffy.init(client.user.id);
    console.log(`Logged in as ${client.user.tag}`);
});

Basic Commands
Here is a comprehensive example demonstrating the !play, !skip, and !stop commands.
client.on("messageCreate", async (message) => {
    if (!message.content.startsWith("!") || message.author.bot) return;

    const args = message.content.slice(1).trim().split(" ");
    const command = args.shift().toLowerCase();
    const query = args.join(" ");

    if (command === "play") {
        if (!message.member.voice.channel) {
            return message.channel.send("You must be in a voice channel to use this command.");
        }

        const player = client.riffy.createConnection({
            guildId: message.guild.id,
            voiceChannel: message.member.voice.channel.id,
            textChannel: message.channel.id,
            deaf: true,
        });

        const resolve = await client.riffy.resolve({
            query: query,
            requester: message.author,
        });

        const { loadType, tracks, playlistInfo } = resolve;

        if (loadType === "playlist") {
            for (const track of resolve.tracks) {
                track.info.requester = message.author;
                player.queue.add(track);
            }
            message.channel.send(
                `Added: \`${tracks.length} tracks\` from \`${playlistInfo.name}\``
            );
        } else if (loadType === "search" || loadType === "track") {
            const track = tracks.shift();
            track.info.requester = message.author;
            player.queue.add(track);
            message.channel.send(`Added: \`${track.info.title}\``);
        } else {
            return message.channel.send("There are no results found.");
        }

        if (!player.playing && !player.paused) {
            player.play();
        }
    } else if (command === "skip") {
        const player = client.riffy.players.get(message.guild.id);
        if (!player || !player.queue.length) {
            return message.channel.send("There is no music to skip.");
        }
        player.stop();
        message.channel.send("Skipped the current track.");
    } else if (command === "stop") {
        const player = client.riffy.players.get(message.guild.id);
        if (!player || !player.queue.length) {
            return message.channel.send("There is no music playing.");
        }
        player.destroy();
        message.channel.send("Stopped the music and cleared the queue.");
    }
});

Configuration
The Riffy constructor accepts an object with the following properties:
| Property | Type | Description |
|---|---|---|
| send | Function | (Required) A function to send voice payloads to Discord. This is critical for Riffy to work. |
| nodes | Array<Object> | (Required) An array of Lavalink node configurations. |
| defaultSearchPlatform | String | The default platform for searching tracks (ytmsearch, ytsearch, scsearch). Defaults to ytmsearch. |
| restVersion | String | The Lavalink REST API version to use. Must be either 'v3' or 'v4'. |
| resumeKey | String | A key to enable session resuming. |
| trackPartial | Boolean | Whether to send partial track data. Defaults to false. |
Events
Riffy extends EventEmitter, providing a number of events to help you manage your bot's state.
| Event | Parameters | Description |
|---|---|---|
| nodeConnect | node | Emitted when a Lavalink node successfully connects. |
| nodeDisconnect | node, reason | Emitted when a node disconnects. reason provides a description. |
| nodeError | node, error | Emitted when a node encounters an error. |
| trackStart | player, track | Emitted when a track starts playing. |
| trackEnd | player, track, reason | Emitted when a track finishes or is stopped. reason gives details. |
| trackError | player, track, error | Emitted when an error occurs during playback. |
| queueEnd | player | Emitted when the queue becomes empty. |
Troubleshooting
 * Bot not joining a voice channel: Ensure your bot has the CONNECT and SPEAK permissions in the channel. Also, double-check that GatewayDispatchEvents.VoiceStateUpdate and GatewayDispatchEvents.VoiceServerUpdate are being handled correctly in your raw event listener.
 * Node "..." encountered an error: Connection refused: This means your bot cannot connect to the specified Lavalink node. Verify the host, port, and password in your nodes configuration. Make sure your Lavalink server is running and accessible from where you're running the bot.
 * No sound from the bot: Check if your bot is deafened. If so, setting deaf: false in createConnection might help. Also, confirm the restVersion in your Riffy configuration matches your Lavalink server's version.
Community & Support
Join our community to get help, share ideas, and connect with other developers building with Riffy.
 * Documentation: riffy.js.org
 * Example Project: Riffy Music Bot
 * Discord Server: discord.gg/TvjrWtEuyP
Our Team
We're a dedicated team of developers committed to creating high-quality, open-source tools for the Discord community.
| Team Member | GitHub | Discord |
|---|---|---|
| 🟪 Elitex | @Elitex07 | @elitex |
| 🟥 FlameFace | @FlameFace | @flameface |
| 🟦 UnschooledGamer | @UnschooledGamer | @unschooledgamer |
License
This project is licensed under the MIT License.
