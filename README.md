I've had several conversations about Python and why I think it's a bad programming language for real software development.
A lot of people are surprised when I say this.

I assert Python persists for a few main reasons:
* It is easy to learn.
* It has a few cool ideas that spark people's interest.
* In math-related and especially machine learning fields, it has acquired a critical mass of libraries and users.
* Writing small programs in Python can be very quick.

Like many others, I learned Python as my first programming language.
For years, it was also my strongest.
I still use Python frequently for machine learning.
Usually training and evaluating a model requires only a modest amount of code, and the trained model can be hooked up into a system that uses only minimal Python, so that's fine by me.
I also use Python for short explorations, and have no quarrel about that.

However, I've found that for larger projects, it quickly becomes untenable.
A number of people have told me they still like it in production systems, but I'm inclined to believe they aren't fully aware of what they're missing out on.

I'll be comparing Python to JavaScript.
Like Python, JavaScript is another common, easy-to-learn, scripting, dynamically-typed language.
Although it definitely also has flaws, it is worlds better for real projects.

Sometimes I'll also mention Scala, my current favorite language for real projects, which is a moderately common, moderately difficulty to learn, compiled, statically-typed language.
It is similar to Java in many ways, runs on the JVM, but is much more concise and has much more powerful features, especially concerning types.

Below are my reasons why you should never use Python for big projects.

<h3>1. Python is verbose in ways that encourage bad practices.</h3>

Let's say you have a function that returns a few pieces of data `a`, `b`, `c`. In JavaScript, one would usually write
```
function fn(...args) {
  ...
  return {a, b, c};
}

const {a, b, c} = fn(myArgs);
```

In Python, one way to write the same code looks like
```
def fn(*args):
  ...
  return {'a': a, 'b': b, 'c': c}

result = fn(myArgs)
a = result['a']
b = result['b']
c = result['c']
```

That's rather tedious (and another similarly tedious way would be to use `dict(a=a, b=b, c=c)`)! If you have the luxury of being on Python 3.7, you could use a data class instead:
```
from dataclasses import dataclass
from typing import Any

@dataclass
class ResultType:
  a: Any
  b: Any
  c: Any

def fn(*args):
  ...
  return ResultType(a, b, c)

result = fn(myArgs)
#you can now access result.a, result.b, result.c without hash lookups
```

