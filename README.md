# TTObserverV0
Torrent tracker site release watcher, notifier and uploader.
Can:
 - Watch site for new torrent releases (it enumerates serial IDs and if new id it like torrent file - continues)
 - Upload new torrent to transmission-server (previously deleting the previous release)
 - Notify some service (for example telegram) about new release
 
Uploading and notifying are optional, observer can only upload and only notify.
Uses:
 - Bash v4+
 - transmission-show (to parse torrents) and transmission-remote (for uploading)
 - curl, tee, less, GNU awk (!!!), GNU sed (!!!)
 - jq and oathtool (for telegram notification)
 
# Usage
## Quick start
1. Create config directory
2. Delete or rename all `*.plugin` files to `*.plugin_noload`
3. Create `target.config` like `conf.example/target.config` with own values,
3. Run `torrent-bot /path/to/conf/dir`

In such variant, if `transmission_url` is set, observer will search for new releases and upload it to remote transmission server.
If `transmission_url` not set... so, it will do nothing.

**Configuration**

 - out_log_file - file to store error and warning messages
 - err_log_file - file to store info messages
 - target_base_url - base URL and context to try check for new torrent, index (1, 2, 3, 765...) of torrent will be concatenated
 - target_crawl_treshold - count of indexes to check in every check iteration
 - transmission_url - URL of transmission server (localhost, server.local:9091 ...), might be empty if upload is not needed
 - transmission_creds - username:password for auth in transmission server, might be empty
 - transmission_down_dir - temp directory for downloading data from target to check

## Notification with plugins
Basically, observer tries to find all `*.plugin` files, placed in the same directory as `torrent-bot`,
source them and call all functions, that have been registered in `announcers`, `watchers` and `notifiers`
lists by calling `add_announcer`, `add_notifier`, `add_watcher` respectively.

 - `announcers` contains functions to notify about new torrent release
 - `watchers` contains functions to watch for command events in some plugin (just like daemons)
 - `notifiers` cotains functions to notify about some other events (now it's only "anniversary" releases (GETs): 1000th, 2000th etc.)

Every plugin can do ANYthing, but if at least one of `add_*` have not been called, it just do something at initialization moment.

Let's see i.e. of telegram.

### Telegram
Telegram notification implements in `telegram.plugin`, it registers own functions by calling all `add_*` functions.
It requires `jq` and `authtool` utilities. `oathtool` is needed to register some chat or channel in telegram
for recieving announces, notifications and for control commands sunch as force upload some torrent into transmisison,
change next index in torrent tracker to check etc (see below). If you don't need to restrict chats to notify, lust set
```
alias oathtool=true
```

**Configuration**

Telegram plugin uses `telegram.config` file to get initial configuration, 
and `telegram.chats`, `telegram.chats.pending` and `telegram.offset` for service purpose.
Config file should contain next variables:

 - telegram_token - bot token, got from @BotFather bot in telegram
 - telegram_otp_seed - TOTP seed for auth (base32-encoded random bytes)
 - telegram_cmd_tag - /cmd to try determine in group chats, that message is for bot
 - telegram_announce_info_regexp_to_extract - regexp to extract values from transmission-show output and put into announce message
 - telegram_announce_info_regexp_replace - list of rexexps to replace some values from `telegram_announce_info_regexp_to_extract` (i.e. for translation)
 - telegram_announce_template - message Markdown template for announce, format is the same as printf
 - telegram_msg_template - message template for notifications (GET)
 
**Bot commands**

Plugin can understand sommands, that starts with `@NameOfBot:` or `/telegram_cmd_tag@NameOfBot:`.
List of commands:

 - `help` - list available commands
 - `ehlo` - register this chat in pending list (see below)
 - `status` - check if the chat is attached to announce
 - `attach	--otp arg --id|--name arg` - attach chat or channel for announces, `--otp` is the TOTP, `--id` is the telegram ID of chat, `--name` - is friendly name of chat
 - `detach	--otp arg --id|--name arg` - detach chat or channel
 - `offset	--otp arg [--set arg]` - get or change next index to check
 - `download	--otp arg --url arg` - download torrent from `--url` to remote transmission server
 - `list	--otp arg --id|--name arg [--backend arg]` - list all torrents in the remote transmission-server (if `--backend` is set, it tries to list from this URL), `--name` can be regex
 
You can attach chat or channel only if you know TOTP or via two-step verification.

**_Easy attach_**

If you know ID of telegram chat and you have TOTP, just send next text to bot:
```
@NameOfBot attach --id 123454321 --otp 123456
```

**_Two-step attach_**

In this case, there is two person: you, and chat/channel owner.

1. Chat/channel owner sends
```
@NameOfBot ehlo
```
Bot registers _pending_ chat (id and name), it means, that this chat should be verified by you

2. You send 
```
@NameOfBot attach --name name_of_chat --otp 123456
```
Bot tries to find chat by it's name and move it to main chats list.

