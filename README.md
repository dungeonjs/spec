# dungeon is a package manager for javascript applications


I don’t like npm. npm is crap. it is a shitload of crap. this is a document specification which describes how dungeon works. (or should work :/)

---

##How it works!!

installation (of packages):

`dungeon install <url>`

specification of the url parameter is described below. the global dungeon config can specify mapping of urls to packages. so if you map `foo.bar/baz` to the FQN “baz”, `dungeon install baz` will work the same way as `dungeon install foo.bar/baz`.

---

##URL specification:

the url can be a web protocol based url, file system based one, or git. it should be resolved to:

 - either a tar.gz
 - a folder which when tar.gz’d becomes a tar.gz (genius!)
 - a git repository which when tar.gz’d without the `.git` folder inside it becomes a tar.gz (mega genius!)

the container folder must contain a package.json file with metadata.

the meta data specification is described below (will be. maybe).

the role of a tar.gz is like that of a `.deb` or `.rpm` or `.msi` installer. when installed, that tar.gz is not required for usage of the package. it is actually extracted and copied to the host environment.

the tar.gz can of course contain dependency packages if required. or not. (do you have any idea how hard it is to deal with these packages?)

due to this, there is no need for dungeon to have any registry. like npm or jspm or any other package manager does. you can obtain the tar.gz from any where, and like you do `dpkg -i package.deb`, you can do `dungeon install package.tar.gz`

---

##metadata:

dungeon is strictly a package manager only. no build or testing or whatever. however, it provides hooks for building and testing. metadata provides those hook information.

metadata file also reduces clutter in your project root. for instance, if you don’t want to have a ton of build information files, and description docs in the project root but in a sub folder called “meta”, or anywhere else for that matter, you can just put the paths in the metadata file.

the metadata file also serves as a signature for the package, thus you can have multiple versions of the package based on the metadata file. nothing new to see here, traditional brilliance.

signature of the package being obtained by the meta-data file is not easy.

the meta-data file contains a lot of easy to change things. like a space in the package name can cause the signature to be different. this is, good and bad. like girls. we don’t want dungeon to behave like a girl.

for this, normalization of the meta-data file is required. we can obviously trim the file contents from white spaces etc., but since it is all json, we also need to look at the order of rules.

consider the following two json objects:

```json

{
	“foo”: “bar”,
	“zirak”: “sucks”
}

```

```json

{
	“zirak: “sucks”,
	“foo”: “bar”
}

```

These are the same objects, but hashing the file will result in a different signature for each of them.

one of the obvious solutions is to parse the file, and re stringify it in a particular order. since the stringification of a json object from a standards compliant ECMAScript implementation is largely the same for each locale, but different for different locales, this is not a very effective solution in the long run. then we have to deal with all the different encodings and crap where json fails.

we can also use yaml, which can greatly solve a few of these problems. but it will break from the traditional trend of js package manager. not to mention, json global object is present in ECMAScript natively, but not yaml. // TODO: Discussion

---

##hooks:

hooking the package manager with build tools has so far been a terrible idea for most package managers. it doesn’t follow the unix principle of making small and focused tools. a manager should manage packages, not do shit with the code.

in the JS land, we do have some very standard shit to do though. transpilation, is probably the most famous one of all.

the problem with this is that JS alone doesn’t do anything. it is most widely used as only an embedded language. so we CSS, yaml/haml/jade templates and what not.

due to this, dungeon will try to stay away from the regular kitchen sink of frontend. we might create abstractions over regular build tools with APIs to integrate with dungeon, but right now, it is only the package manager. but those “APIs” are something we need to reserve space for, in the future (yes, the obvious inspiration is ECMAScript’s future reserved keywords).

thus, certain rules like “build”, “test” etc. in the root level will not be allowed in the meta-data file for now.

presence of such rules will throw a RunTime error during parsing of the meta-data.

A complete list of future reserved rules will be written somewhere down the road. I think.

---

##Management of packages on the file system.


