How to open URLs/files in other VMs
====================================

This document shows how to automatically open files/attachments/URLs in another VM, with or without user confirmation. This setup particularly suits "locked down" setups with restrictive firewalls like VMs dedicated to emails.

There are quite a few approaches that one can choose to open files, each with their pros and cons. However the mechanism is the same for all of them: they use the `qubes.OpenInVM` and `qubes.OpenURL` [RPC services](https://www.qubes-os.org/doc/qrexec3/#qubes-rpc-services) (usually through the use of the `qvm-open-in-vm` and `qvm-open-in-dvm` scripts).

In case dom0 permissions (see section below) allow opening URLs/files in the destination VM without user confirmation but different destination VMs have to be used (eg. depending on the site's level of trust, URL/file type, ...), a custom wrapper to the `qvm-open-in-vm` script can be used to select a specific destination VM based on the file/URL type.

Naming convention:

- `srcVM` is the VM where the files/URLs are
- `dstVM` is the VM we want to open them in ; `dstVM` can be any VM type - a DispVM, a regular AppVM, a Whonix dvm, ...


Configuring dom0 RPC permissions
--------------------------------

When using `qvm-open-in-{vm,dvm}` scripts (which in turn use the `qubes.OpenInVM` and `qubes.OpenURL` RPC calls), one may choose if/when a user confirmation dialog should pop up, depending on the RPC call and the `srcVM` / `dstVM` combo. See the [official doc](https://www.qubes-os.org/doc/rpc-policy/) for the proper syntax.


Configuring `srcVM`
-------------------

### Command-line ###

Save for copy/pasting URLs between VMs, the most basic - and less convenient - approach is to open files or URLs like so:

~~~
qvm-open-in-vm dstVM http://example.com
qvm-open-in-vm dstVM word.doc
~~~

Or, if opening in random dispVMs:

~~~
qvm-open-in-dvm http://example.com
qvm-open-in-dvm word.doc
~~~

Note: `qvm-open-in-dvm` is actually a wrapper to `qvm-open-in-vm`.


### Per application setup ###

Most applications provide a way to configure what program to use depending on URL/file (mime) types. Stepping up from the command line approach, a better solution would be to configure each application to use the `qvm-open-in-{vm,dvm}` scripts. 

The subsections below give additional info on how to configure popular applications.


#### Thunderbird ####

In the case of Thunderbird, one has to define actions for opening attachements (see the [mozilla doc](http://kb.mozillazine.org/Actions_for_attachment_file_types), mainly section "Download Actions" settings"). Changing the way http and https URLs are opened requires tweaking config options though (see [this mozilla doc](http://kb.mozillazine.org/Changing_the_web_browser_invoked_by_Thunderbird)). Those changes can be made in Thunderbird's config editor, or by adding the following to `$HOME/.thunderbird/user.js` like so:

~~~
user_pref("network.protocol-handler.warn-external.http", true);
user_pref("network.protocol-handler.warn-external.https", true);
// http://kb.mozillazine.org/Network.protocol-handler.expose-all
user_pref("network.protocol-handler.expose-all", true);
~~~

Thunderbird will then ask which program to use the next time a link is opened. If `dstVM` should be a regular dispVM, choose `qvm-open-in-dvm`. Otherwise you'll have to create a wrapper since arguments cannot be passed to the program in Thunderbird's dialog. For instance, put the following in `$HOME/bin/thunderbird-url`, make it executable, and choose that script:

~~~
#!/bin/sh
qvm-open-in-vm dstVM "$@"
~~~


#### Firefox, Chrome/Chromium ####

Those browsers offer an option to define programs associated to a file (Mime) type but a flexible alternative is to use Raffaele Florio's [qubes-url-redirector](https://github.com/raffaeleflorio/qubes-url-redirector) add-on: links can be opened with a context menu and the add-on has a settings page embedded in the browser to customize its default behavior, with support for whitelist regexes.

The qubes-url-redirector add-on will likely be included officialy in the next Qubes release (see [this](https://github.com/QubesOS/qubes-issues/issues/3152) issue), easing concerns about installing third-party software. The addon may also support Thunderbird in the future.


#### Vi ####

Put the following in `$HOME/.vimrc` to open URLs in `dstVM` (type `gx` when the cursor is over an URL):

~~~
let g:netrw_browsex_viewer = 'qvm-open-in-vm dstVM'
~~~


### Application independent setup ###

The section above relied on configuring *each* application; while it provides a good amount of flexibility, it is time consuming and might be overkill when the same action/program should be used by all the applications in `srcVM`.

Providing that the applications adhere to the freedesktop standard, defining a global action is straightforward:

- put the following in `~/.local/share/applications/browser_vm.desktop`

	~~~
	[Desktop Entry]
	Encoding=UTF-8
	Name=BrowserVM
	Exec=qvm-open-in-vm dstVM %u
	Terminal=false
	X-MultipleArgs=false
	Type=Application
	Categories=Network;WebBrowser;
	MimeType=x-scheme-handler/unknown;x-scheme-handler/about;text/html;text/xml;application/xhtml+xml;application/xml;application/vnd.mozilla.xul+xml;application/rss+xml;application/rdf+xml;image/gif;image/jpeg;image/png;x-scheme-handler/http;x-scheme-handler/https;
	~~~

- set xdg's "default browser" to the .desktop entry you've just created with `xdg-settings set default-web-browser browser_vm.desktop`

The same can be done with any Mime type (see `man xdg-mime` and `xdg-settings`).

Note again that `qvm-open-in-vm dstVM` can be replaced by a user written wrapper with custom logic for selecting a specific dstVM depending on the URL/file type, site level of trust, ...


**Caveat**: if dom0 default permissions are set to allow without user confirmation applications can leak data through the URL name despite `srcVM`'s restrictive firewall (you may notice that an URL has been open in `dstVM` but it would be too late).


"Semi-permanent" named dispVMs
------------------------------

Opening things in dispVMs is the most secure approach, but the long starting time of dispVMs often gets in the way so users end up opening files/URLs in persistent VMs. An intermediate solution is to create a "semi-permanent" dispVM like so (replace `fedora-28-dvm` with the dvm template you want to use):

~~~
qvm-create -C DispVM -t fedora-28-dvm -l red dstVM
~~~

This VM works like a regular VM, with the difference that its private disk is wiped after it's powered off. However it doesn't "auto power off" like random dispVMs so it's up to the user to power off (and optionaly restart) the VM when he/she deems necessary.


Further considerations/caveats of using dispVMs
-----------------------------------------------

Obviously, using dispVMs as `dstVM` means that changes are lost when `dstVM` is powered off so the increased security of this setup makes saving deliberate changes harder.

- inter-VM copy/paste is probably the easiest way to synchronize passwords and bookmarks between `dstVM` and `srcVM` (or another dedicated secure VM like the oft-used 'vault' VM). The following solutions are for instance popular:
   - manage passwords with KeepassX (or one of its forks).
   - manage bookmarks with a plain html file (that most browsers can export/import) or use a dedicated bookmark manager like [buku](https://github.com/jarun/Buku) (available in Fedora 28 repo - `dnf install buku`).
- any change that cannot be copy/pasted easily will require updating `dstVM`'s template. Care must be taken not to replicate compromised files: working with a freshly started `dstVM` and performing only the required update actions before synchronizing files with the templateVM is a good idea.


`Contributors/Credits:` @Aekez, @raffaeleflorio, [Micah Lee](https://micahflee.com/2016/06/qubes-tip-opening-links-in-your-preferred-appvm/), @taradiddles
