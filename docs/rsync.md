---

template:    article
reviewed:    2018-09-10
title:       rsync
naviTitle:   rsync
lead:        rsync is one of the best ways to deploy code fast and without hassle. It's also an often overlooked option. Let's change this! This article gives you some direction on how to use it (here on fortrabbit).
group:       deployment
stack:       uni
workInProgress: yes

keywords:
    - rsync

---

<!--
  TODO: 
  * what if rsync creates a loop of changes? test: Git push, rsync down (hopefully not all files) — Tested and there was no prob,

-->

## About rsync

`rsync` is a shorthand for **r**emote **sync**hronization. It's a command line tool to synchronize files over the network. It's open source. It works on top of [SSH](/ssh). And you likely already have it included with your Operating System. Usually **use rsync from your local machine**, not on your fortrabbit App directly. Check if it is already installed like so:

```bash
$ which rsync
```

There are some desktop GUI clients around as well, wrapping `rsync` in a more clickable interface. But don't be afraid of the Terminal, it's easier than you might think.

## Use cases

You have multiple ways to deploy here on fortrabbit: with [Git](/git-deployment), with [SFTP](/sftp) and/or [SSH](/ssh). You can hook in `rsync` in most deployment work-flows, either as an enhancement or as a replacement. It just doesn't really work well with [Professional Apps](/app-pro) as those have ephemeral storage and no direct SSH access. These are your main options for using `rsync` to deploy code to fortrabbit [Universal Apps](/app-uni):


### rsync instead of SFTP

**Still using [SFTP](/sftp)?** Consider `rsync` as a replacement. It's up to ten times faster and more convenient to upload only a few changed files. With SFTP - unless your SFTP client has some kind of synchronization method (which still will be slower) - you copy each files manually, one by one. This is mundane and can also be dangerous when forgetting to copy a single critical file.

`rsync` works as a two way street directly on the file system. Easily synchronize files up and down from your [local development](/local-development) to the [App](/app). 

### rsync in addition to Git

**Using [Git to deploy](/git-deployment)?** Consider `rsync` as an essential addition. Here is why: 

Your dependencies are managed with Composer and thus excluded from Git. They will be installed and managed with [Composer](/composer). So you are keeping your Git repo clean by just including the source files of your code. But your project likely also includes run time data and assets, like compiled CSS, JS and images. Ideally, those binary-like files should be excluded from Git. Deployment is faster when the Git repo is small. Only uncompressed text can be "diffed" anyways. 