the npm strategy towards this is as follows:

create a folder inside project root if does not already exists called “node_modules” (which is a node thing, not an npm thing, btw)

parse the package.json file, and extract all dependencies.

    for each d in dependencies:
    	install d in node_modules
    	install all dependencies of d inside node_modules of d

just a simple change in the above algorithm can make this a lot more efficient for some major packages (tests on mocha, and phantomjs suggest a performance gain by about 30%. I suspect it is much more on babel though build failed :/)

    for each d in dependencies:
    	install d in node_modules
    	install all dependencies of d along side d

this is similar to what jspm does.

we need to change this algorithm further to allow global installations

    let ROOT = <path>/usr/share/node_modules/ (overridable by user, good point @amitavaghosh)
    let TEMP = <path>/tmp
    let current_signature = signature obtained by the meta-data file of the presently being installed package
    
    for each d in dependencies:
    	let s be the signature for d obtained from meta-data file
    	if s exists in global store INSTALLED_PACKAGES
    		let dependants be INSTALLED_PACKAGES[s]
    		dependants.push(current_signature)
    		return
    
    	install d in TEMP
    	set INSTALLED_PACKAGES[s] to the new list with one element current_signature
    	copy d from TEMP to ROOT iff setting of the above property went fine, else rollback

this algorithm maintains the global database of which package has how many dependants. instead of recording dependencies,  we are recording dependants! the reason for this will be described below:

dungeon allows garbage collection of packages. so say you got an app from github which looks cool, without any kind of FUD, you can do `dungeon install app`. not to worry.

then run the app. it would have polluted your global scope for now. but you can easily do `dungeon uninstall app` which will take the following steps:

    let current_signature be the signature of the current app
    clear INSTALLED_PACKAGES[current_signature]
    let remnants = INSTALLED_PACKAGES[current_signature].filter(hasDependants)
    inform user that packages could be cleared if remnants.length > 0, and how much space it will clear

after that, user can do `dungeon prune`, and all the remnants will be uninstalled as well, leaving no trace of the package installed, clearing all “pollution”.

##making the installed packages available to userland code

the node_modules structure is largely used by backend apps only running on node. but js land now uses node as a platform for building apps for the client as well.

this is, TBH, a backward solution. `node_modules` by node is to reuse code for backend apps. and until ES6 modules are not implemented by browsers, it doesn’t make much sense for front-end apps to follow that structure. this is why dungeon allows you to store the symlinked packages anywhere, not just `node_modules`.

the problem this solves is a big one, as described below (the problem, that is):

say there is some library called “lib” for frontend, which is distributed as a package and can be bundled into a single large minified JS file called “lib.min.js”. and user wants to include this in the userland js code.

putting this in node_modules makes little sense.

 - it is not a node module
 - it is not an executable module or package either
 - it is not “installable” in real. it is meant to be included
 - because zirak sucks

dungeon allows you to install it like any other node module, marking in the global store that it is not really a node module but user acting retarded, and we are correcting the retard’s mistake.

the user can then specify a folder to place a symlink to that package in your project root. say ours is a PHP app, and the structure is like this:

    src/
    	… backend code which must not be visible to user
    html/
    	… we don’t want a node_modules folder here because this folder has nothing to do with node
    	static/
    		… this is the place where frontend code is kept
    		… we don’t want node_modules here either
    		… instead we call it “vendor” (or whatever your backend language calls it for third party apps)
    		vendor/
    			… dungeon places package sym links here
    			… your build tool can recurse through each package here and bundle it
    			… aids in development since you will always have the development files only
    	index.php
    ...php config files

##different versions of packages

one solution that I thought of this is to simply not have any concept of different versions of packages by numbers. instead, different versions of the packages will be treated as different packages.

in other words, dungeon will not be aware of the fact that `foo@2.5` is the same package as `foo@2.3`. It will store the signature depending on the package version as well, not just the package name. this makes packages as first class citizens.

