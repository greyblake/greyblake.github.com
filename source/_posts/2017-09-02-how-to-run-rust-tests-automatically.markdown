---
layout: post
title: "How to run Rust tests automatically"
date: 2017-09-02 20:13
comments: true
categories: rust, tdd, test
---

If you prefer to use TDD approach to write Rust code like I do, you would need a fast feedback from your tests.

![TDD cycle](http://articles.tbscg.com/wp-content/uploads/2015/11/tdd-cycle-300x235.png)

Running `cargo test` every time you change a code base, quickly becomes a routine task.
But you can automate this using [cargo-watch](https://github.com/passcod/cargo-watch) plugin:

```
cargo watch -x test
```

Now `cargo test` will run automatically on every change in the source files.

However it can be still inconvenient, because you need to switch between your text editor and a terminal to see a result.

To overcome the problem we can use desktop notifications, so I've decided to create
another small cargo plugin [cargo-testify](https://github.com/greyblake/cargo-testify).

You can install it with the following command:

```
cargo install cargo-testify
```

And then run it within your rust project:

```
cargo testify
```

It detects changes in source files, run tests, and shows friendly desktop notifications, just like in the GIF below:

![cargo-testify](https://raw.githubusercontent.com/greyblake/cargo-testify/master/artifacts/demo.gif)
