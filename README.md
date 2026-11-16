# my-appCleaner

Uninstall a macOS app with everything it put on the machine, from the command
line: the app, its launchd jobs and helpers, its files in every user's
`~/Library` and in `/Library`, its extensions and its installer receipts.

- `show` lists what an uninstall removes, and why each item is there. It
  changes nothing.
- `rm ... go` removes it all to the owner's Trash. It never deletes.
- `intel` lists what has no Apple-silicon code (apps, bundles, loose programs,
  kexts), each ready to hand to `rm`.
- `leftovers` lists Library files whose app is gone.

It runs unattended, as root, from scripts. That is why it exists: AppCleaner
does its search in its GUI only. Its one command-line program, `achelper`,
only trashes or restores the paths it is given.

One file, Python 3.9+ standard library only. Copy `my-appCleaner` anywhere on
your PATH.

## Usage

    usage: my-appCleaner [-Q] [-V] [-D [PATH]] [--config <FILE>] [--create-config [FILE]]
                         [--run-tests [FILTER]] [--version] [-h] <VERB> ...

    verbs:
      show <APP>...          what an uninstall of <APP> removes, and why each item is
                             there -- changes nothing (also what a bare <APP> does)
      rm <APP>... go         remove it: Homebrew casks by brew, launchd jobs booted out,
                             processes ended, then everything moved to its owner's
                             Trash; receipts forgotten. Without go: what show prints
      intel [<DIR>...]       what has no Apple-silicon (arm64) code -- apps, bundles,
                             loose programs, kexts -- by owner, marked when it runs now
                             or starts; each line is an <APP> for `rm` (look only;
                             <DIR> narrows it, default: INTEL_ROOTS and every user's own)
      leftovers              files in the Library folders whose app is gone (look only)

`my-appCleaner --help` prints the full text, with the options, the config
search order and the exit codes.

An `<APP>` can be:

- an `.app` path;
- a bundle id (`com.example.app`);
- an installer package id (`com.example.pkg.app`);
- a folder of apps (`/Applications/Example Suite`: every app in it);
- any other path `intel` or `leftovers` lists (a loose program, a kext, a
  folder): then it, its receipt and its launchd jobs.

## What it looks at, and why an item is matched

For an app's bundle id, its team id and its installer receipts:

| | where |
|---|---|
| the app itself | a bundle id given as `<APP>` names every copy with that id |
| launchd jobs | `/Library/Launch{Agents,Daemons}`, every `~/Library/LaunchAgents` |
| helpers | `/Library/PrivilegedHelperTools`; login items are named (macOS forgets them once the item is gone) |
| extensions | Extensions (kexts), Audio plug-ins, QuickLook, Spotlight, PreferencePanes, Printers, Image Capture, Input Methods, Internet Plug-Ins, Screen Savers, Services |
| Library files, every user | Application Support, Caches, Preferences, Containers, Group Containers, Saved Application State, HTTPStorages, WebKit, Logs, Cookies, Application Scripts |
| installer receipts | `pkgutil`: every file the package installed, then `--forget` |

Each item says which of these matched it, in brackets:

- `[bundle id]`: the item's name is the bundle id, or the bundle id followed
  by a dot (`com.example.App.plist`, `com.example.App.savedState`).
- `[team id]`: the item is signed by the app's team (Group Containers are
  named after it). An extension never goes by its team id alone.
- `[receipt <id>]`: a package installed it.
- `[program in item]`: a launchd job whose program lies inside an item. A
  job goes with its program, its receipt or its bundle id -- never by team id
  alone: a job whose program stays is kept.

A name alone matches nothing: too many apps share words like "Scan". The
other rules exist so that removing one app never takes another's files:

- **Another copy**: an app with the same bundle id that is not named stays
  (an old version beside a new one), and so do the files of that id. Naming
  the bundle id itself removes every copy.
- **A shared team id**: a team id matches only while nothing else it signs
  stays -- no other app, and no extension (a scanner driver, a kext) that no
  bundle id or receipt of the removal matched. Those keep what only the team
  id would have matched, and `show` says so.
- **A shared folder**: a receipt's file in a folder that also holds files of
  no receipt being removed is kept. Its receipt is then not forgotten.
- **Never removed**: macOS's own files (`/System`, `com.apple.*`), a folder
  things are installed into (`/Applications`, `/Library/Preferences`, a home,
  ...), or anything holding one.

A Homebrew cask's app is removed by `brew uninstall --zap --cask <cask>`. The
cask's zap list holds the vendor-specific knowledge, and Homebrew keeps it up
to date. As root, brew runs as the user who owns it.

