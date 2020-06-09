---
layout: post
title: Style Guide
categories: [Markdown,Code,HTML]
excerpt: This is a shortened article description rather than pulling from the top of the article.
---
# Pullquotes
In graphic design, a pull quote (also known as a lift-out pull quote) is a key phrase, quotation, or excerpt that has been pulled from an article and used as a page layout graphic element, serving to entice readers into the article or to highlight a key topic. {% include pullquote.html quote="It is typically placed in a larger or distinctive typeface and on the same page." %} Pull quotes are often used in magazine and newspaper articles, annual reports, and brochures, as well as on the web. They can add visual interest to text-heavy pages with few images or illustrations.

# Images
![](/images/ciphermenial.png)

# Code Snippets
This demonstrate the use of code snippets in the theme. The code snippets are powered by [Pygments](http://pygments.org/) and the code theme is called [Dracula](https://draculatheme.com/).


This is a raw snippet:

```
hello world
123
This is a text snippet
```

This is a PHP snippet:

```php
<?php
    echo 'Hello, World!';
?>
```

This is a JavaScript snippet:

```js
const add = (a, b) => a + b
const minus = (a, b) => a - b

console.log(add(100,200))  // 300
console.log(minus(100,200))  // -100
```

This is a Python snippet:

```python
def say_hello():
    print("hello world!")

say_hello()   // "hello world!"
```

# Standard Markdown
# Heading
## Heading Level 2
### Heading Level 3
#### Heading Level 4

> This is a quote.

- List item 1
- List item 2
    - Sublist item 1
- List item 3

1. Numbered list item 1
2. Numbered list item 2

|Table Column Title 1|Table Column Title 2|Table Column Title 3|Table Column Title 4|
|Item 1|Item 2|Item 3|Item 4|
