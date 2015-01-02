---
layout: post
title: Installing jekyll-import On Windows
---

I already had an updated version of Ruby installed, thanks to [Chocolatey](https://chocolatey.org/), so I went ahead and ran:

```
C:\>gem install jekyll-import
ERROR: Could not find a valid gem 'jekyll-import' (>= 0), here is why:
Unable to download data from https://rubygems.org/ - 
SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed (https://api.rubygems.org/latest_specs.4.8.gz)
```

Clearly something to do with certificates and I had to download the .PEM file from [http://curl.haxx.se/ca/cacert.pem](http://curl.haxx.se/ca/cacert.pem) and run the following command in a command prompt:

```
C:\>set SSL_CERT_FILE=C:\path-to-pem-file\cacert.pem
```

Let's try installing the gem again.

```
C:\>gem install jekyll-import
Fetching: fast-stemmer-1.0.2.gem (100%)
ERROR:  Error installing jekyll-import:
        The 'fast-stemmer' native gem requires installed build tools.

Please update your PATH to include build tools or download the DevKit
from 'http://rubyinstaller.org/downloads' and follow the instructions
at 'http://github.com/oneclick/rubyinstaller/wiki/Development-Kit'
```

Ok, we got past the certificate error, but now we'll need to install the Ruby DevKit. Chocolatey can help us with this:

```
C:\>cinst ruby2.devkit
```

When installed I ran this to update the PATH:

```
C:\>c:\bin\DevKit2\devkitvars.bat
Adding the DevKit to PATH...
```

Finally I was able to install the jekyll-import gem successfully and I had exported some old blog posts from wordpress.com which I wanted to import into Jekyll for testing purposes.

```
C:\>ruby -rubygems -e "require 'jekyll-import'; JekyllImport::Importers::WordpressDotCom.run({ 'source' => 'wordpress.xml', 'no_fetch_images' => false, 'assets_folder' => 'assets' })"


Whoops! Looks like you need to install 'hpricot' before you can use this importer.

If you're using bundler:
  1. Add 'gem "hpricot"' to your Gemfile
  2. Run 'bundle install'
```

Fair enough, they mentioned this in the [docs](http://import.jekyllrb.com/docs/installation/), so I ran:

```
C:\>gem install hpricot
```

I was then able to import the old blog posts successfully:

```
Downloading images for foo - bar
  http://foo.files.wordpress.com/2010/03/bar.png
    OK!
Imported 4 posts
Imported 2 attachments
```

Installing Jekyll afterwards was straightforward:

```
C:\>gem install jekyll
Successfully installed jekyll-2.5.3
Parsing documentation for jekyll-2.5.3
Done installing documentation for jekyll after 1 seconds
1 gem installed
```