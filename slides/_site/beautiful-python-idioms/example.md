#Beautiful python Idioms

###Paris.py meetup - 2013/06/12



## Who am I?

- olivier hervieu
- ~~software~~ python developer at tinyclues
- my twitter (junk) account: `@_olivier_`


## Why this presentation?

- inspired by Raymond Hettinger's conference at Pycon
- inspired by Ned Batchelder's conference at Pycon
- 90 minutes of conferences in 10!



## Part 1: The good



### Looping over numbers

bad:

```python
for i in range(6):
    print i**2
```

even worse:

```python
for i in [0, 1, 2, 3, 4, 5]:
    print i**2
```

better:

```python
for i in xrange(6):
    print i**2
```



### Looping over collections

evil things:

```python
for i in xrange(len(colors)):
    print colors[i]
```

```python
for i in xrange(len(colors)-1, -1, -1):
    print colors[i]
```

```python
colors = ['blue', 'red', 'yellow', 'purple']
for i in xrange(len(colors)):
    print i, colors[i]
```


![Maximum FUUUU](/FU.jpg)


please consider (and use!) the following:

```python
for color in colors:
    print color
```

```python
for color in reversed(colors):
    print color
```

```python
colors = ['blue', 'red', 'yellow', 'purple']
for i, color in enumerate(colors):
    print i, color
```



### Looping over two (or more) collections

```python
firstnames = ['Alice', 'Bob', 'John']
lastnames = ['Dupont', 'Sleigh', 'Doe']
ages = [24, 63, 33]
```

dumb version:

```python
zipped_data = []
n = min(len(firstnames), len(lastnames), len(ages))
for i in range(n):
    zipped_data.append((firstnames[i], lastnames[i], ages[i]))

```

and what you really want to do:

```python
zip(firstnames, lastnames, ages)
```

the generator way:

```python
izip(fistnames, lastnames, ages)
```

see also: `izip_longest` in `itertools`



### Creating dictionaries from pairs

```python
firstnames = ['Alice', 'Bob', 'John']
ages = [24, 63, 33]

d = dict(izip(firstnames, ages))  # {'Alice': 24, 'Bob': 63, 'John': 33}
d = dict(enumerate(firstnames))  # {0: 'Alice', 1: 'Bob', 2: 'John'}
```



### Mastering List/Set/Dict Comprehension

```python
sum = 0
for i in xrange(10):
  sum += i ** 2
print sum
```

or

```python
pow_2 = []
for i in xrange(10):
  pow_2.append(i ** 2)
print sum(pow_2)
```

is equivalent to:

```python
print sum([i**2 for i in xrange(10)])
print sum(i**2 for i in xrange(10))
```


```python
colors = ['b', 'r', 'g', 'g', 'r', 'r' ,'r', None]
unique_colors = {color for color in colors if color}
```

```
pow_2_map = {k: k ** 2 for k in xrange(10000)}
```



### Reading files

the `<2.6` way:

```python
f = open('data.txt')
try:
    data = f.read()
finally:
f.close()
```

you should use:

```python
with open('data.txt') as f:
    data = f.read()
```



## Part 2: the beautiful



### Context manager

probably my favorite use case scenario: avoiding the `try/except/pass` pattern

```python
try:
    os.remove('somefile.tmp')
except OSError:
    pass
```

```python
from contextlib import contextmanager
@contextmanager
def ignored(*exceptions):
try:
    yield
except exceptions:
    pass
```

```python
with ignored(OSError):
    os.remove('somefile.tmp')
```



### `import infinity`

```python
itertools.count()
itertools.cycle()
```



### Powerful methods

common methods come with handful parameters:

```python
tall_buildings = {
  "Empire State": 381, "Sears Tower": 442,
  "Burj Khalifa": 828, "Taipei 101": 509,
  }
```

```python
print max(tall_buildings.values())
```

```python
print max(tall_buildings.items(), key=lambda b: b[1])  #('Burj Khalifa', 828)
```

```python
print max(tall_buildings, key=tall_buildings.get)  # 'Burj Khalifa'
```