implementing commands like `dungeon update <package name>` will be easy to implement over this, but the underlying structure of the ROOT directory where packages are globally held will be simplified.

to ease the usage of such a system, we need to implement package aliases. so one can alias “foo@2.5” to “foo” and refer to the package as foo only. note that we already did spec this in the form of mappings from urls to packages, so we can now merge the two concepts together.

for this, we further simplify the definition of a package as simply a url. different url? different package. this allows us to do stuff like following:

```sh

$ dungeon install https://github.com/awalGarg/sdm.git
installed package sdm version 0.1.2 (version extracted from package.json)

// now sdm is available as “awalGarg/sdm@0.1.2” for all the packages on the system

$ dungeon map awalGarg/sdm@0.1.2 sdm
mapped awalGarg/sdm@0.1.2 to sdm

// a few days later, say awalGarg/sdm updates to 1.0.0 which breaks backwards compatibility (semver to the rescue)

$ dungeon update sdm
detected major semver version update. this might break backwards compatibility, are you sure you want to continue?

> yes shut up and take my money

updating package sdm from 0.1.2 to 1.0.0 and remapping to sdm

package sdm updated (major updates)

// say that a few days later, another major update happens and we jump to v2.0.0
// this time, we want to install it along side the present version

$ dungeon install https://github.com/awalGarg/sdm.git
a package from this url is already installed, checking for updates
major update detected, installing alongside since command was install and not update
installed package awalGarg/sdm@2.0.0 available to all code

$ dungeon map awalGarg/sdm@2.0.0 sdm
mapped name sdm already exists, continuing will unbind old mapping, do you want to continue?

> continue bitch

unbinding mapping awalGarg/sdm@1.0.0 sdm
package awalGarg/sdm@2.0.0 mapped to sdm

```

---

##managing placement of binaries

// this, I leave for erik royall to do later *evil laugh*

---

##gitignore recommendations

this is also one topic of debate when using npm, bower, and all kinds of messy stuff. should we include package data in version control or not? and what about some config files.

the latter is resolved easily: no config files for dungeon, so nothing to add to version control.

the first part is difficult.

remote dependencies could go down, but dungeon doesn’t store packages in the root folder. so whether you like it or not, you can’t include them in the version control directly. well, you can ofcourse, git traverses symlinks as well but it is not recommended.

##what about packaging apps to give to a client?

`dungeon package` to the rescue! the way dungeon package works is very generic. you can use it to write a new package as well…

the following steps are taken place when we do `dungeon pack` in the project root:

```
// this will be interactive:

let store be a freshly created folder which is empty for now, and will later be gzipped
put all _user land_ code in the folder as it is preserving the directory structure. so no copying of node_modules

let shouldPackageAlong = []

for d in dependency from node_modules:
	let s be the signature of d
	let location be the url from where d was originally installed (obtained from the global database)
	if location is a relative path, push [d,s] in the shouldPackageAlong
	if location is a remote url, prepare stdout and stdin for QA from user:
		$> encountered dependency D which is not a local url. do you want to bundle it with the package?
		if user says yes, push [d,s] to shouldPackageAlong
		else continue
	if location is an absolute path, prepare stdout and stdin for QA from user:
		$> encountered dependency D located on local FS at url ${location}. bundle along?
		if user says yes, push [d,s] to shouldPackageAlong with the flag LOCAL
		else continue

for d in shouldPackageAlong:
	let s be the signature of d
	modify the meta-data file of the package presently being packaged as follows:
		set location of s to “resource://node_modules/s”
	place all code of d in store/node_modules/{s}

tar.gz the store folder, and prepare the stdout and stdin again:

$> where do you want to place the packaged app?
// get answer

place the tar.gz file at the given path
return

```

when unpacking/installing a tar.gz, dungeon will traverse urls of the resource protocol from the tar.gz/node_modules folder and install them globally.

---

- Awal Garg aka Rash
- Argentum47
- Erik Royall

License: WTFPL
