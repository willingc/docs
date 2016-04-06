# Writing Your Own Notebook with Remote Compute

Interactive data exploration is essential for data science, quantitative finance,
physics, and many other high tech markets.  An increasingly popular way to
**interact** with data is is to use a notebook that mixes narrative content with
programming code. There are many notebook implementations available today.
Before you decide to write your own notebook-like application, you should
investigate the existing implementations.  There's a good chance that an
implementation already exists which meets your needs or that modifying an existing
implementation will also suffice.  For a list of existing notebook implementations,
see [Appendix A](#appendixA).  

This tutorial describes how to write an Electron based interactive notebook
application.  It's closely modeled after the ideas used in the nteract notebook.
nteract technologies will be used to simplify the implementation.

### Requirements

Before continuing, the following software is required:

- [Node 4.x or 5.x](https://nodejs.org/en/download/)
- npm 3.x (should be included in the node installer)
- [git 2.x](https://git-scm.com/)

To check if you have these installed run

```bash
node -v
npm -v
git --version
```

This tutorial was developed both on Linux and OSX computers.  Optionally, you can
install the [atom editor](https://atom.io/) which is the text editor used in
this tutorial.

### Prepare project

First, open up your terminal and create and navigate into a directory for the
your new project's source code.  Here I create one in my home directory.

```bash
cd ~/
mkdir mynotebook
cd mynotebook
```

Now initialize the directory as a git project.

```bash
git init
```

Next initialize the directory as an npm package.  You can do this using the
wizard included with npm.

```bash
npm init
```

You should see the following

![npm init](1-npminit.png)

If you want, you can enter information about your project, like its name and
description as the wizard prompts for that info.  Otherwise you can accept all
of the defaults by hitting return multiple times until the wizard exits.  You
can change these settings later by editing the `package.json` file that the
wizard will create for you.

### Get electron working

The next step is to setup the project as an
[electron app](http://electron.atom.io/).  Electron is a stripped down version
of Chrome that allows you to build desktop applications using web technologies.
First electron must be installed

```bash
npm i electron-prebuilt@^0.36.2 --save
```

"Wait, this command looks cryptic!  What's going on here?"
`npm i` is shorthand for `npm install`.  `npm i` will be used throughout the
rest of the tutorial to save characters.  `electron-prebuilt` is the name of the
[npm package](https://www.npmjs.com/package/electron-prebuilt) that contains the
electron interpreter.  The `@^0.36.2` tells npm to install the major revision
that this tutorial uses.  Lastly `--save` tells npm to install the package
as a runtime dependency so that it gets installed automatically along with your
package.

The electron-prebuilt package will be installed to a specially named folder in
the project's root directory called `node_modules`.  This folder should not be
checked into git, so it should be added to the `.gitignore` file.

```bash
echo "node_modules/" >> .gitignore
```

Because modern web technology is awesome, we'll opt in to using Javascript ES6
instead of ES5.  To do so, install electron-compile

```bash
npm i electron-compile@^2.0.4 --save
```

And then install the required electron-compilers package as a dev dependency,
which means it will only be installed with development installs

```bash
npm i electron-compilers@^2.0.4 --save-dev
```

Electron compile will automatically
[transpile](https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=define%3A%20transpile)
the ES6 Javascript we use into
ES5 Javascript.

Electron-compile only gets us a part of the way there.  It's the mechanism
that triggers the transpilation, but it's not actually smart enought to do the
translation itself.  Instead it relies on other packages, like Babel, to do that.
We will use [Babel](https://babeljs.io/) to perform the translation.  Babel is
composed of many smaller packages.  Here we will install a subset of those packages:

```bash
npm i babel-core@^6.7.4 babel-plugin-transform-class-properties@^6.6.0 babel-plugin-transform-object-rest-spread@^6.6.5 babel-preset-es2015@^6.6.0 babel-preset-react@^6.5.0 --save-dev
```

Lastly we need to configure babel using a `.babelrc` file.

```bash
touch .babelrc
```

Edit the babelrc (`atom .babelrc`) so it looks like this

```json
{
  "presets": [
    "es2015",
    "react"
  ],
  "plugins": [
    "transform-class-properties",
    "transform-object-rest-spread"
  ]
}
```

This babelrc file tells babel to translate
[es2015](https://babeljs.io/docs/plugins/preset-es2015/)
and [react](https://babeljs.io/docs/plugins/preset-react/)
code into es5 code.  es2015 is ES6, ratified into Javascript in 2015.  To learn
more about it, see [this page](https://babeljs.io/docs/learn-es2015/).  The
object rest spread plugin is a stage 2 (draft) spec of Javascript, read more
about it [here](http://babeljs.io/docs/plugins/transform-object-rest-spread/).
The class properties plugin is a stage 1 (proposal) spec of Javascript, read
more about it [here](http://babeljs.io/docs/plugins/transform-class-properties/).

A typical electron application has two or more processes at any given time.
One of these processes is the "main" process, which is responsible for spawning
windows.  The other processes are "renderer" processes, which are responsible
for rendering content inside each window.  There is one renderer process per
window.

The thread structure will be reflected in the folder structure of the project.
In npm projects it is common practice to put all uncompiled code inside a source
directory called `src`.  We will create two subdirectories, one for each thread
type, and `index.js` files for each of the directories.  `index.js` is a common
name used for an entry point file within npm projects.  Additionally we will
create an index.html file which will be the entry point file for the spawned
renderer windows.  This html file will load the renderer index.js file.

```bash
touch index.js
mkdir src/
cd src/
mkdir main/
mkdir renderer/
touch main/index.js
touch renderer/index.js
touch renderer/index.html
cd ..
```

Now, open the project's root directory for editing in your favorite text editor.

```bash
atom .
```

The next step is to setup the execution script.  Before we do so, it's useful to
review how npm package scripts work.  Open the `./package.json` file.  It should
look something like this:

```json
{
  "name": "mynotebook",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "electron-compile": "^2.1.4",
    "electron-prebuilt": "^0.36.12"
  }
}
```

The `scripts` section of the package.json file describes what scripts your
package provides.  Scripts can be run from the terminal via `npm run`.  For
example, to run the `test` script specified in the package.json above, one
would execute `npm run test` in the terminal.

The script names may also have syntactical meaning.  Scripts with a name
prefixed with a `pre` get executed before the named script is invoked.  
Likewise, names prefixed with `post` get executed after.  To illustrate this,
imagine the following scripts existing in a package.json

```json
{
  "scripts": {
    "task": "echo \"a\"",
    "pretask": "echo \"b\"",
    "posttask": "echo \"c\"",
    "other": "echo \"d\"",
    "preother": "echo \"e\"",
    "postother": "echo \"f\"",
  },
}
```

Executing `npm run task` would result in the following output

```bash
b
a
c
```

Scripts can also call other scripts by calling `npm run`.


```json
{
  "scripts": {
    "task": "echo \"a\"",
    "pretask": "echo \"b\" && npm run other",
    "posttask": "echo \"c\"",
    "other": "echo \"d\"",
    "preother": "echo \"e\"",
    "postother": "echo \"f\"",
  },
}
```

Here, executing `npm run task` would result in the following output

```bash
b
e
d
f
a
c
```

Lastly, it's good to know that there are special script names.  "publish",
"install", and "test" are all names that are known and called as a result of
other npm actions.  You can use "pre" and "post" prefixes to automate actions
alongside these scripts.  For example, it's common to create a "prepublish"
script which compiles the code automatically prior to an attempt to publish to
npm.

In our package.json file we will define a "start" script, which will launch
the project using the electron interpreter.


```json
{
  "name": "mynotebook",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "electron index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "electron-compile": "^2.1.4",
    "electron-prebuilt": "^0.36.12"
  }
}
```

Next, edit `index.js` so that it uses electron-compile to transpile and load
the `src/main/index.js` file.

```js
// NOTE: This must remain ES5 code.
var path = require('path');
var appRoot = path.join(__dirname);
require('electron-compile').init(appRoot, './src/main');
```

Edit `src/main/index.js`, add a listener for the `ready` event of the
application.  This is what gets called when the application has loaded.  It is
here that you can spawn the window of your application.

```js
import app from 'app';

app.on('ready', () => {
  // TODO: Spawn window
});
```

To spawn a window, import `BrowserWindow` from the `browser-window` package.
Construct an instance of the BrowserWindow object, passing an options object
which describes the properties of the Window.  Here spawn a window with the
title `My Window Title`

```js
import app from 'app';
import BrowserWindow from 'browser-window';

app.on('ready', () => {
  let win = new BrowserWindow({ title: 'My Window Title' });
});
```

Now run `npm run start` from the root of your project.  You should see a blank
window appear:

![electron quickstart](2-electron.png)

Now edit the `src/renderer/index.html` so that it loads the
`src/renderer/index.js` file relatively.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
  </head>
  <body>
    <script>
      require('./index');
    </script>
  </body>
</html>
```

Add a log statement to `src/renderer/index.js` to we know if it is loaded.

```js
console.info('renderer index loaded');
```

Finally, back in `src/main/index.js`, load the `src/renderer/index.html` file.

```js
import app from 'app';
import BrowserWindow from 'browser-window';
import path from 'path';

app.on('ready', () => {
  let win = new BrowserWindow({ title: 'My Window Title' });

  const index = path.join(__dirname, '..', 'renderer', 'index.html');
  win.loadURL(`file://${index}`);
});
```

Launch the application using `npm run start`.  Once the application starts, open
the developer console by clicking `View->Toggle Developer Tools` in the
application menu.  A Chrome developer console will appear in your application.
In the log you should see the `renderer index loaded` info statement.

![renderer load success](3-loaded.png)

Stage and commit your changes

```bash
git add -A
git commit -m "Skeleton"
```

### Understand the notebook model

**TODO: Rewrite this bit**
The first step in designing a notebook application is to design a notebook
model.  To simplify the tutorial, our notebook model will be a linear array of
"cells", where each cell can either be a code cell or text cell.  This is the
same model that [nteract](https://github.com/nteract/nteract) uses, so you will
be able reuse nteract's document manipulation library,
[commutable](https://github.com/nteract/commutable).  Don't worry about the
details yet, you'll learn about the libraries later in the tutorial.

### Overview of nteract technologies
### Create a state store (redux)
### Load a notebook (commutable)
### Design notebook components (react, react-transformime)
### Launch a kernel (spawnteract)
### Execute code and display results (enchannel)
### Save the notebook (commutable)
### Conclusion

### Appendix A <a name="appendixA"></a>
- [nteract](https://github.com/nteract/nteract)
- [Jupyter Notebook](https://github.com/jupyter/notebook)
- [Jupyter Lab](https://github.com/jupyter/notebook)
- [SageNB](http://www.sagenb.org/)
- [Beaker Notebook](http://beakernotebook.com/)
- [Apache Zeppelin](https://zeppelin.incubator.apache.org/)
