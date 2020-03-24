Blogging @Westermo R&D
======================


1. Create a blog post in kebab case: `_posts/YYYY-MM-DD-my-neat-title.md`
2. `jekyll s`
3. Write some more
4. `git commit -sam "New post ..."`
5. `git push`

> **NOTE:** Talk to the R&D Architect Group before publishing a blog
> post.  Use the `_drafts/` directory if you like, no branching.

When working with drafts you need some more arguments to jekyll:

    jekyll serve --watch --drafts


About Gemfile.lock
------------------

Since we are more than one person blogging using Jekyll, we're trying to
keep the `Gemfile.lock` file out of version control, because our setups
will likely differ.  Hence, to start working, first run:

```sh
bundle update
```

which creates a the `Gemfile.lock` for you.