Also, there likely will be files, like image uploads and alike on the Apps file system itself which will [not be reflected in Git](deployment-methods-uni#toc-git-works-only-one-way). The only way to get those files back into your local development so far was to SFTP or SSH in and grab the files manually. rsync to the rescue!

PRO TIP: For [Craft CMS](/craft-3-about) users we have developed [Craft Copy](https://github.com/fortrabbit/craft-copy), which besides other nice features, also incorporates rsync to keep the assets in sync.



## Using rsync

Hooked? So let's jump into your local terminal:

### The rsync command structure

```bash
# sync all files UP
# 
#      options
#       ─┴─
$ rsync -av ./ {{app-name}}@deploy.{{region}}.frbit.com:~/
#           ─┬─ ──────────────────┬────────────────────────
#          source           destination
```

* **options**: How to sync, see [below](#toc-options).
* **source**: This is your local source directory.
* **destination**: The target URL, where the files should end up.

### Common usage

Here are the most common rsync commands. Likely you will even come by using only the first two:

```bash
# DOWN: from remote to local
$ rsync -av {{app-name}}@deploy.{{region}}.frbit.com:~/ ./

# UP: from local to remote
$ rsync -av ./ {{app-name}}@deploy.{{region}}.frbit.com:~/



# LOCAL: two local folders
$ rsync -av ~/Projects/{{app-name}}/ ~/Projects/{{app-name}}.copy/

# REMOTE TO REMOTE: from App to App
$ rsync -av {{app-name}}@deploy.{{region}}.frbit.com:~/ {{app-name-2}}@deploy.{{region}}.frbit.com:~/
```

#### Remote paths

Remote URLs consist of `{{user}}@{{host}}:{{folder}}`. In the examples here [fortrabbit App placeholders](access-methods#toc-the-code-example-helper) are used. 


#### Local paths

rsync accepts all kind of defining local paths. `./`  will translate to the current directory. You can also use absolute paths like `/home/my-user/Projects/my-app` or relative paths like `../my-app`.


<style>
  td>code {
      white-space: nowrap;
  }
</style>

### Options

Here are some common options to alter the sync mode:

| Option     | Description   |
|----------- |---------------|
| `-a`       | Shorthand for `--archive`, like the set of options: `-rlptgoD`. |
| `-v`       | Verbose output shows all transmitted files and statistical data. Increase verbosity using `-vv` or `-vvv` |
| `-n`       | Shorthand for `--dry-run`. See [below](#toc-previewing-changes). |
| `-r`       | Recursive, so all files and directories below the source directory. |
| `-l`       | Keep symbolic links. |
| `-p`       | Permissions will be synchronized. |
| `-t`       | Preserve modification times. |
| `-g`       | Set Unix group of file/folder on destination to group in source. Also: use group as check criteria |
| `-o`       | Set Unix group of file/folder on destination to group in source. Also: use group as check criteria |
| `-c`       | Instead of modification time and size, use checksum of the file contents. Use with caution when  modification time on destination is not reliable. |
| `-C`       | Shorthand for `--cvs-exclude`. Exclude version control files like `.git`, `.hg`, `.svn`. |
| `-h`       | Human readable output: display byte sizes in MiB, GiB instead of plain bytes. |
| `-delete`  | Remove unused files. See [below](#toc-dealing-with-obsolete-files) |
| `-exclude` | Omit files from being synced. See [below](#toc-excluding-files) |

For an exhaustive list of all the possible options and more in depth info on the above options, check out the official [rsync man page](https://linux.die.net/man/1/rsync). Mind that rsync options can be chained, `rsync -av` combines the two flags `-a` and `-v`.


#### Previewing changes

The handy `--dry-run` option can be shortened to just `-n` and also be merged with other options to something like `-avn`. It will give you a preview of what will happen before doing anything:

```bash
$ rsync -avn ./ {{app-name}}@deploy.{{region}}.frbit.com:~/
sending incremental file list
./
index.php
wp-content/themes/my-theme/404.php
wp-content/themes/my-theme/archive.php
wp-content/themes/my-theme/content-link.php

sent 39,119 bytes  received 196 bytes  11,232.86 bytes/sec
total size is 23,325,044  speedup is 593.29 (DRY RUN)
```

Now, running this will print out everything that `rsync` **would** transfer - without doing anything. Better always execute a dry run before actually syncing. Once you're sure, that only files which you want to transfer are in the change set, you can just remove the `n` again from the options and execute it normally. 


#### Excluding files

Sometimes you want to omit files from being synced. You can just add `--exclude=path/to/file`. Say we don't want the `404.php` from the previous example to be transferred, you would just do:

```bash
# use absolute path, as viewed from the source root
$ rsync -avC --exclude wp-content/themes/my-theme/404.php ./ my-app@deploy.eu2.frbit.com:~/
```

The value of `--exclude` is a pattern. It can be matched against the files to be transferred. In this case, the following patterns will have the same effect:

```bash
# use partial path
$ rsync -avC --exclude themes/my-theme/404.php ./ my-app@deploy.eu2.frbit.com:~/

# use smallest possible partial path
$ rsync -avC --exclude 404.php ./ my-app@deploy.eu2.frbit.com:~/
```

NOTE: The initial `/` character is important. `--exclude 404.php` and `--exclude /404.php` are _not_ the same. The former means: Any path, which contains "404.php" is to be excluded. The latter means: Any path, which starts with "/404.php" is to be excluded.

##### Advanced exclude patterns

You can also use wildcard characters. For example:

```
$ rsync -av --exclude "*.jpg" --exclude "*.jpeg"` ...
#  exclude all JPEG files

$ rsync -av --exclude "/wp-content/themes/*/404.php" ...
# exclude all files, which start with `/wp-content/themes`, followed by an arbitrary name (no slashes! so only one level of sub directory!) and ending in `404.php`. So basically: All `404.php` files of all themes.

$ rsync -av --exclude "themes/**/*.css" ...
# exclude all files, which path name contains `themes/` then followed by anything (including any amount of sub directories) and ending in `.css`. So all `.css` files in all Themes.
```

Also, As you can see, in the JPEG example, you can add any amount of `--exclude` options to the command.


#### Remembering excludes in a file

If you have a set of files which you always want to exclude, you can create an file containing all exclusions and then use it via `--exclude-from <file>`:

```bash
# add two excludes to a plain text file named "excludes"
$ echo 404.php >> .rsyncignore
$ echo something-else.php >> .rsyncignore

# run rsync, using the excludes file
$ rsync -av --exclude-from .rsyncignore ./ {{app-name}}@deploy.{{region}}.frbit.com:~/
```

NOTE: The file name `.rsyncignore` is just an example name, use any name you want for your excludes.

There is still a lot more you can do with exclude and filtering. Not only is there `--include`, which allows to granulate previous `--exclude` patterns, but there is also `--filter`. Here is a [very interesting blog post by Ira Cooke](http://blog.mudflatsoftware.com/blog/2012/10/31/tricks-with-rsync-filter-rules/).


### Partial syncing

This is the example for using rsync as an addition deployment tool when primarily working with Git. In this case you only want to include a specific folder and it's contents. You might want to "upload" your `dist` folder with your compiled CSS, JS and images. Or you want to "download" the assets folder, when an editor has uploaded new images to the CMS. This is the basic command:

```bash
$ rsync -av --include='/{{folder}}/***' --exclude='*' ./ {{app-name}}@deploy.{{region}}.frbit.com:~/
```

With the `--include` parameter you can specify the path to include, still you need to exclude everything else.

### Dealing with obsolete files

rsync will work in a non-destructive way per default, like our ]overwrite but not delete](deployment-methods-uni#toc-git-push-overwrite-but-not-deletes) strategy when deploying with Git. So new files will be written and replaced, but old files will not get deleted. Sometimes that's not what you want. Sometimes you want an exact copy - a mirror.

So, how to remove obsolete files on the remote? The short answer is: add the option `--delete` to your command line and you are done. To give you and example, using the WordPress setup from before: say you deleted this pesky `404.php` file locally. Now, if you run rsync without the `--delete` option (and no other added or modified files), rsync would tell that it will do nothing:

```
$ rsync -av ./ my-app@deploy.eu2.frbit.com:~/
sending incremental file list
wp-content/themes/my-theme/

sent 39,037 bytes  received 162 bytes  15,679.60 bytes/sec
total size ...
```

Although it marks the folder `wp-content/themes/my-theme/`, as there have been changes (the removal of `404.php`), but no changes which `rsync` is gonna apply. Now, running with the `--delete` option, then the file will be removed from destination. **This is a feature, not a bug**, meaning: rsync won't let you down by deleting files without your say-so.

Either way, the first delete run, as always, using the condensed form `-n` of the `--dry-run` option, will show you exactly what would be deleted:

```bash
$ rsync -avn --delete ./ my-app@deploy.eu2.frbit.com:~/
sending incremental file list
deleting wp-content/themes/my-theme/404.php

sent 39,062 bytes  received 231 bytes  15,717.20 bytes/sec
total size ...
```

After you confirm that `rsync` would only delete, what you want (otherwise: `--exclude` works also to exclude files which are not in your local file set but remote, and you don't want to remove them from remote), you can go ahead and remove the `-n` option and run again.

Now, rsync wouldn't be if it would give you not at least four different ways to handle deletes: Besides the `--delete` flag, there is also `--delete-before`, `--delete-after`, `--delete-during` and `--delete-delay` (and `--delete-excluded`, but that's another special case in it's own). Those four variants of `--delete` just let you control when files are remove. This is actually quite handy: When thinking larger amounts of changed files to a live website, you might want to use `--delete-after` instead of `--delete-before`, so that first all new files are in place, then obsolete files are removed, which makes it more likely that your website is not "interrupted", when handling a request during the synchronization, which relies on files which would be removed.. I think you get the gist.


## Advanced topics

### SSH edge-cases

Should you need to set specific SSH options, for example, if you need to provide a specific private key, then you can use the `--rsh` option, which stands for "remote shell" and can be shortened to `-e`. Here an example:

```bash
# use specific private key
$ rsync -av -e 'ssh -i /path/to/your/key' my-app@deploy.eu2.frbit.com:~/ ./

# enforce password authentication
$ rsync -av -e 'ssh -o PreferredAuthentications=password' my-app@deploy.eu2.frbit.com:~/ ./
```

### How rsync transfers only changes

Say you have changed ten files in your local code set and want to deploy them now. `rsync` will first build a local set of files and directories and a remote set of files and directories. For each item in either set it will generate a check value. This check value, can be either the timestamp of the last change of a file, the size of a file, the current permissions or even a checksum (think MD5) of the file contents. Or any combination of those. Using the `-a` option rsync is gonna use timestamp + file size which is a good balance between performance and accuracy.



## Further readings

This article is loosely based on our still popular [blog post](https://blog.fortrabbit.com/deploying-code-with-rsync) which is even more complete and includes more details. Also of interest:

* [Manual page of rsync](https://linux.die.net/man/1/rsync)
* [How does rsync work](https://rsync.samba.org/how-rsync-works.html)