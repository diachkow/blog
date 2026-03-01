---
title: "On Variable Type Hints in Python"
date: 2026-03-01
tags: ["python", "typing"]
description: "What are inline variable type hints and why I think they are cool"
---

During my recent projects I have developed a new coding habit: **leaving inline type hints
on variable assignments in Python**. It first started randomly and then developed into a
well-established practice that we follow now as a coding guideline rule with my colleagues,
especially in the AI era.

## Briefly on Type Hints

Teaching you about [type annotations in Python](https://docs.python.org/3/library/typing.html)
is not the goal of this blog post. However, to clarify my stance, I need to make a few
general points on type hints.

Starting from Python 3.5, you can leave type hints next to your function/method arguments,
their return values, and variable assignments:

```python
def add(a: int, b: int) -> int:          # type hints for functions
    return a + b

name: str = "John Doe"                   # inline variable type hints
age: int = 42

sum: int = add(3, 4)                     # ✅ valid function call according to the signature
concat: str = add(first_name, last_name) # ❌ invalid call, but evaluates at runtime
```

In my experience, the common practice is to leave type hints for *functions* and *class
methods*, but omit those on *variable assignments* and let the editor infer the type for you.
I want to talk specifically about the latter ones here in this article.

## When and Why To Use Variable Annotations

So, why might you want to use variable type hints in your codebase after all? Remember this line
from [Zen of Python](https://peps.python.org/pep-0020/#the-zen-of-python):

> Explicit is better than implicit.

Having your types written next to variable assignments gives you the entire picture of your code
at first sight, without any additional keystrokes or mouse movements needed. You might be a
Python professional and keep return types of all built-in methods of `str` in memory, but what
about other libraries and third-party code? Consider the following example:

```python
post = get_most_recent_blogpost()
```

Just by looking at it, can you tell what type the variable `post` has? Is it `str`, and you
can directly pass it to your rendering library? Is it a complex object, and to get the post
contents, you actually need to access some property on it? You never know.

Let's bring variable type hints here:

```python
post: MarkdownPost = get_most_recent_blogpost()
```

Of course, it's `MarkdownPost`, that `pydantic.BaseModel` subclass you saw recently in your
project's `models.py` file. It should have some property on it. `.content`, I guess. Your editor
will tell you anyway.

```python
post: MarkdownPost = get_most_recent_blogpost()
content = sanitize_text(post.content)
```

At this point, after typing `post.` on line 2, your LSP should start giving you autocompletion
suggestions with all the properties available for the `MarkdownPost` object. The point here is that
by just knowing that `get_most_recent_blogpost` is returning something more complex than just
`str`, you, as a developer, can expect it to be of a certain shape, and you can edit code more
efficiently.

But that's not all. Admittedly, if you wrote the line `post = get_most_recent_blogpost()` you will
know the return type anyway. The real benefits of leaving inline type hints come not when writing,
but when reading the code.

Let's repeat it once again: **variable type hints are useful when reading the code**. Not just in
your editor, but this is particularly useful when viewing your code **outside of your editor**.
GitHub doesn't show you the variable type on hover, so you can't get the type during a code
review or when browsing through project files.

> **Note**: I wrote this part before I started actively using AI agents to write code.
> Now that LLMs write and read code, inline type hints are even more relevant.
> With all types specified, AI agents don’t have to infer types or look up definitions
> elsewhere, reading and loading the whole project in its context window.

Of course, there are other minor benefits you get from this practice. Just from the top
of my head, those could be:

- You can assign a more general type than the actual function return type:
```python
items: Iterable[str] = "item1,item2".split(",")
```
- If you update a function's return type but haven't changed the function result assignments
accordingly, a static type checker can now detect this more efficiently.
- This makes relations and dependencies in your code clearer and even more obvious.

... and I am sure there are more than that. But for me, the main selling point and the sole reason
I am following this practice is the extremely improved readability and maintainability of the code.
It is dramatically simpler to perform a code review or make any changes to a codebase that has
inline variable type hints.

## When **NOT** To Use Variable Annotations

Of course, nothing in this world can be perfect. Adding inline type hints brings some visual noise
to your code, so it is very important to keep only the relevant ones. There is no point in adding
the most obvious type hints when a variable is assigned a value of an object that is
constructed in line, e.g.

```python
post = """
# Heading

... Content ...
"""
```

The type of the variable `post` here is obviously `str`. LSPs, human readers and AI agents can easily
infer this with no additional context required.

This is also true for non-builtin types:

```python
mailing = EmailService()
```

The type of `mailing` variable is clear from the code itself.

However, you may want to add type hints for mutable objects and "containers" to set the
expectations on what can be added to this data structure:

```python
m: dict[str, int] = {}
for letter in set(word):
    m[letter] = word.count(letter)
```

Needless to say, these type hints will be useful as long as you are using static type
checkers and enforce them in your project CI.

---

**Additional notes**:

Some LSPs support inlay hints ([example](https://www.jetbrains.com/help/idea/inlay-hints.html)).
While they provide the same editor experience, you cannot use them for code reviews (unless
you are using a special editor extension for in-editor code reviews). In addition, I personally
don't like editor features that modify the original source code contents, where what you see in
your editor is not what is actually written to the file.
