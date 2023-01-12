+++
title = "Managing local common lisp projects :common-lisp:@micro:"
date = 2023-10-12
draft = false
+++

When I started using lisp, I came from the world of modern dependency managers (npm, pip, maven, and the like). So I started searching for an equivalent in lisp. I quickly came across `quicklisp`. To my surprise, however, `quicklisp` worked differently than these packages. It works more like a dependency cache than true dependency manager, but that is a topic for another article.

The biggest struggle that `quicklisp` brought was managing local packages. `quicklisp` had no way to reference a dependency at a certain path, and couldn't automatically import the project I was in. This is all for good reason since ASDF (the common lisp build system) allows a lot of freedom.

I went looking for a better way to manage my local projects. I wanted a way to load local projects in a similar way to external dependencies while keeping things organized.

It turns out that `quicklisp` will check multiple locations for local project before it tries to pull down a dependency. This means that as long as all local projects exist in one of these locations, they get treated no differently than external dependencies.

One location that particularly appealed to me was the `\~/common-lisp` folder. This became where I store actual applications or scripts that will never become `quicklisp` libraries.

If I want to modify a library or start my own, on the other hand, I prefer to use one of the directories in `ql:*local-project-directories*`. This variable holds a list of directories for local projects, and can be expanded in any way needed. I, however, stick with the default `\~/.quicklisp/local-projects/"`.

Keeping my projects split up in this manner allows for easy separation of concerns. When I work on a library, I can take the extra care needed to make sure I document things and keep the public api slim.

When I start working on an application, especially a personal tool, these become less of a concern and I can just focus on getting it working.

This simple scheme has helped me keep things organized while loading everything in the same manner with `ql:quickload`.