```python
colors = ['red', 'green', 'blue', 'yellow']
for color in sorted(colors):
    print color
```

```python
for color in sorted(colors, reverse=True):
    print color
```

```python
def compare_length(c1, c2):
    if len(c1) < len(c2):
        return -1
    if len(c1) > len(c2):
        return 1
    return 0
print sorted(colors, cmp=compare_length)
print sorted(colors, key=len)
```



### For/Else Statements

use the `else` statement to detect if a `break` has been encountered:

```python
def find(seq, target):
    found = False
    for i, value in enumerate(seq):
        if value == tgt:
            found = True
            break
    if not found:
        return -1
    return i
```

```python
def find(seq, target):
    for i, value in enumerate(seq):
        if value == tgt:
            break
    else:
        return -1
return i
```



### Counting with Dictionnaries
####(or know your standard library)


naive implementation:

```python
colors = ['red', 'green', 'red', 'blue', 'green', 'red']
d = {}
for color in colors:
    if color not in d:
        d[color] = 0
    d[color] += 1
```


slightly better:

```python
d = {}
for color in colors:
    d[color] = d.get(color, 0) + 1
```


in limbo:

```python
from collections import defaultdict
d = defaultdict(int)
for color in colors:
    d[color] += 1
```


python guru:

```python
from collections import Counter
Counter(colors)
```



### Generators

```python
def evens(stream):
    for n in stream:
        if n % 2 == 0:
            yield n
```

```python
for n in evens(xrange(10000)):
    do_something(n)
```



### NamedTuple

```python
class BookOrder(object)
    def __init__(self, name, quantity):
        self.name = name
        self.quantity = quantity
```

```python
from collection import namedtuple
BookOrder = namedtuple('BookOrder', ['name', 'quantity'])
```



## Part 3: the Awesome



### Powerful iteration

####Continue a function until a sentinel value is reached

everybody write this:
```python
blocks = []
while True:
    block = f.read(32)
    if block == '':
        break
    blocks.append(block)
```

write this instead:
```python
blocks = []
for block in iter(partial(f.read, 32), ''):
    blocks.append(block)
```


#### Find elements

```python
next((i for i in xrange(1000) if i > 10 and i % 2), 0)
```



### Abstracting iteration

thanks to generators, simplify your code:

```python
f = open("my_config.ini")
for line in f:
    line = line.strip()
    if line.startswith('#'):
        # A comment line, skip it.
        continue
    if not line:
        # A blank line, skip it.
        continue

    # An interesting line.
    do_something(line)
```


```python
def interesting_lines(f):
    for line in f:
        line = line.strip()
        if line.startswith('#'):
            continue
        if not line:
            continue
        yield line
```

```python
with open("my_config.ini") as f:
    for line in interesting_lines(f):
        do_something(line)
```



### Using decorators

the web cache example:

```python
def web_lookup(url, saved={}):
    if url in saved:
        return saved[url]
    page = urllib.urlopen(url).read()
    saved[url] = page
    return page
```

powered re-usability with decorators!
```python
@cache
def web_lookup(url):
    return urllib.urlopen(url).read()
```

```python
def cache(func): saved = {}
    @wraps(func)
    def newfunc(*args):
        if args in saved:
            return newfunc(*args)
        result = func(*args)
        saved[args] = result
        return result
    return newfunc
```



## Part 4: Goodies and advice



### Explicit is better than implicit

```python
firsname, lastname, age = ["John", "Doe", 33]
```

is better than:

```python
whoami = ["John", "Doe", 33]
firstaname = whomai[0]
lastname = whoami[1]
age = whoami[2]
```

and:

```python
twitter_search('@obama', retweets=False, numtweets=20,
popular=True)
```

is better than:

```python
twitter_search('@obama', False, 20, True)
```



### Unlimited default dict

```python
infinite_defaultdict = lambda: defaultdict(infinite_defaultdict)
```



### Matrix transposition with zip

```python
a=[[1,2,3],[4,5,6],[7,8,9]]
tr_a = [list(i) for i in zip(*a)]
```



### Thank You!
