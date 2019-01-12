# Gmailieer

<img src="doc/demo.png">

This program can pull email and labels (and changes to labels) from your GMail
account and store them locally in a maildir with the labels synchronized with a
[notmuch](https://notmuchmail.org/) database. The changes to tags in the
notmuch database may be pushed back remotely to your GMail account.

## Disclaimer

Gmailieer will not and can not:

* Add or delete messages on your remote account (except syncing the `trash` or `spam` label to messages, and those messages will eventually be [deleted](https://support.google.com/mail/answer/7401?co=GENIE.Platform%3DDesktop&hl=en))
* Modify messages other than their labels

While Gmailieer has been used to successfully synchronize millions of messages and tags by now, it comes with **NO WARRANTIES**.

## Requirements

* Python 3
* `notmuch >= 0.25` python bindings
* `google_api_python_client` (sometimes `google-api-python-client`)
* `oauth2client`
* `tqdm` (optional - for progress bar)

## Installation

After cloning this repository, symlink `gmi` to somewhere on your path, or use `python setup.py`.

# Usage

This assumes your root mail folder is in `~/.mail` and that this folder is _already_ set up with notmuch.

1. Make a directory for the gmailieer storage and state files (local repository).

```sh
$ cd    ~/.mail
$ mkdir account.gmail
$ cd    account.gmail/
```

All commands should be run from the local mail repository unless otherwise specified.


2. Ignore the `.json` files in notmuch. Any tags listed in `new.tags` will be added to newly pulled messages. Process tags on new messages directly after running gmi, or run `notmuch new` to trigger the `post-new` hook for [initial tagging](https://notmuchmail.org/initial_tagging/). The `new.tags` are not ignored by default if you do not remove them, but you can prevent custom tags from being pushed to the remote by using e.g. `gmi set --ignore-tags-local new`.

```
[new]
tags=new
ignore=*.json;
```

3. Initialize the mail storage:

```sh
$ gmi init your.email@gmail.com
```

`gmi init` will now open your browser and request limited access to your e-mail.

> The access token is stored in `.credentials.gmailieer.json` in the local mail repository. If you wish, you can specify [your own api key](#using-your-own-api-key) that should be used.

4. You're now set up, and you can do the initial pull.

> Use `gmi -h` or `gmi command -h` to get more usage information.

## Pull

will pull down all remote changes since last time, overwriting any local tag
changes of the affected messages.

```sh
$ gmi pull
```

the first time you do this, or if a full synchronization is needed it will take longer.

## Push

will push up all changes since last push, conflicting changes will be ignored
unless `-f` is specified. these will be overwritten with the remote changes at
the next `pull`.

```sh
$ gmi push
```

## Normal synchronization routine

```sh
$ cd ~/.mail/account.gmail
$ gmi sync
```

This effectively does a `push` followed by a `pull`. Any conflicts detected
with the remote in `push` will not be pushed. After the next `pull` has been
run the conflicts should be resolved, overwriting the local changes with the
remote changes. You can force the local changes to overwrite the remote changes
by using `push -f`.

> Note: If changes are being made on the remote, on a message that is currently being synced with `gmailieer`, the changes may be overwritten or merged in weird ways.

See below for more [caveats](#caveats).

# Settings

Gmailieer can be configured using `gmi set`. Use without any options to get a list of the current settings as well as the current history ID and notmuch revision.

**`Account`** is the GMail account the repository is synced with. Configured during setup with [`gmi init`](#usage).

**`historyId`** is the latest synced GMail revision. Anything since this ID will be fetched on the next [`gmi pull`](#pull) (partial).

**`lastmod`** is the latest synced Notmuch database revision. Anything changed after this revision will be pushed on [`gmi push`](#ush).

**`Timeout`** is the timeout in seconds used for the HTTP connection to GMail. `0` means the forever or system error/timeout, [whichever occurs first](https://github.com/gauteh/gmailieer/issues/83#issuecomment-396487919).

**`File extension`** is an optional argument to include the specified extension in local file names (e.g., `mbox`) which can be useful for indexing them with third-party programs.  

*Important:* If you change this setting after synchronizing, the best case scenario is that all files will apear to not have being pulled down and will be re-downloaded (and duplicated with a different extension in the maildir). There might also be changes to tags. You should in theory be able to change it by renaming all files, but since this will update the lastmod you will get a check on all files.

**`Drop non existing labels`** can be used to silently ignore errors where GMail gives us a label identifier which is not associated with a label. See [Caveats](#caveats).

**`Replace slash with dot`** is used to replace the sub-label separator (`/`) with a dot (`.`). I think this is easier to work with. *Important*: See note below on [changing this setting after initial sync](#changing-ignored-tags-and-translation-after-initial-sync).

**`Ignore tags (local)`** can be used to specify a list of tags which should not be synced from local to remote (e.g. [`new`](#usage)). In addition to the user-configured tags these tags are ignored: `'attachment', 'encrypted', 'signed', 'passed', 'replied', 'muted', 'mute', 'todo', 'Trash', 'voicemail'`. Some are special tags in notmuch and some are unsupported by GMail. See [Caveats](#caveats) below for more explanations. *Note:* This setting expects [_translated_ tags](#translation-between-labels-and-tags).
  
  *Important*: See note below on [changing this setting after initial sync](#changing-ignored-tags-and-translation-after-initial-sync).

**`Ignore tags (remote)`** can be used to specify a list of tags (labels) which should not be synced from remote (GMail) to local. By default the [`CATEGORY_*` type](https://developers.google.com/gmail/api/guides/labels) labels which are mapped to the Personal/Promotions/etc tabs in the GMail interface are ignored. You can specify that no label should ignored by doing: `gmi set --ignore-tags-remote ""`. *Note:* This setting expects [_*un*translated_ tags](#translation-between-labels-and-tags).
  
  *Important*: See note below on [changing this setting after initial sync](#changing-ignored-tags-and-translation-after-initial-sync).


## Changing ignored tags and translation after initial sync

If you change the [ignored tags](#settings) after the initial sync this will not update already synced messages. This means that if a change is made locally on an already synced message the previously ignored remote labels may be deleted. Conversely, if a change occurs remotely on a message which previously which has local tags that were ignored before, these ignored tags may be deleted.

The best way to deal with this is to do a full push or pull after having changed one of the settings. **Do not change both `--ignore-tags-locally` and `--ignore-tags-remote` at the same time.**

Before changing either setting make sure you are fully synchronized. After changing e.g. `--ignore-tags-remote` do first a dry-run and then a real run of full `gmi pull -f --dry-run`. This will fetch the full tag list for all messages and overwrite the local tags of all your messages with the remote labels.

When changing the opposite setting: `--ignore-tags-local`, do a full push (dry-run first): `gmi push -f --dry-run`.

The same goes for the option `--replace-slash-with-dot`. I prefer to do `gmi pull -f --dry-run` after changing this option. This will overwrite the local tags with the remote labels.

# Translation between labels and tags

We translate some of the GMail labels to other tags. The map of labels to tags are:

```py
  'INBOX'     : 'inbox',
  'SPAM'      : 'spam',
  'TRASH'     : 'trash',
  'UNREAD'    : 'unread',
  'STARRED'   : 'flagged',
  'IMPORTANT' : 'important',
  'SENT'      : 'sent',
  'DRAFT'     : 'draft',
  'CHAT'      : 'chat',

  'CATEGORY_PERSONAL'     : 'personal',
  'CATEGORY_SOCIAL'       : 'social',
  'CATEGORY_PROMOTIONS'   : 'promotions',
  'CATEGORY_UPDATES'      : 'updates',
  'CATEGORY_FORUMS'       : 'forums',
```

# Using your own API key

Gmailieer ships with an API key that is shared openly, this key shares API quota, but [cannot be used to access data](https://github.com/gauteh/gmailieer/pull/9) unless access is gained to your private `access_token` or `refresh_token`.

You can get an [api key](https://console.developers.google.com/flows/enableapi?apiid=gmail) for a CLI application to use for yourself. Store the `client_secret.json` file somewhere safe and specify it to `gmi auth -c`. You can do this on a repository that is already initialized.


# Caveats

* The GMail API does not let you sync `muted` messages. Until [this Google
bug](https://issuetracker.google.com/issues/36759067) is fixed, the `mute` and `muted` tags are not synchronized with the remote.

* The [`todo`](https://github.com/gauteh/gmailieer/issues/52) and [`voicemail`](https://github.com/gauteh/gmailieer/issues/74) labels seem to be reserved and will be ignored.

* The `draft` and `sent` labels are read only: They are synced from GMail to local notmuch tags, but not back (if you change them via notmuch).

* [Only one of the tags](https://github.com/gauteh/gmailieer/issues/26) `inbox`, `spam`, and `trash` may be added to an email. For
the time being, `trash` will be prefered over `spam`, and `spam` over `inbox`.

* `Trash` (capital `T`) is reserved and not allowed, use `trash` (lowercase, see above) to bin messages remotely.

* Sometimes GMail provides a label identifier on a message for a label that does not exist. If you encounter this [issue](https://github.com/gauteh/gmailieer/issues/48) you can get around it by using `gmi set --drop-non-existing-labels` and re-try to pull. The labels will now be ignored, and if this message is ever synced back up the unmapped label ID will be removed. You can list labels with `gmi pull -t`.

* You [cannot add any new files](https://github.com/gauteh/gmailieer/issues/54) (files starting with `.` will be ignored) to the gmailieer repository. Gmailieer uses the directory content an index of local files. Gmailieer does not push new messages to your account (note that if you send messages with GMail, GMail automatically adds the message to your mailbox).

* Make sure that you use the same domain for you GMail account as you initially created your account with: usually `@gmail.com`, but sometimes `@googlemail.com`. Otherwise you might get a [`Delegation denied` error](https://github.com/gauteh/gmailieer/issues/88).

