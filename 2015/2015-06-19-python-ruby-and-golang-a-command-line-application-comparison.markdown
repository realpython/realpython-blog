# Python, Ruby, and Golang: A Command-Line Application Comparison

This is a guest blog post by **[Kyle Purdon](https://twitter.com/PurdonKyle)​**, a software engineer in Boulder, CO. Kyle is a Python first developer with experience in Ruby, Golang, and many more languages. This post was originally authored on Kyle's personal blog and included great discussion on [Reddit](http://www.reddit.com/r/programming/comments/37x1o0/python_ruby_and_golang_a_commandline_application/).

<hr>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/python-ruby-golang/python-ruby-golang3.png" style="max-width: 100%;" alt="python, ruby, and golang images">
</div>

<br>

In late 2014 I built a tool called [pymr](https://github.com/kpurdon/pymr). I recently felt the need to learn golang and refresh my ruby knowledge so I decided to revisit the idea of pymr and build it in multiple languages. **In this post I will break down the "mr" (merr) application (pymr, gomr, rumr) and present the implementation of specific pieces in each language. I will provide an overall personal preference at the end but will leave the comparison of individual pieces up to you.**

> For those that want to skip directly to the code check out the [repo](https://github.com/realpython/python-ruby-golang).

## Application Structure

The basic idea of this application is that you have some set of related directories that you want to execute a single command on. The "mr" tool provides a method for registering directories, and a method for running commands on groups of registered directories. The application has the following components:

* A command-line interface
* A registration command (writes a file with given tags)
* A run command (runs a given command on registered directories)

## Command-Line Interface

The command line interface for the "mr" tools is:

```sh
$ pymr --help
Usage: pymr [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  register  register a directory
  run       run a given command in matching...
```

To compare building the command-line interface let's take a look at the register command in each language.

**Python (pymr)**

To build the command line interface in python I chose to use the [click](http://click.pocoo.org/4/) package.

```python
@pymr.command()
@click.option('--directory', '-d', default='./')
@click.option('--tag', '-t', multiple=True)
@click.option('--append', is_flag=True)
def register(directory, tag, append):
    ...
```

**Ruby (rumr)**

To build the command line interface in ruby I chose to use the [thor](http://whatisthor.com/) gem.

```ruby
desc 'register', 'Register a directory'
method_option :directory,
              aliases: '-d',
              type: :string,
              default: './',
              desc: 'Directory to register'
method_option :tag,
              aliases: '-t',
              type: :array,
              default: 'default',
              desc: 'Tag/s to register'
method_option :append,
              type: :boolean,
              desc: 'Append given tags to any existing tags?'
def register
  ...
```

**Golang (gomr)**

To build the command line interface in Golang I chose to use the [cli.go](https://github.com/codegangsta/cli) package.

```go
app.Commands = []cli.Command{
        {
            Name:   "register",
            Usage:  "register a directory",
            Action: register,
            Flags: []cli.Flag{
                cli.StringFlag{
                    Name:  "directory, d",
                    Value: "./",
                    Usage: "directory to tag",
                },
                cli.StringFlag{
                    Name:  "tag, t",
                    Value: "default",
                    Usage: "tag to add for directory",
                },
                cli.BoolFlag{
                    Name:  "append",
                    Usage: "append the tag to an existing registered directory",
                },
            },
        },
```

## Registration

The registration logic is as follows:

1. If the user asks to `--append` read the `.[py|ru|go]mr` file if it exists.
2. Merge the existing tags with the given tags.
3. Write a new `.[...]mr` file with the new tags.

This breaks down into a few small tasks we can compare in each language:

* Searching for and reading a file.
* Merging two items (only keeping the unique set)
* Writing a file

### File Search

**Python (pymr)**

For python this involves the [os](https://docs.python.org/2/library/os.html) module.

```python
pymr_file = os.path.join(directory, '.pymr')
if os.path.exists(pymr_file):
    ...
```

**Ruby (rumr)**

For ruby this involves the [File](http://ruby-doc.org/core-1.9.3/File.html) class.

```ruby
rumr_file = File.join(directory, '.rumr')
if File.exist?(rumr_file)
  ...
```


**Golang (gomr)**

For golang this involves the [path](http://golang.org/pkg/path/) package.

```go
fn := path.Join(directory, ".gomr")
if _, err := os.Stat(fn); err == nil {
   ...
```

### Unique Merge

**Python (pymr)**

For python this involves the use of a [set](https://docs.python.org/2/library/sets.html).

```python
# new_tags and cur_tags are tuples
new_tags = tuple(set(new_tags + cur_tags))
```

**Ruby (rumr)**

For ruby this involves the use of the [.uniq](http://ruby-doc.org/core-2.2.0/Array.html#method-i-uniq) array method.

```ruby
# Edited (5/31)
# old method:
#  new_tags = (new_tags + cur_tags).uniq

# new_tags and cur_tags are arrays
new_tags |= cur_tags
```

**Golang (gomr)**

For golang this involves the use of custom function.

```go
func AppendIfMissing(slice []string, i string) []string {
    for _, ele := range slice {
        if ele == i {
            return slice
        }
    }
    return append(slice, i)
}

for _, tag := range strings.Split(curTags, ",") {
                newTags = AppendIfMissing(newTags, tag)
            }
```

### File Read/Write

I tried to choose the simplest possible file format to use in each language.

**Python (pymr)**

For python this involves the use of the [pickle](https://docs.python.org/2/library/pickle.html) module.

```python
# read
cur_tags = pickle.load(open(pymr_file))

# write
pickle.dump(new_tags, open(pymr_file, 'wb'))
```

**Ruby (rumr)**

For ruby this involves the use of the [YAML](http://ruby-doc.org/stdlib-2.2.1/libdoc/yaml/rdoc/YAML.html) module.

```ruby
# read
cur_tags = YAML.load_file(rumr_file)

# write
# Edited (5/31)
# old method:
#  File.open(rumr_file, 'w') { |f| f.write new_tags.to_yaml }
IO.write(rumr_file, new_tags.to_yaml)
```

**Golang (gomr)**

For golang this involves the use of the [config](https://github.com/robfig/config) package.

```go
// read
cfg, _ := config.ReadDefault(".gomr")

// write
outCfg.WriteFile(fn, 0644, "gomr configuration file")
```

## Run (Command Execution)

The run logic is as follows:

1. Recursively walk from the given basepath searching for `.[...]mr` files
2. Load a found file, and see if the given tag is in it
3. Call the given command in the directory of a matching file.

This breaks down into a few small tasks we can compare in each language:

* Recursive Directory Search
* String Comparison
* Calling a Shell Command

### Recursive Directory Search

**Python (pymr)**

For python this involves the [os](https://docs.python.org/2/library/os.html) module and [fnmatch](https://docs.python.org/2/library/fnmatch.html) module.

```python
for root, _, fns in os.walk(basepath):
        for fn in fnmatch.filter(fns, '.pymr'):
            ...
```

**Ruby (rumr)**

For ruby this involves the [Find](http://ruby-doc.org/stdlib-2.2.0/libdoc/find/rdoc/Find.html) and [File](http://ruby-doc.org/core-2.2.0/File.html) classes.

```ruby
# Edited (5/31)
# old method:
#  Find.find(basepath) do |path|
#        next unless File.basename(path) == '.rumr'
Dir[File.join(options[:basepath], '**/.rumr')].each do |path|
  ...
```

**Golang (gomr)**

For golang this requires the [filepath](http://golang.org/pkg/path/filepath/) package and a custom callback function.

```go
func RunGomr(ctx *cli.Context) filepath.WalkFunc {
    return func(path string, f os.FileInfo, err error) error {
        ...
        if strings.Contains(path, ".gomr") {
            ...

filepath.Walk(root, RunGomr(ctx))
```

### String Comparison

**Python (pymr)**

Nothing additional is needed in python for this task.

```python
if tag in cur_tags:
    ...
```

**Ruby (rumr)**

Nothing additional is needed in ruby for this task.

```ruby
if cur_tags.include? tag
  ...
```

**Golang (gomr)**

For golang this requires the [strings](http://golang.org/pkg/strings/) package.

```go
if strings.Contains(cur_tags, tag) {
    ...
```

### Calling a Shell Command

**Python (pymr)**

For python this requires the [os](https://docs.python.org/2/library/os.html) module and the [subprocess](https://docs.python.org/2/library/subprocess.html) module.

```python
os.chdir(root)
subprocess.call(command, shell=True)
```

**Ruby (rumr)**

For ruby this involves the [Kernel](http://ruby-doc.org/core-2.2.0/Kernel.html#method-i-system) module and the Backticks syntax.

```ruby
# Edited (5/31)
# old method
#  puts `bash -c "cd #{base_path} && #{command}"`
Dir.chdir(File.dirname(path)) { puts `#{command}` }
```

**Golang (gomr)**

For golang this involves the [os](https://golang.org/pkg/os/) package and the [os/exec](https://golang.org/pkg/os/exec/) package.

```go
os.Chdir(filepath.Dir(path))
cmd := exec.Command("bash", "-c", command)
stdout, err := cmd.Output()
```

## Packaging

The ideal mode of distribution for this tool is via a package. A user could then install it `tool install [pymr,rumr,gomr]` and have a new command on there systems path to execute. I don't want to go into packaging systems here, rather I will just show the basic configuration file needed in each language.

**Python (pymr)**

For python a `setup.py` is required. Once the package is created and uploaded it can be installed with `pip install pymr`.

```python
from setuptools import setup, find_packages

classifiers = [
    'Environment :: Console',
    'Operating System :: OS Independent',
    'License :: OSI Approved :: GNU General Public License v3 or later (GPLv3+)',
    'Intended Audience :: Developers',
    'Programming Language :: Python',
    'Programming Language :: Python :: 2',
    'Programming Language :: Python :: 2.7'
]

setuptools_kwargs = {
    'install_requires': [
        'click>=4,<5'
    ],
    'entry_points': {
        'console_scripts': [
            'pymr = pymr.pymr:pymr',
        ]
    }
}


setup(
    name='pymr',
    description='A tool for executing ANY command in a set of tagged directories.',
    author='Kyle W Purdon',
    author_email='kylepurdon@gmail.com',
    url='https://github.com/kpurdon/pymr',
    download_url='https://github.com/kpurdon/pymr',
    version='2.0.1',
    packages=find_packages(),
    classifiers=classifiers,
    **setuptools_kwargs
)
```

**Ruby (rumr)**

For ruby a `rumr.gemspec` is required. Once the gem is created and uploaded is can be installed with `gem install rumr`.

```ruby
Gem::Specification.new do |s|
  s.name        = 'rumr'
  s.version     = '1.0.0'
  s.summary     = 'Run system commands in sets' \
                  ' of registered and tagged directories.'
  s.description = '[Ru]by [m]ulti-[r]epository Tool'
  s.authors     = ['Kyle W. Purdon']
  s.email       = 'kylepurdon@gmail.com'
  s.files       = ['lib/rumr.rb']
  s.homepage    = 'https://github.com/kpurdon/rumr'
  s.license     = 'GPLv3'
  s.executables << 'rumr'
  s.add_dependency('thor', ['~>0.19.1'])
end
```

**Golang (gomr)**

For golang the source is simply compiled into a binary that can be redistributed. There is no additional file needed and currently no package repository to push to.

## Conclusion

For this tool Golang feels like the wrong choice. I don't need it to be very performant and I'm not utilizing the native concurrency Golang has to offer. This leaves me with Ruby and Python. For about 80% of the logic my personal preference is a toss-up between the two. Here are the pieces I find better in one language:

### Command-Line Interface Declaration

Python is the winner here. The [click](http://click.pocoo.org/) libraries decorator style declaration is clean and simple. Keep in mind I have only tried the Ruby [thor](https://github.com/erikhuda/thor) gem so there may be better solutions in Ruby. This is also not a commentary on either language, rather that the CLI library I used in python is my preference.

### Recursive Directory Search

Ruby is the winner here. I found that this entire section of code was much cleaner and more readable using ruby's `Find.find()` and especially the `next unless` syntax.

### Packaging

Ruby is the winner here. The `rumr.gemspec` file is much simpler and the process of building and pushing a gem was much simpler as well. The [bundler](http://bundler.io/) tool also makes installing in semi-isolated environments a snap.

## Final Determination

Because of packaging and the recursive directory search preference I would choose Ruby as the tool for this application. However the differences in preference were so minor that Python would be more than fitting as well. Golang however, is not the correct tool here.
