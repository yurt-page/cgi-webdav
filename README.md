# cgi-webdav
Essential WebDAV server as a CGI program. You can mount a WebDAV share as a network disk then list files in a folder, add and remove files. 

The main purpose is to make the smallest possible WebDAV server implementation so it can fit into old 4mb routers and to be used with BusyBox httpd or OpenWrt uhttpd.
But if it's size will be very small it may be even included by default to OpenWrt.
This opens an easy way to millions users to set up their own personal cloud by putting a flash drive or disk into their router.
Interesing application may be for a very small embedded devcies that even doesnt't have an SSH server but just a plain Telnet so it's not possible to uplad files into it.

With a simple UI like [webdav-js](https://github.com/dom111/webdav-js) the WebDAV share can used just from a browser.
This will be a similar to NextCloud but use much less resources.

Currently, the WebDAV is always implemented as a web server module e.g. Lighttpd [mod_webdav](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModWebDAV) or Apache Httpd [mod_dav](https://httpd.apache.org/docs/current/mod/mod_dav.html).
The Lighttpd is used in GL.Inet and Turris routers which are powerful enough and using a custom OpenWrt based firmware.
I created an instruction [WebDAV with Lighttpd on OpenWRT](https://gist.github.com/stokito/5db2aa2cc184717d45600889d8115100) and it works perfectly.
But a vanilla OpenWrt uses own uhttpd and there is no such WebDAV module for it.
The BusyBox httpd is not modular at all.
I believe that having just a simple CGI program will be preferable that trying to add the WebDAV functionality into them.

There is already exists the [webdavcgi](https://github.com/DanRohde/webdavcgi) but it's Perl based and too heavy for embedded devices. It looks like it was developed before even Apache mod_dav was developed.

One another interesting project is [webdav-server-rs](https://github.com/miquels/webdav-server-rs) which is a full WebDAV server implemented in Rust.
Still, our main target is embedded devices with MIPS processors that are still not supported by Rust conpiler.

## Design goals
### Only few clients supported
Let's say that the main client that should be supported is Windows Net Share and MacOS Finder.
They are proprietary, even without any ability to report a bug and may behave not by specification.
But most users will use them. That means that instead reading of spec we have to sniff traffic and use them as a reference.
The webdavfs in Linux is probably the second client by priority.
GNOME and KDE uses their own libraries to access WebDAV so they may have their own bugs.
I sniffed traffic that sends GNOME when opening a WebDAV and it's just terrible. There is opened bugs not fixed for many years. 

### Don't to use any dependencies like XML parsers.
That means that things like a custom namespaces may cause it to not work. Ideally just not to parse anything.
For example if in the `PROPFIND` was requested only few fields the webdav.cgi anyway will ignore it and return a full set of fields.

### Don't be a fully compliant to a specification.
For example locks may be not implemented. The [Litmus test](http://www.webdav.org/neon/litmus/) will be failed but anyway it covers 80% of needs.

## See also
* [Awesome WebDAV](https://github.com/fstanis/awesome-webdav)

## Implementation
Discussion for Lighttpd https://redmine.lighttpd.net/boards/2/topics/10872

Here is a PoC project with a webdav server in Go that works without XML parsing and locks and was tested with Windows MiniRedirector 
https://github.com/stokito/go-webdav-server/tree/noxml
It has a mocked pseudo locks because Windows sends them but generally it works!
So now I need to make the plain C implementation.

There should be additional problems: the uhttpd/BysyBox httpd won't forward WebDAV methods (e.g. PROPFIND) into a CGI script. So we need to patch them.
The uclient, OpenWrt's wget clone, also can't be used to send the PROPFIND requests and needs for a patch too.
But basically if just add GET/HEAD request handler then we can turn it into a dedicated server
