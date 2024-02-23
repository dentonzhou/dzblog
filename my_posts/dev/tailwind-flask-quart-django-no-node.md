---
title: "Tailwind in Flask and Quart Without Node"
date: 2024-02-20T15:02:20
weight: 50
subtitle: ""
subheading: ""

author: "denton"
snippet: "You don't need Node to use Tailwind on your Flask or Quart project. Here's how you can use Tailwind's standalone CLI to watch your files and create production-ready CSS files."
description: ""
sidebar: "dev"
category: ""

prevPost: ""
nextPost: ""

image: ""
imagePositionX: 50
imagePositionY: 50
showImageInHeader: true

isVisible: true
isArchived: false
isContentHidden: false
contentMessage: ""

relatedPost1: ""
relatedPost2: ""
relatedPost3: ""
relatedPost4: ""
relatedPost5: ""
relatedPost6: ""
relatedPost7: ""
relatedPost8: ""
relatedPost9: ""
---

In the past few years, I've been using either Vue, React, or a framework built on the two. For these, I'd use Node, which made Tailwindcss relatively trivial to set up.

More recently, I decided to rework parts of OpenCourser, which is built on Flask.

I have a few lingering questions about this rework. One of these, whether I should stick with Flask or try Quart or something else entirely (Go and HTMX momentarily piqued my interest), is why this article deals with both Flask *and* Quart.

But in this post, my primary concern is whether I could use Tailwind without Node.

> If you don't need anymore backstory or rationale, jump ahead to [the set up instructions](#setting-up-the-standalone-cli)

