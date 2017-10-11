---
title: How npm 5+ works with package-lock
date: 2017-10-11 15:41:16
tags: npm package.json package-lock.json package-lock dependency management npm5 react
---

Having had hard times understanding how exactly npm processes package-lock.json and at what stages it is being updated, I put a little guide together to sort of reverse engineer in what way the locked dependencies are updated. This is not going to be an exhausting research about every aspects of npm dealing with these locked dependecies, but rather a very quick step-by-step guide that lines up my findings.

## Preparations
To get the ball rolling, I prepared a rather simple React boilerplate project to experiment with. It incorporates the very basics needed to generate a "Hello React!" application. This will perfectly make it for our little demonstration.

Let's swiftly take a glimpse through the fundamental project files.

```app.js```
![carbon 2](https://user-images.githubusercontent.com/6104164/31431351-bb5ce0c4-ae73-11e7-8ce9-880decff9a1b.png)

```webpack.config.js```
![carbon 3](https://user-images.githubusercontent.com/6104164/31431357-c08d9ad4-ae73-11e7-91b8-22745e01e579.png)

```package.json```
![carbon 4](https://user-images.githubusercontent.com/6104164/31431358-c0a5b970-ae73-11e7-9315-e78caa154371.png)

And where our most attention will stick around, the infamous ```package-lock.json```. Since it takes in a massive lump of rigorious configurations, I merely copied our ```react``` dependency, which is currently set to ```v16.0.0```.  

```package-lock.json```
![carbon 5](https://user-images.githubusercontent.com/6104164/31431359-c0bd0b02-ae73-11e7-9fab-37e1ae7dc8f1.png)

## Break the build 
We're now mischeviously going to introduce a breaking change with substituting our app to use ```React.createClass``` instead of ```React.Component```. It is regarded as deprecated, besides React 16 discontinue supporting it. Nicely done. If I now build the app and execute it in a browser a nice redish error appears. 

<img width="474" alt="captura de pantalla 2017-10-11 a las 10 29 23" src="https://user-images.githubusercontent.com/6104164/31431965-840083c2-ae75-11e7-90c7-fcb8e7b65c6a.png">


## Fix the build 
Without making a fuss out of this, I'll just have to downgrade our ```react``` dependency back to ```15.5.0```. If you want to learn more about this version check out this official changelog https://reactjs.org/blog/2017/04/07/react-v15.5.0.html. 


## Attempt #1
Who wouldn't want to be the hero who saves the day for the support team next to no time. Updating one file with replacing a number with another shouldn't be such a pain in the ass. So fully nerved and intoxicated with confidence we know we just need to change the ```react``` dependency back to ```15.5.0``` and lock it. You can already tell them that it would take an hour with all CI processes. But obviously, like many things in the javascript world, things don't come off such smooth. So let's see what happens. 

Just by changing the ```package-lock.json``` and re-running ```npm install``` no change will take place, furthermore ```npm``` will change our locked ```react``` back to ```16.0.0``` at the very first opportunity when it runs. So it seems like we've got to invalidate the cache perhaps. 

We all know what's the most relaible way to do so. 

```
rm -rf ./node_modules
npm install 
```

This again regenerates our ```package-lock.json``` with the invalid ```16.0.0``` version.

## Attempt #2
There is nothing really shocking in why my first attempt failed. ```npm``` regenerates the ```package-lock.json``` according to our ```package.json```. Having another look at the locked dependency configuration for ```react``` in ```package-lock.json``` we could easily spot that there is much more just than a simple version number. 

![carbon 5](https://user-images.githubusercontent.com/6104164/31431359-c0bd0b02-ae73-11e7-9fab-37e1ae7dc8f1.png)

As you see, there are clearly critical attributes that ```npm``` takes into account when executing the install process, so I'm pretty intimidated at this point that we could actually change anything here manually. But, for the sake of completeness I'll radiply do it again. 

![carbon 7](https://user-images.githubusercontent.com/6104164/31434626-255864e0-ae7d-11e7-9ebd-d7e5dc1406e8.png)

Alright, I've changed everything I could, but still pretty concerned that the ```integrity``` attribute will just simply screw me over, but hey, be positive.

#### Introducing --no-package-lock

Another significant change in my approach will be the usage of ```--no-package-lock``` flag. My assumption is that executing ```npm``` with the ```--no-package-lock``` flag will avoid regenerating our modified ```package-lock.json```. 

As the next step, I'm revalidating the ```npm``` cache and re-installing the dependencies in hopes for getting ```react v15.5.0``` installed as desired.

```
rm -rf ./node_modules
npm install --no-package-lock

```

Let's see our ```package-lock.json```.

![carbon 7](https://user-images.githubusercontent.com/6104164/31434626-255864e0-ae7d-11e7-9ebd-d7e5dc1406e8.png)

Alright, ```npm``` has not regenerated the ```package-lock.json```, but let's see if the appropriate version of ```react``` has been installed. To check this, I'm going to open the corresponding ```package.json``` file in ```node_modules/react/``` folder. 

```
...
 "version": "16.0.0"
...
 ```

So to recap, ```--no-package-lock``` flag makes ```npm``` completely disregard the shrinkwrap functionality and therefore all the dependencies will be installed as ```package.json``` defines. 

## Attempt #3

So it seems like we've got to do this the cumbersome way. My intent is to directly change our ```package.json``` and sort of make ```npm``` regenerate ```package-lock.json``` with ```react 15.5.0```. 

![carbon 11](https://user-images.githubusercontent.com/6104164/31436047-f2469432-ae81-11e7-8bd9-81a10052f310.png)

Revalidate and re-install the packages accordingly. 

```
rm -rf ./node_modules
npm install
```

Check our relevant bit of ```package-lock.json```.
![carbon 10](https://user-images.githubusercontent.com/6104164/31436052-f3ee505e-ae81-11e7-96da-5b4786987725.png)

Cool! It's now obviously generated with the right version. I'm going to verify whether we've got the right version and then execute the app in the browser to double check whether the React error has disappeared. 

Version of React package in ```./node_modules/react/```.
```
...
 "version": "15.5.0"
...
 ```
 
Result of execution. 

<img width="1155" alt="captura de pantalla 2017-10-11 a las 12 34 13" src="https://user-images.githubusercontent.com/6104164/31435632-8b895636-ae80-11e7-9c95-32ac9ba0a022.png">

The app renders our message and we can now see only a Warning about ```React.createClass```. Everything as expected.


## Lock the version
To go on, we're going to assume that a newer version of the app, nonetheless requires ```react 16.0.0```. Let's see what happens if we update our ```package.json```. To addition to our previous steps, I'm increasing the major version of my package to tell ```npm``` to persist this into the lockfile.

```
...
  "version": "2.0.0",
  
  "dependencies": {
  ...
    "react": "16.0.0",
  ...
  }
...
```

Revalidate ```npm``` chache and re-install dependencies with ```--no-package-lock``` flag to forbid ```hpm``` regenerating the lock file.

```
rm -rf ./node_modules
npm install --no-package-lock
```

Though our ```package-lock.json``` has been intact without setting our react dependency to ```16.0.0``` 

![carbon 7](https://user-images.githubusercontent.com/6104164/31434626-255864e0-ae7d-11e7-9ebd-d7e5dc1406e8.png)

the app crashes in the browser since the react dependency has been upgraded to ```16.0.0```. 

<img width="474" alt="captura de pantalla 2017-10-11 a las 10 29 23" src="https://user-images.githubusercontent.com/6104164/31431965-840083c2-ae75-11e7-90c7-fcb8e7b65c6a.png">

#### So how we can do it actually?

Apparently, we cannot lock dependencies with explicit version definitions and bind them to the various versions of our package. What I mean by this is you just simply cannot tell ```npm``` to use ```react 15.5.0``` for our package version ```1.0.0``` and subsequently ```react 16.0.0.``` for newer versions. 

#### What's the whole purpose of package-lock.json then? 

This question stroke me right after learning the disappointing true about how simple ```npm``` is. Yes, yes I reckon this is a bit harse indeed to claim this and locking the dependencies ensures that at any point in time the package will be installed with the right dependencies, but apparently the only actual benefit of ```package-lock.json``` shows up when using the ```caret (^) or tidle (~)``` symbols for transitive dependencies. 

In our case. 

![carbon 12](https://user-images.githubusercontent.com/6104164/31437279-02ab5940-ae85-11e7-84b6-06a2eed120a7.png)

The usage of the caret symbol along with our locked dependency in ```package-json.lock``` disallows ```npm``` to fetch any transitive dependencies and therefore all dependency will be downloaded as defined in the lockfile. 

## Dispel the confusion around --no-package-lock
```--no-package-lock``` flag will make ```npm``` to fully operate without the shrinkwrap functionality. As a result, not only ```package-lock.json``` will **NOT** be regenerated, but it also will **NOT** be processed whatsoever and ```npm``` will subsequently install the newest available transitive versions of the dependencies. 

## Using npm update
Be careful as ```npm update``` will update your ```package-lock.json``` with the available transitive dependencies. In my case it updated both my `package.json` along with `package-lock.json`.

```
$ npm update 
+ react@15.6.2
+ react-dom@15.6.2
added 2 packages and updated 2 packages in 3.054s
```

![carbon 13](https://user-images.githubusercontent.com/6104164/31438869-542e2270-ae8a-11e7-90e3-fb7227cdcb0c.png)

![carbon 14](https://user-images.githubusercontent.com/6104164/31438871-544d4632-ae8a-11e7-80fb-8733106f398f.png)


## How to lock dependencies and bind them to various versions

At the time of writing ```npm``` is incapable of doing this and further version control tool is to be employed. 
