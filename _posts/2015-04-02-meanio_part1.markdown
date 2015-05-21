---
layout: post
title:  "A Kue tutorial using MEAN.io - Part 1"
date:   2015-04-02
header-img: img/meanio-ninja.jpg
author: Cayle Sharrock
categories: node.js job-scheduling kue angular
---

# Setting up the project

I'm going to try out the [mean.io](http://learn.mean.io/) tool for scaffolding and managing this demo.

        sudo npm -g install mean-cli
        mean init kueDemo
        cd kueDemo 
        npm install
        bower install
  
Some interesting things happen here:

  * `mean init kueDemo` clones the github project for mean.io into this folder
  * `npm install` runs bower automatically, although the docs indicate that this doesn't always happen, so
    run bower anyway to be sure.
    
Since I now had 2 git repositories nested, I removed the link to mean.io by running

        git remote rm upstream
        rm -fr .git
        cd ..
        git add kueDemo

I'm sure there's a better way of doing this (that e.g. keeps the link to mean.io as a remote), but I couldn't
figure it out. I think if the mean.io script did a fetch rather than a clone, this might have been easier.

Maybe I can re-add the mean.io as a remote in future and fetch from it to keep the library up to date, I'm not sure.

## Installing dev dependencies

One of the dependencies causes an out-of-memory failure on npm. If you go `npm install --dev`, you run out of memory,
at least on a 8GB system.

    
# Checking the basic install

  1. Start Mongo: `mkdir /tmp/kueDemo && mongod --dbpath /tmp/kueDemo --smallfiles`.
  
     The `--smallfiles` option doesn't use as much disk space, but is less efficient.
  2. Run `gulp` to run the default task    
  3. Open a browser at <http://localhost:3000> and you should see the basic scaffolding
  
![Default mean.io app](/img/kue-homepage-scaffold.jpg)

# Looking around

The basic folder structure is pretty deep, so here's a quick summary:

| Folder                       | Filename      | Description                                  | Comment                                                                                                |
|------------------------------|---------------|----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| root                         | gulpfile      | The task runner definition file              | Hardly need to modify, if ever; customisation will happen in the gulp/*.js files                       |
|                              | bower.json    |                                              |                                                                                                        |
|                              | package.json  |                                              |                                                                                                        |
|                              | karma.conf.js | Karma test config file                       | Looks pretty comprehensive. Can probably leave alone; maybe add more browsers                          |
|                              | mean.js       | *No idea at this point*                      |                                                                                                        |
|                              | server.js     | Fires the app up                             | Usually don't need to touch                                                                            |
| config                       | *.js          | one config file for each type of environment | Pretty intuitive as to what you need to edit here                                                      |
| config/middlewares           |               | *No idea at this point*                      |                                                                                                        |
| gulp                         | *.js          | Gulp task definitions; per environment       | Some customisation will probably be needed; e.g. adding less, moving to jade                           |
| packages                     |               | This is where the magic happens              |                                                                                                        |
| packages/custom              |               | User-generated modules go here               | use `mean package <PKG_NAME>` to scaffold packages. For god's sake, don't try create them from scratch |
| packages/custom/MY_PACKAGGE  | app.js        | Auto-generated.                              | Usually don't need to touch                                                                            |

# Mocha Test bug

Two days of frustration to figure this out. The mocha tests weren't running in the mean.io app

The problem lies with the relative paths used to both

 1. glob the model paths
 2. glob the unit tests

**The fix:**

  * Changing the `../packages/...` paths to `./packages/...` in `gulp/test.js` sorts out 2 and partly 1.
  * But now the packages won't load in `require`, so You also need to add a few lines in function `walk` in 
   `node_modules/meanio/lib/core_modules/module/util` as follows:
   
         function walk(wpath, type, excludeDir, callback) {
           // regex - any chars, then dash type, 's' is optional, with .js or .coffee extension, case-insensitive
           // e.g. articles-MODEL.js or mypackage-routes.coffee
           var rgx = new RegExp('(.*)-' + type + '(s?).(js|coffee)$', 'i');
           if (!fs.existsSync(wpath)) return;
           fs.readdirSync(wpath).forEach(function(file) {
             var newPath = path.join(wpath, file);
             var stat = fs.statSync(newPath);
             if (stat.isFile() && (rgx.test(file) || (baseRgx.test(file)) && ~newPath.indexOf(type))) {
               var realPath = fs.realpathSync(newPath);                //New line here
               callback(realPath);                                     //Modified line here
             } else if (stat.isDirectory() && file !== excludeDir && ~newPath.indexOf(type)) {
               walk(newPath, type, excludeDir, callback);
             }
           });
         }
         
Now the tests run:

      27 passing (7s)

        

In [Part 2]({% post_url 2015-04-03-meanio_part2 %}), we'll start with some actual coding.
 
 