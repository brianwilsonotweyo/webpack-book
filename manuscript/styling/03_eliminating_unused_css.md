# Eliminating Unused CSS

Frameworks like [Bootstrap](https://getbootstrap.com/) or [Tailwind](https://tailwindcss.com/) tend to come with a lot of CSS. Often you use only a small part of it and if you aren't careful, you will bundle even the unused CSS. There's tooling that exists to work around the problem.

[PurgeCSS](https://www.npmjs.com/package/purgecss) is a tool that can achieve this by analyzing files. It walks through your code and figures out which CSS classes are being used. Often there is enough information for it to strip unused CSS from your project. It also works with single page applications to an extent.

[uncss](https://www.npmjs.com/package/uncss) is a good alternative to PurgeCSS. It operates through PhantomJS and performs its work differently. You can use uncss itself as a PostCSS plugin. [dropcss](https://www.npmjs.com/package/dropcss) is another option.

Given PurgeCSS works the best with webpack, we'll demonstrate the usage in this chapter.

{pagebreak}

## Setting up Tailwind

To make the demo more realistic, let's install Tailwind to the project.

```bash
npm add tailwindcss postcss-loader --develop
```

To load Tailwind, we'll have to use PostCSS:

**webpack.parts.js**

```javascript
exports.tailwind = () => ({
  loader: "postcss-loader",
  options: {
    postcssOptions: {
      plugins: [require("tailwindcss")()],
    },
  },
});
```

The new configuration still needs to be connected:

**webpack.config.js**

```javascript
leanpub-start-insert
const cssLoaders = [parts.tailwind()];
leanpub-end-insert

const commonConfig = merge([
  ...
leanpub-start-delete
  parts.extractCSS(),
leanpub-end-delete
leanpub-start-insert
  parts.extractCSS({ loaders: cssLoaders }),
leanpub-end-insert
]);
```

{pagebreak}

To make the project aware of Tailwind, `import` it from CSS:

**src/main.css**

```javascript
@tailwind base;
@tailwind components;

/* Write your utility classes here */

@tailwind utilities;

body {
  background: cornsilk;
}
```

## Using Tailwind classes

You should also make the demo component use Tailwind classes, so there is something to work with:

**src/component.js**

```javascript
export default (text = "Hello world") => {
  const element = document.createElement("div");

  element.className = "rounded bg-red-100 border max-w-md m-4 p-4";
  element.innerHTML = text;

  return element;
};
```

{pagebreak}

If you run the application (`npm start`), the "Hello world" should look like a button.

![Styled hello](images/styled-button.png)

Building the application (`npm run build`) should yield output:

```bash
⬡ webpack: Build Finished
⬡ webpack: asset main.css 1.99 MiB [emitted] [big] (name: main)
  asset index.html 237 bytes [compared for emit]
  asset main.js 193 bytes [emitted] [minimized] (name: main)
  Entrypoint main [big] 1.99 MiB = main.css 1.99 MiB main.js 193 bytes
  orphan modules 261 bytes [orphan] 2 modules
  code generated modules 309 bytes (javascript) 1.99 MiB (css/mini-extract) [code generated]
    ./src/index.js + 1 modules 309 bytes [built] [code generated]
    css ./node_modules/css-loader/dist/cjs.js!./node_modules/postcss-loader/dist/cjs.js??ruleSet[1].rules[0].use[2]!./src/main.css 1.99 MiB [code generated]

...
```

As you can see, the size of the CSS file grew, and this is something to fix with PurgeCSS.

{pagebreak}

## Enabling PurgeCSS

[purgecss-webpack-plugin](https://www.npmjs.com/package/purgecss-webpack-plugin) allows you to eliminate most of the CSS as ideally we would bundle only the CSS classes we are using.

```bash
npm add glob purgecss-webpack-plugin --develop
```

You also need to configure as below:

**webpack.parts.js**

```javascript
const path = require("path");
const glob = require("glob");
const PurgeCSSPlugin = require("purgecss-webpack-plugin");

const ALL_FILES = glob.sync(path.join(__dirname, "src/*.js"));

exports.eliminateUnusedCSS = () => ({
  plugins: [
    new PurgeCSSPlugin({
      paths: ALL_FILES, // Consider extracting as a parameter
      extractors: [
        {
          extractor: (content) =>
            content.match(/[^<>"'`\s]*[^<>"'`\s:]/g) || [],
          extensions: ["html"],
        },
      ],
    }),
  ],
});
```

T> For exceptions, [PurgeCSS 3.0](https://github.com/FullHuman/purgecss/releases/tag/v3.0.0) includes **safelist** and **blocklist** options.

{pagebreak}

Next, the part has to be connected with the configuration. It's essential the plugin is used _after_ the `MiniCssExtractPlugin`; otherwise, it doesn't work:

**webpack.config.js**

```javascript
const productionConfig = merge([
  parts.extractCSS({ loaders: cssLoaders }),
leanpub-start-insert
  parts.eliminateUnusedCSS(),
leanpub-end-insert
]);
```

The order of the CSS related calls doesn't matter as the plugins will register to different parts of the build.

If you execute `npm run build` now, you should see something:

```bash
⬡ webpack: Build Finished
⬡ webpack: asset main.css 7.68 KiB [emitted] (name: main)
  asset index.html 237 bytes [compared for emit]
  asset main.js 193 bytes [compared for emit] [minimized] (name: main)
  Entrypoint main 7.87 KiB = main.css 7.68 KiB main.js 193 bytes
  orphan modules 261 bytes [orphan] 2 modules
  code generated modules 309 bytes (javascript) 1.99 MiB (css/mini-extract) [code generated]
    ./src/index.js + 1 modules 309 bytes [built] [code generated]
    css ./node_modules/css-loader/dist/cjs.js!./node_modules/postcss-loader/dist/cjs.js??ruleSet[1].rules[0].use[2]!./src/main.css 1.99 MiB [code generated]
  webpack 5.1.3 compiled successfully in 2429 ms

```

The size of the style has decreased noticeably. Instead of 1.99 MiB, we have roughly 7 KiB now.

W> Tailwind includes PurgeCSS out of the box and it can be preferable to use that. See [Tailwind documentation](https://tailwindcss.com/docs/controlling-file-size/#removing-unused-css) for more information. The example above is enough to illustrate the idea, and it works universally.

{pagebreak}

### Critical path rendering

The idea of [critical path rendering](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/) takes a look at CSS performance from a different angle. Instead of optimizing for size, it optimizes for render order and emphasizes **above-the-fold** CSS. The result is achieved by rendering the page and then figuring out which rules are required to obtain the shown result.

[critical-path-css-tools](https://github.com/addyosmani/critical-path-css-tools) by Addy Osmani lists tools related to the approach.

## Conclusion

Using PurgeCSS can lead to a significant decrease in file size. It's mainly valuable for static sites that rely on a massive CSS framework. The more dynamic a site or an application becomes, the harder it becomes to analyze reliably.

To recap:

- Eliminating unused CSS is possible using PurgeCSS. It performs static analysis against the source.
- The functionality can be enabled through **purgecss-webpack-plugin**.
- At best, PurgeCSS can eliminate most, if not all, unused CSS rules.
- Critical path rendering is another CSS technique that emphasizes rendering the above-the-fold CSS first. The idea is to render something as fast as possible instead of waiting for all CSS to load.

In the next chapter, you'll learn to **autoprefix**. Enabling the feature makes it more convenient to develop complicated CSS setups that work with older browsers as well.
