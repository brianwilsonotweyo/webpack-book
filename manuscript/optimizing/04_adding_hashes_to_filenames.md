# Adding Hashes to Filenames

Even though the generated build works, the file names it uses is problematic. It doesn't allow to leverage client level cache efficiently as there's no way tell whether or not a file has changed. Cache invalidation can be achieved by including a hash to the filenames.

T> Starting from version 5, webpack is using a deterministic way of generating filenames that's a good compromise between bundle size and long time caching. The behavior is controllable through `optimization.moduleIds` and `optimization.chunkIds`. Latter applies to _Code Splitting_.

## Placeholders

Webpack provides **placeholders** for this purpose. These strings are used to attach specific information to webpack output. The most valuable ones are:

- `[id]` - Returns the chunk id.
- `[path]` - Returns the file path.
- `[name]` - Returns the file name.
- `[ext]` - Returns the extension. `[ext]` works for most available fields.
- `[fullhash]` - Returns the build hash. If any portion of the build changes, this changes as well.
- `[chunkhash]` - Returns an entry chunk-specific hash. Each `entry` defined in the configuration receives a hash of its own. If any portion of the entry changes, the hash will change as well. `[chunkhash]` is more granular than `[fullhash]` by definition.
- `[contenthash]` - Returns a hash generated based on content. It's the new default in production mode starting from webpack 5.

It's preferable to use particularly `hash` and `contenthash` only for production purposes as hashing doesn't do much good during development.

T> It's possible to slice `hash` and `contenthash` using specific syntax: `[contenthash:4]`. Instead of a hash like `8c4cbfdb91ff93f3f3c5` this would yield `8c4c`.

T> There are more options available, and you can even modify the hashing and digest type as discussed at [loader-utils](https://www.npmjs.com/package/loader-utils#interpolatename) documentation.

W> If you are using webpack 4, be careful with `contenthash` as [it's not fully reliable](https://github.com/webpack/webpack/issues/11146). There `chunkhash` may be the preferable option. Webpack 5 provides a better solution for this.

### Example placeholders

Assume you have the following configuration:

```javascript
{
  output: {
    path: PATHS.build,
    filename: "[name].[contenthash].js",
  },
},
```

{pagebreak}

Webpack generates filenames like these based on it:

```bash
main.d587bbd6e38337f5accd.js
vendor.dc746a5db4ed650296e1.js
```

If the file contents related to a chunk are different, the hash changes as well, thus the cache gets invalidated. More accurately, the browser sends a new request for the file. If only `main` bundle gets updated, only that file needs to be requested again.

The same result can be achieved by generating static filenames and invalidating the cache through a querystring (i.e., `main.js?d587bbd6e38337f5accd`). The part behind the question mark invalidates the cache. According to [Steve Souders](http://www.stevesouders.com/blog/2008/08/23/revving-filenames-dont-use-querystring/), attaching the hash to the filename is the most performant option.

## Setting up hashing

The build needs tweaking to generate proper hashes. Adjust as follows:

**webpack.config.js**

```javascript
const commonConfig = merge([
  ...
  parts.loadImages({
    options: {
      limit: 15000,
leanpub-start-delete
      name: "[name].[ext]",
leanpub-end-delete
leanpub-start-insert
      name: "[name].[contenthash:4].[ext]",
leanpub-end-insert
    },
  }),
  ...
])

const productionConfig = merge([
leanpub-start-insert
  {
    output: {
      chunkFilename: "[name].[contenthash:4].js",
      filename: "[name].[contenthash:4].js",
    },
  },
leanpub-end-insert
  ...
]);
```

To make sure extracted CSS receives hashes as well, adjust:

**webpack.parts.js**

```javascript
exports.extractCSS = ({ options = {}, loaders = [] } = {}) => {
  return {
    ...
    plugins: [
      new MiniCssExtractPlugin({
leanpub-start-delete
        filename: "[name].css",
leanpub-end-delete
leanpub-start-insert
        filename: "[name].[contenthash:4].css",
leanpub-end-insert
      }),
    ],
  };
};
```

If you generate a build now (`npm run build`), you should see something:

```bash
⬡ webpack: Build Finished
⬡ webpack: assets by path *.js 129 KiB
    asset vendor.1622.js 126 KiB [emitted] [immutable] [minimized] (name: vendor) (id hint: commons) 2 related assets
    asset main.db11.js 3.4 KiB [emitted] [immutable] [minimized] (name: main) 2 related assets
    asset 34.a4c5.js 257 bytes [emitted] [immutable] [minimized] 2 related assets
  asset main.bdca.css 1.87 KiB [emitted] [immutable] (name: main)
  asset index.html 285 bytes [emitted]
...
  webpack 5.1.3 compiled successfully in 6593 ms
```

The files have neat hashes now. To prove that it works for styling, you could try altering _src/main.css_ and see what happens to the hashes when you rebuild.

W> The hashes have been sliced to make the output fit better in the book. In practice, you can skip slicing them.

## Conclusion

Including hashes related to the file contents to their names allows to invalidate them on the client-side. If a hash has changed, the client is forced to download the asset again.

To recap:

- Webpack's **placeholders** allow you to shape filenames and enable you to include hashes to them.
- The most valuable placeholders are `[name]`, `[contenthash]`, and `[ext]`. A content hash is derived based on the chunk content.
- If you are using `MiniCssExtractPlugin`, you should use `[contenthash]` as well. This way the generated assets get invalidated only if their content changes.

Even though the project generates hashes now, the output isn't flawless. The problem is that if the application changes, it invalidates the vendor bundle as well. The next chapter digs deeper into the topic and shows you how to extract webpack **runtime** to resolve the issue.

T> [Philip Walton has written about caching in detail](https://philipwalton.com/articles/cascading-cache-invalidation/) and his article is a great read if you want to know more about the topic on a more general level.
