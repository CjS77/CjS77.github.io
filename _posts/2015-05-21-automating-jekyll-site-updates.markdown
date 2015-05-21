---
layout: post
title:  "Automatic updates for jekyll-based sites"
date:   2015-05-21
header-img: img/auto-update.png
author: Cayle Sharrock
categories: jekyll git 
---

# The problem

I'm using [jekyll] to write my [company's website](http://nimbustech.biz) (in addition to this blog). I'm using [git] to manage the content 
revisions, naturally, but since it's a private repository, I'm not using github pages to keep the website and the 
site content in the repo synchronised.

The website itself is hosted on rented server space that I have ssh access to, but is not the same server that hosts
the git repository, or the development machines (obviously).

I *could* follow one of the ideas presented in the [jekyll website](http://jekyllrb.com/docs/deployment-methods/), but 
 I found most of them to be, well *meh.*, especially since I don't want to recreate the entire Git repository on the web 
 server -- I just want the static Jekyll output put there, and kept in sync. 
 
So I rolled my own.
 
# My solution

The solution is actually just two simple scripts.

## Development side script

On my development machine, I have a script that does the following:

1. runs `grunt prod` to produce the production version of the site [^1]
2. runs `jekyll build` to produce the website code (in `_site`)
3. Compresses and secure copies the tarballed site to an 'incoming' folder on the web server.
    * There are some steps to make this automatic, like setting up SSH keypairs between the dev machine and the webserver.
        The steps for doing this can be [found elsewhere](https://www.debian.org/devel/passwordlessssh).

Pretty simple, yeah? The script is essentially this:

    #!/bin/bash
    grunt prod
    jekyll build
    tar -czf my_website.tar.gz _site/*
    scp my_website.tar.gz mywebserver.com:~/incoming/

[^1]: I maintain some grunt scripts that allows me to make both a development and production version of the site. 
      The latter combines, minifies and optimises css and js scripts and carries out some other tweaks that makes debugging hard.    

## Server side script

Ok, now on the server, I want it to automatically respond when a new version of the site is uploaded, and update 
the website source with the new version.

For good measure, it will email me when it's done.

I'm not covering how to set up the web server here. I'll assume that the live content is being served out of /var/www/my_site
 and that the 'incoming' folder is in a user home folder. This user will also need to be in the sudoers list to carry out the
 admin tasks we'll set up. For security reasons, the live code is owned by the webserver user, www-data.
  
In summary
  
| Location | Folder | Owner | Permissions
|----------------------------------|
| Live site | /var/www-data | www-data | 664
| incoming updates folder | /home/cayle/incoming | cayle | 660
  
 
There are a couple of things that need to happen here:

1. The system must watch for changes in the `incoming` folder
2. A script must them launch that does the following:
    * extract the tarball over the old code
    * changes the ownership of the new code to www-data
    * Mails me when it's done
    
The first step is achieved by using [incron](http://www.cyberciti.biz/faq/linux-inotify-examples-to-replicate-directories/).

1. Install incron (`sudo apt-get install incron`)
2. Add a line along the lines of this to `/etc/incron.d/my_website`:

     `/home/cayle/incoming IN_CLOSE_WRITE /home/cayle/update_site.sh $#`

<div class="note warning" markdown='1'>
Don't get incron to watch for modifications, i.e. IN_MODIFY, unless you *want* fire the update_site script hundreds of times while
      a new tarball is being uploaded. I managed to DOS attack my own server learning this lesson `:face_palm:`
</div>       

The `/home/cayle/update_site.sh` is the last piece of the puzzle and is pretty straightforward:
 
    #!/bin/bash
    echo `date` >> update_site.log
    echo "Detected change in $1" >> update_site.log
    # This file gets called automatically by incron when files in this folder are modified
    if [ x$1 == xmy_website.tar.gz ]; then
      echo "Extracting archive...">> update_site.log
      tar -xzvf /home/cayle/incoming/my_website.tar.gz -C /var/www/my_site >> update_site.log
      echo "Changing ownership..." >> update_site.log
    
      chown -R www-data.www-data /var/www/my_site/* >> update_site.log
    
      echo "Done." >> update_site.log
    
      echo "My website has been updated" | mail -r support@nmywebserver.com -s "Website updated" myemail@address.here >> update_site.log
    
    fi
{: .highlight}
 

    
  
 [jekyll]: http://jekyllrb.com/
 [git]: http://github.com 