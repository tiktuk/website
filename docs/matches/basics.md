---
sidebar_position: 0
title: Basics
---

Matches are one of the Espanso's core concepts and define the replacements that will take place.

## Static Matches

In their most basic form, **Matches are pairs that associate a *trigger* with a *replacement text***.

For example, we can define a match that will expand every occurrence of `hello` with `world` while we are typing. Using the [YAML](https://en.wikipedia.org/wiki/YAML) syntax, it can be expressed as:

```yml
- trigger: "hello"
  replace: "world"
```

These kind of expansions are simple text replacements and can be considered *static*.

### Multi-line expansions

To replace the original text with a multi-line expansion, we can either use the `\n` line terminator character, such as:

```yml
- trigger: "hello"
  replace: "line1\nline2"
```
> Notice that when using `\n` as the line terminator character, quotes are needed.

Or values can span multiple lines using  `|` or `>`. Spanning multiple lines using a *Literal Block Scalar* `|` will include the newlines and any trailing spaces. Using a *Folded Block Scalar* `>` will fold newlines to spaces; it’s used to make what would otherwise be a very long line easier to read and edit. In either case the indentation will be ignored. Examples are:


```yml
- trigger: "include newlines"
  replace: |
            exactly as you see
            will appear these three
            lines of poetry
```
```yml
- trigger: "fold newlines"
  replace: >
            this is really a
            single line of text
            despite appearances
```

> As you can see, no quotes are needed in this case.


There are a number of characters that are special (or reserved) and cannot be used as the first character of an unquoted scalar: ``' "  [] {} > | * & ! % # ` @ ``

:::tip

If you want more information about the YAML syntax for multiline strings, please check out this website:
https://yaml-multiline.info/

:::

## Dynamic Matches

Static matches are suitable for many tasks, but can be problematic when we need an **expansion that changes dynamically**. For these situations, Espanso introduces the concepts of **variables** and **extensions**.

**Variables** can be used in the **replace** clause of a Match to include the *output* of a dynamic component, the **extension**. To make things more clear, let's see an example:

We want to create a match that everytime we type `:now` gets expanded to include the current time, like:

```
It's 11:29
```

Let's add the following match to your configuration, such as the `match/base.yml` file


```yaml title=$CONFIG/match/base.yml
  - trigger: ":now"
    replace: "It's {{mytime}}"
    vars:
      - name: mytime
        type: date
        params:
          format: "%H:%M"
```

After a while, Espanso should pick up the new configuration.

At this point, everytime we type `:now`, we should see something like `It's 09:33` appear!

Let's analyze the match step by step:

```yml
  - trigger: ":now"
```

In the first line we declare the trigger `:now`, that must be typed by the user to expand the match.


```yml
    replace: "It's {{mytime}}"
```


In the second line, we declare the *replace text* as usual, but this time we include the `mytime` **variable**,
that will contain the output of the **extension** used below.


```yml
    vars:
      - name: mytime
        type: date
```


In the following lines, we defined the `mytime` variable as type **date**. The type of a variable defines
the **extension** that will be executed to calculate its value. 
In this case, we use the [Date Extension](../extensions/#date-extension).

```yml
        params:
          format: "%H:%M"
```

In the remaining lines we declared the **parameters** used by the extension, in this case the *date format*.

## Global Variables

*Global variables* are variables that can be used across multiple matches. 
You can define them above your matches, and they will be available across all 
matches defined in that file and it's children.

For example, if you add the following into your `match/base.yml` file:

```yaml
global_vars:
  - name: "global1"
    type: "shell"
    params:
      cmd: "echo global var"
  - name: "greet"
    type: "echo"
    params:
      echo: "Hey"
```

You can then use `global1` and `greet` in the following matches:


```yaml
  - trigger: ":hello"
    replace: "{{greet}} Jon"
```


Typing `:hello` will result in `Hey Jon` to be expanded.

## Word Triggers

If you ever considered using Espanso as an **autocorrection tool for typos**, you may have experienced
this problem:

Let's say you occasionally type `ther` instead of `there`. Before the introduction of *word triggers*, 
you could have used espanso like this:

```yaml
  - trigger: "ther"
    replace: "there"
```

This would correctly replace `ther` with `there`, but it would also expand
`other` into `othere`, making it unusable.

With *word triggers* you can now add the `word: true` property to a match, telling espanso
to only trigger that match if surrounded by *word separators* ( such as *spaces*, *commas* and *newlines*). 
So in this case it becomes:

```yaml
  - trigger: "ther"
    replace: "there"
    word: true
```

At this point, espanso will only expand `ther` into `there` when used as a standalone word.
For instance:

Before | After |
--- | ---
Is ther anyone else? | Is there anyone else? | `ther` is converted to `there`
I have other interests | I have other interests | `other` is left unchanged

## Case propagation

Espanso also supports *case-propagation*, which makes it possible to expand a match
preserving the trigger casing.

For example, imagine you want to speedup writing the word `although`. You can define a word match as:

```yml
  - trigger: "alh"
    replace: "although"
    word: true
```

As of now, this trigger will only be able to be expanded to the **lowercase** `although`. If we now add `propagate_case: true` to the match:

```yml
  - trigger: "alh"
    replace: "although"
    propagate_case: true
    word: true
```

we are now able to propagate the case style to the match:
* If you write `alh`, the match will be expanded to `although`. 
* If you write `Alh`, the match will be expanded to `Although`.
* If you write `ALH`, the match will be expanded to `ALTHOUGH`.

:::tip Multi-word capitalization

When using multi-word replacements, the default behavior is to only capitalize the first word.
For example, the following match:

```yaml
  - trigger: ";ols"
    replace: "ordinary least squares"
    propagate_case: true
```

gets expanded to `Ordinary least squares` when typing `;Ols`.

If you want to **capitalize each word**, you can use the `uppercase_style: capitalize_words` option:

```yaml
  - trigger: ";ols"
    replace: "ordinary least squares"
    uppercase_style: "capitalize_words"
    propagate_case: true
```

In this case, typing `;Ols` gets expanded to `Ordinary Least Squares`.

:::


## Cursor Hints

Let's say you want to use espanso to expand some HTML code snippets, such as:

```yaml
  - trigger: ":div"
    replace: "<div></div>"
```

With this match, any time you type `:div` you get the `<div></div>` expansion, with the cursor at the end.

While being useful, this snippet would have been much more convenient if **the cursor was positioned
between the tags**, such as `<div>|</div>`.

To solve this problem, Espanso supports _cursor hints_, a way to control the position of the cursor
after the expansion. 

Using them is very simple, just insert `$|$` where you want the cursor to be positioned, in this case:

```yaml
  - trigger: ":div"
    replace: "<div>$|$</div>"
```

If you now type `:div`, you get the `<div></div>` expansion, with the cursor between the tags!

:::caution Things to keep in mind

* You can only define **one cursor hint** per match. Multiple hints will be ignored. If you need multiple hints, a decent replacement would be to use [Forms](#forms).
* This feature should be used with care in **multiline** expansions, as it may yield
  unexpected results when using it in code editors that support **auto indenting**. 
  This is due to the way the feature is implemented: espanso simulates a series of `left arrow`
  key-presses to position the cursor in the correct position. This works perfectly in single line
  replacements or in non-autoindenting fields, but can be problematic in code editors, as they
  automatically insert indentations that modify the number of required presses in a way
  espanso is not capable to detect.

:::

## Multiple triggers

Sometimes it's useful to expand a snippet using various aliases.
Because of this, Espanso supports *multi-trigger* matches, which allows the user to specify multiple triggers to expand the same match.
To use the feature, simply specify a list of triggers in the `triggers` field (instead of `trigger`):

```yml
- triggers: ["hello", "hi"]
  replace: "world"
```

Now typing either `hello` or `hi` will be expanded to `world`.

## Image Matches

Espanso also supports **expanding matches into images**. 
This can be useful in many situations, such as when writing emails or chatting.

![Image match example](/img/docs/imagematches.gif)

The syntax is pretty similar to the standard one, except you have to specify `image_path`
instead of `replace`. This will be the path to your image.

```yaml
  - trigger: ":cat"
    image_path: "/path/to/image.png"
```

:::caution Format remarks

On Windows and macOS, the most commonly used formats (such as PNG, JPEG and GIF) should work as expected.
On Linux, **you should generally use PNG** as it's the most compatible.

:::

### Path convention

While you can use any valid path in the `image_path` field, there are times in which it proves limited.
For example, if you are synchronizing your configuration across different machines, you could have problems
creating the same path on each of them.

In those cases, the best solution is to create a folder into the espanso configuration directory and put all
your images there.
At this point, you can use the `$CONFIG` variable in `image_path` to avoid hard-coding the path. For example:

Create the `images` folder inside the espanso configuration directory (the one which contains the `match` and `config`
directories),
and store all your images there. Let's say I stored the `cat.png` image. We can now create a Match with:

```yaml
  - trigger: ":cat"
    image_path: "$CONFIG/images/cat.png"
```

## Nested Matches

*Nested matches* makes it possible to include the output of a match inside another one.


```yaml
  - trigger: ":one"
    replace: "nested"

  - trigger: ":nested"
    replace: "This is a {{output}} match"
    vars:
      - name: output
        type: match
        params:
          trigger: ":one"
```


At this point, if you type `:nested` you'll see `This is a nested match appear`.

## Forms

Espanso is capable of creating arbirarly complex input forms.

![Espanso Form](/img/docs/macform.png)

These open up a world of possibilities, allowing the user to create matches with many arguments, as well as injecting those values into custom Scripts or Shell commands.

For more informations, visit the [Forms section](../forms).