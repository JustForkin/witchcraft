
# dot.js: GreaseMonkey for developers

> "I almost wish you could just stick JavaScript in ~/.js. Do you know what I'm saying?"

dot.js is a Google Chrome extension that executes JavaScript and CSS files in `~/.js` based on the page domain being accessed.

If you navigate to `https://www.google.com`, dot.js will execute `~/.js/google.com.js` and `~/.js/google.com.css`.

This makes it super easy to spruce up your favorite pages using JavaScript and your own stylesheets.

On subdomains such as http://gist.github.com, dot.js will try to load `~/.js/gist.github.com.js` as well as `~/.js/github.com.js` and `~/.js/com.js`. The same goes with CSS: `~/.js/gist.github.com.css`, `~/.js/github.com.css` and `~/.js/com.css` will be tried.

GreaseMonkey user scripts are great, but you need to publish them somewhere and re-publish after making modifications. With dot.js, just add or edit files in `~/.js`. Script changes will immediately be seen by the extension; no need to reload anything.

## What if I want to inject jQuery (or any other library?)

Use `@include` directives. Inside your main script (say, `google.com.js`), write:

    // @include jquery.js

And then just add `jquery.js` to your `~/.js` folder. You can any number of scripts you want. Included scripts will also have their `@include` directives parsed in a recursive fashion. Dependency cycles (e.g., `foo` includes `bar`, which includes `foo` won't break the parser and the resulting script will be `<contents of bar>\n<contents of foo>`.

*Note: this is my rendition of [defunkt](https://github.com/defunkt)'s original tool, [dotjs](https://github.com/defunkt/dotjs). Although I never got to actually use his implementation, I really wanted something like that. My approach works just the same as his, but it's way easier to install and should work for any platform that is able to run Node.js.*

# Example

    > cat ~/.js/google.com.js

    document.querySelectorAll('img')
        .forEach(img => img.style.transform = 'scaleX(-1)');

Or you could also just:

    > cat ~/.js/google.com.css

    img { transform: scaleX(-1); }

![defaced avatars](elgoog.png)

# How to install

This extension is composed of two parts: the Chrome extension itself and a local HTTP server running on Node.js. Follow these two steps below to install it.

## Step 1: start the server

First make sure you have [Node.js](https://nodejs.org) installed.

Go to the root of this repository and simply run:

    node dot.js

*Note: you may want to add it to your shell's login script, otherwise you'll need to restart it every time you reboot your system.*

## Step 2: install Chrome extension

The extension is in the folder `chrome-extension/`. Please refer to Google on how to install development Chrome extensions on the most recent Chrome release. I am not writing the steps here since this is likely to change for newer Chrome versions.

After both the server is running and the extension is installed, you're good to go. Write your kick-ass scripts and just put them under `~/.js`.

# How it works

Chrome extensions can't access the local file system, so dot.js runs a tiny web server on port 3131 that serves files out of `~/.js`.

The dot.js Chrome extension then makes ajax requests to http://localhost:3131/www.google.com any time you hit a page on `www.google.com`, for example, and executes the returned JavaScript. It will also append any relevant CSS files to the page's head tag.

Our tiny server, upon receiving a request to `www.google.com`, looks for these scripts, in this exact order:

* `~/.js/com.css`
* `~/.js/google.com.css`
* `~/.js/www.google.com.css`
* `~/.js/com.js`
* `~/.js/google.com.js`
* `~/.js/www.google.com.js`

And it returns a bundled version of all scripts it can find for each type (Javascript and CSS), ready to be executed by the extension. If there were a `com.js` and a `www.google.com.js`, the resulting script would be a concatenation of them. The same with CSS.

Defunkt's original dot.js server ran over HTTPS, since Chrome complains if you request something over HTTP on a HTTPS page. This is called "mixed content" (see [this explanation](https://developers.google.com/web/fundamentals/security/prevent-mixed-content/what-is-mixed-content)).

This new approach allow us to request HTTP just fine, though. Chrome extensions have a foreground environment, where you have access to a page's content, and a background one, where your scripts run in a exclusive process created just for your extension. This same background process is shared among all your foreground instances. It also happens that the background process can request whatever it wants, including HTTP (non-secure) requests; and this is just what we need.

# To do

- rename the extension to something that makes more sense, since the extension now also handles CSS;
- implement include directives for CSS
- improve UI by coloring the icon when at least one script was loaded
- allow for loading of custom images;
- cache scripts in memory to avoid going to the disk all the time
  (but will have to find out when scripts change on disk, otherwise cache will serve outdated stuff)
- think of some way to avoid hitting the disk every time some page is loaded, given that most won't have a matching script
  (keep an up-to-date set of all existing script names in memory to quickly answer requests)

# Credits

* [defunkt](https://github.com/defunkt) and his [original implementation](https://github.com/defunkt/dotjs);
* witch icon downloaded from https://www.flaticon.com/free-icon/witch_477108.

# Other approaches

[Matthew Hadley](https://github.com/diffsky) made [an experiment](https://github.com/diffsky/chromedotfiles) using the [chrome.tabs API](https://developer.chrome.com/extensions/tabs) to load js and css without the need of a web server. The drawback is that you have to reload the extension every time a new script is added or updated. That's why I decided to have a web server running; I want script loading to be as seamless as possible.

There's also [Jonathan Cremin](https://github.com/kudos)'s [Punkjs](https://github.com/kudos/punkjs), but I tried and it is currently not working (the app breaks when you select the .js folder) and I think that between extension+server and extension+app, I prefer the former. You can just schedule the server to run every time your system boots and then forget about it. The Chrome app approach expects you to open the app every time you open your browser. Besides, it seems to be no longer maintained as well.