## Examples

    $ my-appCleaner show "/Applications/Example Software"
     >>> 2 app(s), 1 item(s), team id ABCDE12345, 3 receipt(s)
       > /Applications/Example Software                                     [given]
       > /Applications/Example Software/Event Manager.app                   [in the folder]
       > /Applications/Example Software/Example Manuals.app                 [in the folder]
     >>> launchd: 1 job(s), running now: 1
       > /Library/LaunchAgents/com.example.eventmanager.agent.plist         [program in item; running]
     >>> extensions: 1 (/Library 1), 1.2 MB
       > /Library/Extensions/ExampleRAID.kext                               [receipt com.example.pkg.raid]
     >>> Library files: 5 (/Library 1, alice 3, bob 1), 212.0 MB
       > ~alice/Library/Preferences/com.example.Event-Manager.plist         [bundle id]
       > ~alice/Library/Group Containers/ABCDE12345.shared                  [team id ABCDE12345]
       ...
     >>> receipts: 3 package(s), all their files covered above
     >>> nothing changed -- `my-appCleaner rm '/Applications/Example Software' go` removes it

    $ my-appCleaner rm "/Applications/Example Software" go
    ERROR 022 [1]: needs root for 8 of 9 item(s) and 3 receipt(s) to forget -- nothing changed; run: sudo my-appCleaner rm '/Applications/Example Software' go

    # as root: the plan as above, then
     >>> launchctl bootout gui/501/com.example.eventmanager.agent   [rc 0]
     >>> ended 1 process(es): Event Manager [pid 4242]
     >>> 9 item(s) to Trash: alice 5, root 4 (214.4 MB)
     >>> pkgutil --forget com.example.pkg.raid ...   [3 forgotten]
     >>> done: 9 removed [rc 0]

    $ my-appCleaner intel
     >>> 4 owner(s) with no arm64 code (1 of them an arm64 app with Intel parts) (4 453 executable file(s) typed of 206 263 looked at)
       > x86_64     /Applications/Example Scan.app                               [runs now, login item]
       > app arm64  /Applications/Big Suite.app (3 Intel part(s))
       > x86_64     /Library/Extensions/ExampleRAID.kext                         [never loads on Apple silicon]
       > x86_64     /usr/local/bin/example-tool                                  [launchd: com.example.tool]
     >>> look only -- remove one: my-appCleaner rm <the path> go

`intel` types every executable file with `file(1)`, in batches, reading its
output as bytes (`LC_ALL=C`, because descriptions can hold any encoding). It
takes the outermost bundle around a file as its owner. With `-V`, each owner's
Intel files are listed too.

## The Trash, root, the log

- Each item goes to its owner's Trash: root's for root's files
  (`/var/root/.Trash`). An item on another volume goes to that volume's
  `.Trashes`. Nothing is ever deleted, nor copied and then deleted.
- `rm ... go` needs root for anything outside your own home. Without root it
  prints the exact command and stops, changing nothing. `show` without root
  cannot see other users' Library folders, and says so.
- `LOG` keeps one line per item moved, with where it came from and where it
  went. That is what you need to put an item back.

## Config

A config file is required: plain `KEY="value"` lines, read and never run.
`my-appCleaner --create-config` prints the default config. The keys are:

- the commands it calls (`FILE`, `PKGUTIL`, `LAUNCHCTL`, `CODESIGN`, `PS`,
  `SFLTOOL`, `BREW`), with their standard paths as defaults;
- `INTEL_ROOTS`, `INTEL_USER_DIRS`, `INTEL_PRUNE` (lists separated by `|`);
- `LOG`;
- the defaults of `-Q`/`-V`/`-D`.

Search order (the first found wins):

1. `$MY_APPCLEANER_CONFIG`
2. `--config <FILE>`
3. the site's config folder, when `/LINKS/default/site.conf` names one
4. `~/.my-appCleaner.conf`
5. `/etc/my-appCleaner.conf`
6. `/usr/local/etc/my-appCleaner.conf`

With no config found, it lists where to write one.

`/LINKS/default/site.conf` is optional. It is a file of plain `KEY="value"`
lines shared by a site's tools (`LOG_BASE_DIR`, `CONFIG_SEARCH_DIR`, ...).
Without it, the neutral values apply (`/var/log`, no search folder).

## Exit status, tests

- **Exit status**: 0 done (or shown), 1 something could not be removed (each
  item listed) or an error, 2 usage, 70 internal error (one line, with the
  trace in a file it names), 130 Ctrl-C.
- **Tests**: `my-appCleaner --run-tests [FILTER]` runs the built-in tests
  (TAP). Each test builds its own system tree in a temp folder, with stand-ins
  for `pkgutil`, `launchctl`, `codesign`, `ps`, `sfltool`, `brew` and
  `file`. Nothing on the machine is read or changed.
