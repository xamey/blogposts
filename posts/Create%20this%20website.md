Hey! Welcome to this first blog post.

This has been a test to try out Astro.build and see how it works, and in the end, it will end up being my personal website. Looks neat, right?

Let's take you on my journey to create this website.

## Bolt.new

I've discovered about [bolt.new](https://bolt.new/) a few weeks ago, and I've been quite amazed by the what you can achieve with it.
From a simple prompt, you can generate a fully functional Javascript-based codebase, with the entire codebase scaffolded out for you.

In order to make sure it works at the first try (it can be buggy at times), I've decided to use React. I wanted to try out Astro so badly, as it's quite new I was afraid it might not work, but code ran at first try.

<img src="https://github.com/xamey/blogposts/blob/main/imgs/bolt2.png?raw=true" alt="bolt.new" width="500" height="auto"/>

_This is the landing page. You can even chose an pre-existing stack._

I'll let you the pleasure to discover it by yourself.

Anyway, it gives you a good starting point, and you can always tweak it from there.

## Iterate with Cursor.sh

If you don't know me yet, I'm a big fan of AI-assisted coding.

I've used Cursor.sh for a few months now, and I'm amazed by the quality of the autocomplete, as well as the ability to have conversations with the codebase.

I've used it to iterate on the design of this website, which I'm pretty happy with.

<img src="https://github.com/xamey/blogposts/blob/main/imgs/composer-cursor.png?raw=true" alt="cursor.sh" width="500" height="auto"/>

_This is the Composer feature UI. A small window at the bottom of your IDE._

It has been a first try for Composer feature, and this is particularly useful for refactoring between different parts of the codebase. It hasn't been a real game changer for such a simple project, but I'm sure it will be more useful for more complex ones.

You can have a look at Cursor.sh features [right there](https://www.cursor.com/features).

## Blog posts hosting

I wanted that website to be as easy to manage as possible, so I used a little trick to upload my blog posts without relying on a CMS.

Any blog post is a simple markdown file, hosted on this repository: [xamey/blogposts](https://github.com/xamey/blogposts). Even the one you're currently reading!

When I push to this repository, it automatically updates the blog posts on my website. Well, your client retrieves the blog posts from this repository when you visit the website.

That way, if I want to write a new blog post, I just have to write it in a markdown file and push it to the repository.

The title is the name of the markdown file, URL encoded so I can add spaces and special characters.
And for the content, Markdown is rendered with [ReactMarkdown](https://www.npmjs.com/package/react-markdown).