> I've created two repositories that will help you get started more quickly:
> [flask-tailwind-no-node](https://github.com/dentonzh/flask-tailwind-no-node) and [quart-tailwind-no-node](https://github.com/dentonzh/quart-tailwind-no-node)

## Why Tailwind without Node?
I'm opting out of Node for my Flask / Quart projects because:

1. I want to keep my project directory free of clutter
2. Node, even at idle, consumes resources

Point number one is purely a matter of personal preference. I'd prefer to keep my Python projects free of `node_modules`, `package.json`, and `package-lock.json`. I see them all too much when working on my frontend projects, so I think this is perfectly reasonable.

On the latter point, I sometimes `ssh` directly into my server to make quick fixes. I'll do this either from another machine that doesn't have the whole environment set up. Often it's from my phone (using the fantastic UserLand app).

Usually, the fixes are urgent in nature, a 500 error on a route with moderate traffic or data not showing up correctly. But every now and then I need to make a tweak to my templates. And what if I need to adjust a Tailwind class?

The last thing I'd want is to keep Node running just to rebuild my CSS every now and again.

You might say, "well, Node doesn't really consume that many resources," and you'd be correct.

But I run OpenCourser on a droplet with 1GiB of memory. I'm already eating up a lot of memory if you account for the operating system, the app itself, nginx, gunicorn, Elasticsearch, MySQL, etc. Even at idle, Node would consume a good chunk of whatever free memory remains.

## Three possible solutions
The easiest solution I thought of was simply to pull Tailwind into the app using a CDN. As of 3.7.1, Tailwind weighs in at 370KB minified, so this is very much a sub-optimal solution. It certainly wouldn't be viable to do this in production.

The second solution is to run Node in my development environment. I would forego having a tidy project directory or being able to edit templates by `ssh`ing into my server. But at least I'd still get to enjoy Tailwind.

Or, the solution I ultimately settled on, I could use Tailwind's standalone CLI. This CLI conveniently does everything Tailwind does when it's run via Node. It only runs in the background during development. 

Unlike Tailwind on Node, however, it does take some setting up.

But if you're already familiar with setting up Flask and Quart projects, this process should be fairly straightforward.

## Set up the standalone CLI

> The instructions here assume you already have a Flask or Quart project set up already with all of the necessary dependencies installed in a virtual environment.

On Linux and Mac, launch your terminal in your project directory or `cd` into it. 

We'll start by downloading Tailwind's standalone CLI, making it executable, and then renaming our executable to tailwindcss:

```shell session
$ curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-linux-x64
chmod +x tailwindcss-linux-x64
mv tailwindcss-linux-x64 tailwindcss
```

If you're not using Linux on x64 architecture, replace `tailwindcss-linux-x64` with one of the following:
- `tailwindcss-linux-arm64`
- `tailwindcss-linux-armv7`
- `tailwindcss-macos-arm64`
- `tailwindcss-macos-x64`

On Windows, download the latest `.exe` executable to your project directory:
x64: [tailwindcss-windows-x64.exe](https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-windows-x64.exe)
arm64: [tailwindcss-windows-arm64.exe](https://github.com/tailwindlabs/tailwindcss/releases/download/v3.4.1/tailwindcss-windows-arm64.exe)

## Ignoring the executable in version control
If you're using version control, make sure to ignore this executable so as not to include it in your repository.

I'm using Git, so I simply need to add one line to my `.gitignore` file:

```
...

tailwindcss
```

## Initialize Tailwind in your project
In the root of your project folder, run `./tailwindcss init`, which generates `tailwind.config.js`. Your project directory will vary. Mine looks like this:

```
project
├── app
│  ├── __init__.py
│  └── templates
│     ├── base.html
│     └── index.html
└── tailwind.config.js
```

## Setting up our CSS and template files
Next, we'll create our `input.css` and `output.css` files

Where you create these files will vary depending on the framework you choose and any conventions you might follow (e.g. blueprints).

For purposes of demonstration, we'll keep our CSS files in `app/static/css`.

```
project
├── app
│  ├── __init__.py
│  ├── static
│  │  └── css
│  │     ├── input.css
│  │     └── output.css
│  └── templates
│     ├── base.html
│     └── index.html
└── tailwind.config.js
```

For testing purposes, we'll also set up our template files, `base.html` and `index.html`:

`base.html`

```python
<!DOCTYPE html>
<html lang="en">
    <meta charset="UTF-8">
    <link href="/static/css/output.css" rel="stylesheet">

    <title>{% block title %}{% endblock title %}</title>
    <div>
        {% block content %}{% endblock content %}
    </div>
</html>

```

`index.html`

```python
{% extends 'base.html' %}

{% block title %}
    {{ title }}
{% endblock title %}

{% block content %}
    <div class="bg-gray-300 text-red-600">{{ title }}</div>
{% endblock content %}
```

Because Tailwind will automatically generate our `output.css` file, we won't need to ever need to modify this file ourselves.

`input.css` is a different story. In order for the `tailwindcss` CLI to work correctly, we'll need to add Tailwind directives to this file:

```css
/* Import Tailwind's base styles */
@import 'tailwindcss/base';

/* Import Tailwind's components */
@import 'tailwindcss/components';

/* Import Tailwind's utilities */
@import 'tailwindcss/utilities';
```

You can modify this file further as needed. To see how, see [Functions & Directives](https://tailwindcss.com/docs/functions-and-directives) from the Tailwind docs.

## Have Tailwind watch for changes
We've laid the groundwork for using Tailwind without Node. The last two steps will help Tailwind detect changes in our template files and generate a new CSS file for us.

First, we'll open `tailwind.config.js`. In `module.exports`, we'll add a string to the array contained in the `content` key that tells Tailwind which files to watch:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './app/templates/**/*.html',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```


Finally, in our terminal, we'll run the following `tailwindcss` command:

```shell session
./tailwindcss -i app/static/css/input.css -o app/static/css/output.css --watch
```

You'll need to start this command each time you start a fresh session (e.g. after a restart). 

> Be mindful of where you store your CSS and template files. Your `tailwind.config.js` and terminal command to run `tailwindcss` will differ from mine if your paths differ.

To test if everything is working properly, run your app in debug mode. As the development server is running, make some changes to `index.html` like so:

```python
{% extends 'base.html' %}

{% block title %}
    {{ title }}
{% endblock title %}

{% block content %}
    <div class="bg-gray-800 text-green-600 font-bold italic text-center text-3xl">{{ title }}</div>
{% endblock content %}
```

Save your file and refresh your browser. You should now see your changes applied. Note that you may need to do a full refresh (Ctrl + F5 / Cmd + Shift + R) to see your changes.

## Output CSS for production
When you're ready to deploy your app in a production setting, run the following command:

```shell session
./tailwindcss -i app/static/css/input.css -o app/static/css/output.css --minify
```

This command minifies the CSS file that Tailwind outputs, eliminating as many characters
from it as possible to make it space-efficient.

> Alternatively, you could set up a minifier with your Flask or Quart project. If you use a service like Cloudflare, you may also look into configuring something like [Auto Minify](https://developers.cloudflare.com/speed/optimization/content/auto-minify/).

## Run Tailwind automatically when starting in debug mode
We now have all of the pieces we need to use Tailwind without Node.

For most, this should be good enough, but if you're like me, you'll want to streamline how you run Flask / Quart with Tailwind in development.

Right now, we need to call either `flask run` or `quart run` *and* `tailwindcss` with the `--watch` flag. For those counting, that's two terminal windows and two commands that you have to execute just to be able to work on your templates!

To consolidate things, you could write a shell script. But it's even easier just to call `tailwindcss` from your app.
### Set up tailwindcss in your project folder
In an ideal world, we'd only need to call `flask run --debug` or `quart run --debug` to run our app in development mode *with* `tailwindcss` watching our templates.

Thankfully, we can make use of Python's `subprocess` module to achieve exactly this.

Specifically, we'll check if we're running the app in debug mode. If we are, we'll use `subprocess.Popen()` to run `tailwindcss` with the `--watch` flag.

The code to do this varies slightly between Flask and Quart, so I've split the following steps into two different sections.

### Calling tailwindcss from Flask
In Flask, we'll check `app.debug` to see if we're in development mode. If we are, we'll go ahead and call `subprocess.Popen()` like so:

```python
from flask import Flask, render_template
import subprocess

app = Flask(__name__)
app.debug = True

if app.debug:
    subprocess.Popen(
        [
            './tailwindcss',
            '-i',
            'app/static/css/input.css',
            '-o',
            'app/static/css/output.css',
            '--watch',
        ]
    )

@app.route('/')
def hello():
    return render_template('index.html', title='Hello', text='Hi!')


app.run(debug=True)
```

## Calling tailwindcss from Quart
We'll also call `subprocess.Popen()` to spawn our process in Quart. The key difference from Flask is that we'll need to use the `@app.before_serving` decorator, which helps us run setup tasks before Quart begins serving requests.

```python
from quart import Quart, render_template
import subprocess

app = Quart(__name__)
app.debug = True

@app.before_serving
def run_tailwind_cli():
    if app.debug:
        subprocess.Popen(
            [
                './tailwindcss',
                '-i',
                'app/static/css/input.css',
                '-o',
                'app/static/css/output.css',
                '--watch',
            ]
        )

@app.route('/')
async def hello():
    return await render_template('blog/index.html', title='Hello', text='Hi!')

app.run(debug=True)
```

## Recap
In this post, we downloaded Tailwind's standalone CLI to our project directory. In Mac and Linux, we used `curl` to do this and used `chmod` to make `tailwindcss` executable. On Windows, we saved this file via the browser.

In our project directory, we called `./tailwindcss init` to create `tailwind.config.js`. 

We modified this file to tell `tailwindcss` where our templates are. We manually created an `input.css`, populating it with a set of Tailwind directives we wish to use. We also created an empty `output.css` that Tailwind will populate.

To ensure that the `tailwindcss` executable doesn't get merged into our repository, we added it to `.gitignore` (if using Git).

Finally, we ran `tailwindcss` from the terminal with the `--watch` flag, passing to it parameters that tell it where our `input.css` and `output.css` files are.

And that's it!

We're now able to use Tailwind classes in our Python web project without needing to install and run Node.