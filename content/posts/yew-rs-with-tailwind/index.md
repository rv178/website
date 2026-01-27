---
title: "Yew.rs With TailwindCSS"
date: 2022-08-12T18:57:42+05:30
draft: false
tags: ["Rust", "WASM"]

cover:
    image: "cover.png"
    alt: "Description of image"
    relative: true
---

---

## Yew

Like a lot of people, I think [WASM](https://webassembly.org/) is the future. [Yew](https://yew.rs) is
an amazing frontend framework that helps in building web apps using WASM.

It's incredibly easy to use and the performance of the app is EXTREMELY fast! I'm currently re-writing
[Snowstry](https://github.com/snowstry/snowstry) with yew.rs for the frontend (switching from NextJS).
Here's the repo [link](https://github.com/snowstry/rewrite).

## Yew + Tailwind

Setting up yew with tailwind was very easy as well. If you don't know what tailwind is, here's the
[link](https://tailwindcss.com) to their website.

First things first, we need the tailwindcss binary for compiling the output css file.

I prefer using yarn over npm, but it's upto you.

```bash
yarn global add tailwindcss
```

Now let's setup a tailwind config file:

```js
module.exports = {
	content: ["./src/**/*.rs", "./index.html", "./public/css/*.css"],
	theme: {},
	plugins: [],
};
```

Directory structure:

```diff
	├── Cargo.toml
+	├── index.html (index.html in root)
	├── public
	│   ├── css
+	│   │   ├── main.css (global css file)
+	│   │   └── out.css (output css)
	├── src
	│   ├── components
	│   │   ├── mod.rs
	│   │   ├── nav.rs
	│   ├── main.rs
	│   └── pages
	│       ├── home.rs
	│       ├── mod.rs
	│       └── notfound.rs
+	├── tailwind.config.js (tailwind config)
	└── Trunk.toml
```

Now in our Trunk.toml, we setup a hook that executes before build, so output css is compiled
automatically.

```toml
[[hooks]]
stage = "pre_build"
command = "tailwindcss"
command_arguments = ["-c", "./tailwind.config.js", "-o", "./public/css/out.css", "--minify"]
```

Now in your `index.html` (at the root of your project), link the output css file.

```html
<link data-trunk rel="css" href="public/css/out.css" />
```

Run `trunk serve` and you're done! That's it for this blog, see you soon.