That's a sorta reasonable option, and the best Python provides.
The overhead of defining `ResultType` is probably worth it for the extra clarity and reduced verbosity each time you define/use `fn` (but again, only available if you're on Python 3.7+).
Another solution that's almost as good is to use a `namedtuple`, but those need to do a hash lookup to access elements by name, so you either have to manually unpack them like in the `dict` example or take the performance hit.

It does irk me that a scripting language is making me do even more syntax sugar than a good compiled language with the `@dataclass` and `import`s, but this solution is feasible.
Even Python's best solution here is substantially more work than in JavaScript or Scala.

However, what people often end up doing is this sirenic solution:
```
def fn(*args):
  ...
  return (a, b, c)

a, b, c = fn(myArgs)
```
So concise!
So effortless!
But *it's a trap*.

Now, if we ever want to change `fn` to output `(a, b, d, c)`, every piece of code that calls `fn` will need to be updated.
In other words, what should be an `O(1)` coding time change costs `O(n)` coding time when using this pattern.

Also, clarity is forfeit.
The only way to know the order of results `fn` returns is to check its definition; you can no longer ask for `result.a`.

If your Python software project exhibits this pattern, I offer my sympathy.
Python got you hooked by giving you that first use for (what seemed to be) free.

<h3>2. Python environments are hell.</h3>

I'm not the first one to [remark on this](https://xkcd.com/1987/).
Working with pip, Conda, and global and virtual versions of Python is such a nightmare that I don't want to get into it here.
I've had far more trouble with package and version management in Python than any other language.

In Python, elaborate tools like Conda are necessities.
Some packages simply have contradictory version requirements, making them impossible to install together.
For me, not too long ago, that was `tensorflow` and `jupyter`.
The most practical way to dev around this is with Conda, which uses virtual environments.
I now have one environment for `tensorflow` and one for `jupyter`.
It's just an utter drain to keep all your packages happy and to switch between environments.

Plus, your system will look very complicated for those cases when you really need to debug a package issue.
It will be a nightmare of pipes, paths, and directories.
That XKCD comic I linked shows it best.

Better languages avoid these issues by mostly relying on project-level dependencies, rather than globally-installed packages.
In my view, packages should be cached locally for projects to access with specific version requirements, rather than forcing all projects in an environment to share the same global packages.
NPM for JavaScript does alright by mostly sticking to per-project dependencies (but it unfortunately it lacks a global cache).
Build tools and package managers for just about any other language seem to have figured out what Python has not.

<h3>3. Python built-in functions are lame.</h3>

<h4>3a. They steal the good variable names.</h4>

I can't use `file` or `dir` or `len` or `id` or `input` for my variables names without overriding some built-in function?
Those are often the names I want to give to variables!
As a result, I often see (and, shamefully, use) lousy variable names like `f` for files.

<h4>3b. They cause confusion and ugly code.</h4>

Many of the built-in Python functions just defer to the particular class's implementation.
For instance, `len` just calls the `__len__` method.

First, I would argue that this is unclear, because `len` appears to be a function that computes the length of a general data structure.
This can mislead beginners into thinking that `len` has the same `O(1)` runtime for any data structure.
It also wrongly suggests that `len` has some general logic for computing length.

Second, as a result of these domineering built-in functions, people sometimes give their classes implementations even when they don't really make sense.
What is the `len` of a 2D numpy array - its first dimension, or its total number of elements?

Finally, I find the syntax for these global methods much uglier than the more natural function chains that would arise from simply using the data structure's methods directly.
For instance, here is a simple chain in JavaScript:
```
const arr = [-4, 1, 2, 3];
const result = arr
  .filter((x) => x % 2 === 0)
  .map((x) => x * x)
  .sort()
```
It's concise, and it reads nicely.
The operators follow the same order you want to do them in.
Here is how you would have to do it with Python built-ins:
```
arr = [-4, 1, 2, 3]
result = sorted(
  map(
    lambda x: x * x,
    filter(
      lambda x: x % 2 == 0,
      arr
    )
  )
)
```
Not very pretty, eh?
Lots of nested parentheses, and the operators come in the opposite order you want to do them in.
You have to think backwards to write or read this code.
Plus, I always forget whether Python puts the function or data structure first in `filter` and `map`, an issue that doesn't arise with function chains.

Of course, this example could also be done with a list comprehension:
```
arr = [-4, 1, 2, 3]
result = [x * x for x in arr if x % 2 == 0].sort()
```
This is a nice solution, but it only strengthens the point that Python built-in functions are useless.

<h3>4. Python standard libraries are lame and excessive.</h3>

The Python standard library (what you can get to via `import x` without doing any installation of `x`) isn't very well thought out, and the composition of packages within it feels like a mess.
A tiny sample of its nuisances:
* The `json` library writes non-standard JSON by default, including `NaN`'s and `Infinity`'s.
* There are at least 4 libraries that measure time and differences between them: `datetime`, `time`, `calendar`, and `timeit`.
* Standard libraries exist for utterly niche and sometimes obsolete use cases, like `macpath`, a tool for working with file paths in MacOS 9 and earlier.

To see how bloated the Python standard library is, just compare its [list of 246 libraries](https://docs.python.org/3/library/index.html) with NodeJS's [list of 48](https://nodejs.org/api/).
I think for most use cases, they cover about the same ground, and NodeJS's do it a bit more consistently and elegantly.

<h3>5. Python concurrency is hell.</h3>

Any significant project or backend eventually needs concurrency, whether to take full advantage of your CPU's, execute long-running network calls without hanging, or just let higher-priority threads do their work.
But Python is famously bad at this.

Due to an implementation detail in Python called the Global Interpreter Lock (GIL), each Python process runs only one command at a time.
This means your code can still be efficient if it's just waiting for a network call or external process, but that it can't take full advantage of CPU resources.

Also, the GIL [probably isn't going away](https://wiki.python.org/moin/GlobalInterpreterLock).

Sure, you can use the Python `multiprocessing` library to run a lot of Python processes until you have full CPU usage, but Python has to pickle and unpickle all the data each process needs.
If each process has low CPU usage, or the amount of data they share is large, this will be horribly inefficient.

This isn't a problem in Scala, any JavaScript implementations I'm aware of, or pretty much any other common language.

<h3>6. Python imports are hell.</h3>

<h4>6a. Getting them to work is hideous.</h4>

Oh, boy.
Let's say we have a simple, plausible project structure:
```
src/
  main
  utils
  handlers/
    my_handler
```

In JavaScript, imports are relative, and might look like this:
```
>utils.js
const handler = require('./handlers/my_handler').handler;

>my_handler.js
const utilFunc = require('../utils').func;
```

In another language, like Scala, imports are absolute and might look like this:
```
>utils.java
import my.proj.handlers.my_handler.handler;

>my_handler.java
import my.proj.utils.func;
```

But in Python, these imports are IMPOSSIBLE.
Yes, impossible.
You literally can't do these simple imports.
lol.

First of all, for some quirky reason, you can't do circular `from my_file import my_thing` imports in Python.
This just seems inexcusable to me.

Second, even once you give up on `from ... import`s and just import the whole files circularly, you'll still need even more ridiculous setup.
At the end of the day, your project will look like this
```
src/
  __init__.py
  main.py
  utils.py
  handlers/
    __init__.py
    my_handler.py
```
AND you'll need to either mess with your `~/.bashrc` or equivalent like so:
```
export $PYTHONPATH=$PYTHONPATH:/path/to/my/project/src/
```
or add a `setup.py` file that you need to remember to run before developing on the project.

AND even then, you might need to run your code with `python(3) -m main`.

Here's what your imports will look like:
```
>utils.py
from handlers import my_handler

>my_handler.py
import utils
```

Wait... `import utils`?
IMPORT UTILS?
How does that make any sense?
What if I have a global package called `utils`?
What if I have another file in the same directory called `utils`?
The syntax does not distinguish; Python imports hide where `utils` is really coming from.
Python does have a relative import pattern that might be clearer in some cases, but Python discourages its usage because it has yet more quirks and limitations.

I haven't even gotten into `sys.path` and the many unnecessarily complicated mechanisms used in Python to get stuff imported.
Far from the dream of `import my_module` just working, Python has managed to make the internals and syntax of imports far more abstruse than any other language I've seen.
I recommend [this guide](https://chrisyeh96.github.io/2017/08/08/definitive-guide-python-imports.html) for trying to sort out some of your Python import woes.

<h4>6b. They have weird behavior.</h4>

I've observed some other annoying oddities about Python imports.
For instance, let's say you have two files:
```
>temp1.py
import temp2
print('here')

>temp2.py
import temp1
print('there')
```

What would you expect to see when you run `python(3) temp1.py`?
Probably `there` then `here`, or perhaps `here` then `there`?

Nope.
Python double-runs the running file, printing
```
here
there
here
```

So be careful putting any global code in your files - it might be run twice, depending on the entry point.
No sane language does this.

<h2>Is Python zen?</h2>

I couldn't finish this post without tying it into Python's guiding philosophy, [Zen of Python](https://en.wikipedia.org/wiki/Zen_of_Python), obtainable by `python(3) -c "import this"`.
For the most part, these are hard to argue with, but how people choose to interpret them varies wildly.

The only one of these platitudes that actually bothers me is, "If the implementation is hard to explain, it's a bad idea."
In practice, hard problems always come up, and code will become hard to explain at some point.
Hiding from the problem, or dumbing down any challenge that comes your way into the most easily-explainable solution, will only result in slow code that is hard to maintain.

But my main take on Zen of Python is that Python doesn't really meet these platitudes.
* "Beautiful is better than ugly." Then why is every Python codebase littered with `default_x if x is None else x`?
* "Readability counts." Then why must functions like `map`, `filter`, `sorted` come before the thing they're applied to rather than after?
* "Although practicality beats purity." Then why is concurrency so horrible?
* "Special cases aren't special enough to break the rules." Then why are imports to full of special cases?
* "There should be one—and preferably only one—obvious way to do it." But why is the obvious way sometimes so terrible for developer time, like returning tuples rather than objects/structs? And why are there 4 different standard libraries for time?

The gestalt I get is that Python tries to coddle us, hiding complexities that necessarily exist.
Whenever you need to do the lower level/higher performance/more general/better coding practice thing that Python did not make simple, the facade breaks down, and code that should be clean becomes awkward or monstrous.
We can have readable, easily-maintained, high-level programs without this nonsense.
We just can't do it with Python.
