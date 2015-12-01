# Emacs - the best python editor?

Thus far, in our series of posts on Python development environments we've looked at Sublime Text and VIM:

- [Setting Up Sublime Text 3 for Full Stack Python Development](https://realpython.com/blog/python/setting-up-sublime-text-3-for-full-stack-python-development/)
- [VIM and Python - a Match Made in Heaven](https://realpython.com/blog/python/vim-and-python-a-match-made-in-heaven/)

In this post we'll present another powerful editor for Python development - [Emacs](https://www.gnu.org/software/emacs/). **While it's an indisputable fact that Emacs is the best editor, we'll (try to) keep an open mind and present Emacs objectively, from a fresh installation to a complete Python IDE so that you can make an informed decision when choosing your go-to Python IDE.**

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-logo-with-python.png" style="max-width: 100%;" alt="emacs and python logos">
</div>

<br>

*This is a guest blog post by [Kyle Purdon](https://twitter.com/PurdonKyle)​, an Application Engineer at [Bitly](https://bitly.com/pages/careers) in Denver.*

## Installation and Basics

### Installation

Installation of Emacs is a topic that does not need to be covered in yet another blog post. [This Guide](http://ergoemacs.org/emacs/which_emacs.html), provided by [ErgoEmacs](http://ergoemacs.org/), will get you up and running with a basic Emacs installation on Linux, Mac, or Windows. Once installed, start the application and you will be greeted with the default configured Emacs:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-fresh-launch.png" style="max-width: 100%;" alt="emacs fresh launch">
</div>

### Basic Emacs

Another topic that does not need to be covered again is the basics of using Emacs. The easiest way to learn Emacs is to follow the built-in tutorial. The topics covered in this post do not require that you know how to use Emacs yet; instead, each topic highlights what you can do after learning the basics.

To enter the tutorial, use your arrow keys to position the cursor over the words "Emacs Tutorial" and press Enter. You will then be greeted with the following passage:

```
Emacs commands generally involve the CONTROL key (sometimes labeled
CTRL or CTL) or the META key (sometimes labeled EDIT or ALT).  Rather than
write that in full each time, we'll use the following abbreviations:

 C-<chr>  means hold the CONTROL key while typing the character <chr>
    Thus, C-f would be: hold the CONTROL key and type f.
 M-<chr>  means hold the META or EDIT or ALT key down while typing <chr>.
    If there is no META, EDIT or ALT key, instead press and release the
    ESC key and then type <chr>.  We write <ESC> for the ESC key.
```

So, key entries/commands like `C-x C-s` (which is used to save) will be shown throughout the remainder of the post. This command indicates that the CONTROL and X key are pressed at the same time, and then the CONTROL and S key. This is the basic form of interacting with Emacs. Please follow the built-in tutorial as well as the [Guided Tour of Emacs](http://www.gnu.org/software/emacs/tour/) to learn more.

### Configuration and Packages

One of the great benefits of Emacs is the simplicity and power of configuration. The core of Emacs configuration is the [Initialization File](http://www.gnu.org/software/emacs/manual/html_node/emacs/Init-File.html), `init.el`.

In a UNIX environment this file should be placed in *$HOME/.emacs.d/init.el*:

```sh
$ touch ~/.emacs.d/init.el
```

Meanwhile, in Windows, if the *HOME* environment variable is not set, this file should reside in  `C:/.emacs.d/init.el`. See [GNU Emacs FAQ for MS Windows > Where do I put my init file?](http://www.gnu.org/software/emacs/manual/html_node/efaq-w32/Location-of-init-file.html#Location-of-init-file) for more info.

> Configuration snippets will be presented throughout the post. So, create the init file now if you want to follow along. Otherwise you can find the complete file in the [Conclusion](#conclusion).

[Packages](https://www.gnu.org/software/emacs/manual/html_node/emacs/Packages.html) are used to customize Emacs, which are sourced from a number of repositories. The primary Emacs package repository is [MELPA](https://melpa.org/#/). All of the packages presented in this post will be installed from this repository.

### Styling (Themes & More)

To begin, the following configuration snippet provides package installation and installs a theme package:

```
;; init.el --- Emacs configuration

;; INSTALL PACKAGES
;; --------------------------------------

(require 'package)

(add-to-list 'package-archives
       '("melpa" . "http://melpa.org/packages/") t)

(package-initialize)
(when (not package-archive-contents)
  (package-refresh-contents))

(defvar myPackages
  '(better-defaults
    material-theme))

(mapc #'(lambda (package)
    (unless (package-installed-p package)
      (package-install package)))
      myPackages)

;; BASIC CUSTOMIZATION
;; --------------------------------------

(setq inhibit-startup-message t) ;; hide the startup message
(load-theme 'material t) ;; load material theme
(global-linum-mode t) ;; enable line numbers globally

;; init.el ends here
```

The first section of the configuration snippet, `;; INSTALL PACKAGES`, installs two packages called *[better-defaults](http://melpa.org/#/better-defaults)* and [material-theme](http://melpa.org/#/material-theme). The *better-defaults* package is a collection of minor changes to the Emacs defaults that makes a great base to begin customizing from. The *material-theme* package provides a customized set of styles.

> My preferred theme is [material-theme](http://melpa.org/#/material-theme), so we'll be using that for the rest of the post.

The second section `;; BASIC CUSTOMIZATION`:

1. Disables the startup message (this is the screen with all the tutorial information). *You may want to leave this out until you are more comfortable with Emacs.*
1. Loads the material theme.
1. Enables line numbers globally.

Enabling something globally means that it will apply to all [buffers](http://www.gnu.org/software/emacs/manual/html_node/elisp/Buffers.html) (open items) in Emacs. So if you open a Python, markdown, and/or text file, they will all have line numbers shown. You can also enable things per mode - e.g., python-mode, markdown-mode, text-mode. This will be shown later when setting up Python.

Now that we have a complete *basic* configuration file we can restart Emacs and see the changes. If you placed the `init.el` file in the correct default location it will automatically be picked up.

As an alternative, you can start Emacs from the command-line with `emacs -q --load <path to init.el>`. When loaded, our initial Emacs window looks a bit nicer:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-themed.png" style="max-width: 100%;" alt="emacs themed">
</div>

The following image shows some other basic features that come with Emacs out of the box -  including simple file searching and split layouts:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-simple-features.png" style="max-width: 100%;" alt="emacs simple features">
</div>

One of my favorite simple features of Emacs is being able to do a quick [recursive-grep search](http://www.gnu.org/software/emacs/manual/html_node/emacs/Grep-Searching.html) - `M-x rgrep` For example, say you want to find all instances of the word *python* in any *.md* (markdown) in a given directory:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-rgrep.gif" style="max-width: 100%;" alt="emacs rgrep">
</div>

With this basic configuration complete we can begin to dive into configuring the environment for Python development!

## Elpy - Python Development

Emacs is distributed with a python-mode (`python.el`) that provides indentation and syntax highlighting. However, to compete with Python-specific IDE's (Integrated Development Environments), we'll certainly want more. The [elpy](https://elpy.readthedocs.org/en/latest/) (Emacs Lisp Python Environment) package provides us with a near complete set of Python IDE features, including:

- Automatic Indentation,
- Syntax Highlighting,
- Auto-Completion,
- Syntax Checking,
- Python REPL Integration,
- Virtual Environment Support, and
- Much [more](https://elpy.readthedocs.org/en/latest/ide.html)!

To install and enable elpy we need to add a bit of configuration and install `flake8` and `jedi` using your preferred method for installing Python packages (pip or conda, for example).

The following will install the elpy package:

```
(defvar myPackages
  '(better-defaults
    elpy ;; add the elpy package
    material-theme))
```

Now just enable it:

```
(elpy-enable)
```

With the new configuration, we can restart Emacs and open up a Python file to see the new mode in action:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-elpy-basic.png" style="max-width: 100%;" alt="emacs elpy basic">
</div>

Shown in this image are the following features:

- Automatic Indentation (as you type and hit RET lines are auto-indented)
- Syntax Highlighting
- Syntax Checking (error indicators at line 3)
- Auto-Completion (list methods on line 9+)

In addition, let's say we want to run this script. In something like IDLE or Sublime Text you'll have a button/command to run the current script. Emacs is no different, we just type `C-c C-c` in our Python buffer and...

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-elpy-execute.png" style="max-width: 100%;" alt="emacs elpy execute">
</div>

Often we'll want to be running a virtual environment and executing our code using the packages installed there. To use a virtual environment in Emacs, we type `M-x pyvenv-activate` and follow the prompts. You can deactivate a virtualenv with `M-x pyvenv-deactivate`. Elpy provides an interface for debugging the environment and any issues you may encounter with elpy itself. By typing `M-x elpy-config` we get the following dialog, which provides valuable debugging information:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-elpy-config.png" style="max-width: 100%;" alt="emacs elpy config">
</div>

With that, all of the basics of a Python IDE in Emacs have been covered. Now let's put some icing on this cake!

## Additional Python Features

In addition to all the basic IDE features described above, Emacs provides some additional features for Python. While this is not an exhaustive list, PEP8 compliance (with autopep8) and integration with IPython/Jupyter will be covered. However, before that let's cover a quick syntax checking preference.

### Better Syntax Checking (Flycheck v. Flymake)

By default Emacs+elpy comes with a package called *[Flymake](https://www.gnu.org/software/emacs/manual/html_node/flymake/Overview-of-Flymake.html#Overview-of-Flymake)* to support syntax checking. However, another Emacs package, *[Flycheck](http://www.flycheck.org/)*, is available and supports realtime syntax checking. Luckily switching out for *Flycheck* is simple:

```
(defvar myPackages
  '(better-defaults
    elpy
    flycheck ;; add the flycheck package
    material-theme))
```
and

```
(when (require 'flycheck nil t)
  (setq elpy-modules (delq 'elpy-module-flymake elpy-modules))
  (add-hook 'elpy-mode-hook 'flycheck-mode))
```

Now we get realtime feedback when editing Python code:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-elpy-flycheck.gif" style="max-width: 100%;" alt="emacs elpy flycheck">
</div>

### PEP8 Compliance (Autopep8)

Love it or hate it, [PEP8](https://www.python.org/dev/peps/pep-0008/) is here to stay. If you want to follow all or some of the standards, you'll probably want an automated way to do so. The *[autopep8](https://pypi.python.org/pypi/autopep8/)* tool is the solution. It integrates with Emacs so that when you save - `C-x C-s` - *autopep8* will automatically format and correct any PEP8 errors (excluding any you wish to).

First, you will need to install the Python package *autopep8* using your preferred method, then add the following Emacs configuration:

```
(defvar myPackages
  '(better-defaults
    elpy
    flycheck
    material-theme
    py-autopep8)) ;; add the autopep8 package
```
and

```
(require 'py-autopep8)
(add-hook 'elpy-mode-hook 'py-autopep8-enable-on-save)
```

Now (after forcing some pep8 errors) when we save our demo Python file, the errors will automatically be corrected:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-elpy-autopep8.gif" style="max-width: 100%;" alt="emacs elpy autopep8">
</div>

### IPython/Jupyter Integration

Next up is a really cool feature: Emacs integration with the IPython REPL and Jupyter Notebooks. First, let's look at swapping the standard Python REPL integration for the IPython version:

```
(elpy-use-ipython)
```

Now when we run our Python code with `C-c C-c` we will be presented with the IPython REPL:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-elpy-ipython.png" style="max-width: 100%;" alt="emacs elpy ipython">
</div>

While this is pretty useful on its own, the real magic is the notebook integration. We'll assume that you already know how to launch a Jupyter Notebook server (if not [check this out](http://jupyter-notebook-beginner-guide.readthedocs.org/en/latest/what_is_jupyter.html)). Again we just need to add a bit of configuration:

```
(defvar myPackages
  '(better-defaults
    ein ;; add the ein package (Emacs ipython notebook)
    elpy
    flycheck
    material-theme
    py-autopep8))
```

The standard Jupyter web interface for notebooks is nice but requires us to leave Emacs to use:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/jupyter-web.png" style="max-width: 100%;" alt="jupyter web">
</div>

However, we can complete the exact same task by connecting to and interacting with the notebook server directly in Emacs.

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/emacs-elpy-jupyter.gif" style="max-width: 100%;" alt="emacs elpy jupyter">
</div>

## Additional Emacs Features

Now that all of the basic Python IDE features (and some really awesome extras) have been covered, there are a few other things that an IDE should be able to handle. First up is git integration...

### Git Integration (Magit)

*[Magit](http://magit.vc/)* is the most popular non-utility package on MELPA and is used by nearly every Emacs user who uses git. It's incredibly powerful and far more comprehensive than we can cover in this post. Luckily [Mastering Emacs](https://www.masteringemacs.org/) has a great post covering Magit [here](https://www.masteringemacs.org/article/introduction-magit-emacs-mode-git). The following image is from the Mastering Emacs post and gives you a taste for what the git integration looks like in Emacs:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/emacs/masteringemacs-magit.png" style="max-width: 100%;" alt="mastering emacs magit">
</div>

### Other Modes

One of the major benefits of using Emacs over a Python-specific IDE is that you get compatibility with much more than just Python. In a single day I often work with Python, Golang, JavaScript, Markdown, JSON, and more. Never leaving Emacs and having complex support for all of these languages in a single editor is very efficient. You can check out my personal Emacs configuration [here](https://github.com/kpurdon/kp-emacs). It includes support for:

- Python
- Golang
- Ruby
- Puppet
- Markdown
- Dockerfile
- YAML
- Web (HTML/JS/CSS)
- SASS
- NginX Config
- SQL

In addition to lots of other Emacs configuration goodies.

### Emacs In The Terminal

After learning Emacs you'll want Emacs keybindings everywhere. This is as simple as typing `set -o emacs` at your bash prompt. However, one of the powers of Emacs is that you can run Emacs itself in headless mode in your terminal. This is my default environment. To do so, just start Emacs by typing `emacs -nw` at your bash prompt and you'll be running a headless Emacs.

## Conclusion

As you can see, Emacs is clearly the best editor... To be fair, there are a lot of great options out there for Python IDEs, but I would absolutely recommend learning either Vim or Emacs as they are by far the most versatile development environments possible. I said I'd leave you with the complete Emacs configuration, so here it is:

```
;; init.el --- Emacs configuration

;; INSTALL PACKAGES
;; --------------------------------------

(require 'package)

(add-to-list 'package-archives
       '("melpa" . "http://melpa.org/packages/") t)

(package-initialize)
(when (not package-archive-contents)
  (package-refresh-contents))

(defvar myPackages
  '(better-defaults
    ein
    elpy
    flycheck
    material-theme
    py-autopep8))

(mapc #'(lambda (package)
    (unless (package-installed-p package)
      (package-install package)))
      myPackages)

;; BASIC CUSTOMIZATION
;; --------------------------------------

(setq inhibit-startup-message t) ;; hide the startup message
(load-theme 'material t) ;; load material theme
(global-linum-mode t) ;; enable line numbers globally

;; PYTHON CONFIGURATION
;; --------------------------------------

(elpy-enable)
(elpy-use-ipython)

;; use flycheck not flymake with elpy
(when (require 'flycheck nil t)
  (setq elpy-modules (delq 'elpy-module-flymake elpy-modules))
  (add-hook 'elpy-mode-hook 'flycheck-mode))

;; enable autopep8 formatting on save
(require 'py-autopep8)
(add-hook 'elpy-mode-hook 'py-autopep8-enable-on-save)

;; init.el ends here
```

Hopefully this configuration will spark your Emacs journey!
