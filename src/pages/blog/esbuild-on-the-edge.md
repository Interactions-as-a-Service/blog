---
layout: '../../layouts/Post.astro'
title: Running Esbuild on the Edge
image: /images/esbuild-edge/icon
publishedAt: 2022-11-17
category: 'Cloudflare Workers, WebAssembly'
---

### What is Esbuild?

Esbuild is a JavaScript bundler and minifier. It is a tool that takes your JavaScript code and bundles it into a single file. It can also minify the code, which means it will remove all the unnecessary characters from the code. This makes the code smaller and faster to load. Additionally, it supports parsing and transforming TypeScript and JSX.

The problem is this typically requires either Node.js or Golang to function, which means that you need to run it on some form of a server. This is not ideal for those looking to run their whole apps on the edge, which is what Cloudflare Workers are _perfect_ for.

### How can we get around this?

Recently, Cloudflare introduced a 5Mib script size limit for those on the paid Workers plan, as opposed to the 1Mib limit for the free plan. This increased limit means that we are able to run WASM binaries on the edge, which is conveniently [a format that Esbuild is distributed in](https://www.npmjs.com/package/esbuild-wasm).

### How do we run it?

Let's get started by creating a new Cloudflare Workers project. We can do this by running the following command:

```bash
npx wrangler init
```

Then you can follow the steps to set up your project. This guide will use TypeScript, but you can use JavaScript if you prefer. This will look something like the following

![Wrangler init](/images/shared/wranglerinit.png)

Now, let's navigate to your `src/index.ts` file, and replace the contents with the following:

```ts
import esbuild from 'esbuild-wasm';
import wasm from '../node_modules/esbuild-wasm/esbuild.wasm';

let initialised = false;
globalThis.performance = Date;

export default {
	async fetch(): Promise<Response> {
		if (!initialised) {
			await esbuild.initialize({
				wasmModule: wasm,
				worker: false,
			});
			initialised = true;
		}
		return new Response('Hello World');
	},
};
```

Let's walk through what this code does. We start by importing `esbuild`, as well as it's wasm file. We then create a variable called `initialised` which will be used to check if we have initialised the WASM binary - it's possible if your worker was requested very recently that it's still warm, and doing it again will cause an error. We then polyfill `globalThis.performance` to `Date`, which is required for Esbuild to work, as it needs the `performance.now()` method. Finally, we export a default object with a `fetch` method. This method will be called whenever a request is made to our Workers script.

Within the fetch method, we check if the WASM binary has been initialised. If it hasn't, we initialise it, and set `initialised` to `true`. We then return a new `Response` with the text "Hello World".

For those using TypeScript - we can also add a `types.d.ts` file to the src directory, and add the following:

```ts
declare namespace globalThis {
	// eslint-disable-next-line no-var
	var performance: typeof Date;
}

declare module '*.wasm' {
	const value: WebAssembly.Module;
	export default value;
}
```

This worker should now be functional - let's test it out by running `wrangler dev --local` and navigating to `localhost:8787`. You should see something like the following - if you don't, check your console for any errors:

![Console output](/images/esbuild-edge/runworker.png)

### How do we bundle our code?

Now that we have a working worker, let's bundle our code. We can do this by modify our existing fetch handler to look like the following:

```ts
...
export default {
	async fetch() {
		const code = `const a: number = 1;
console.log(a);`;

		if (!initialised) {
			await esbuild.initialize({
				wasmModule: wasm,
				worker: false,
			});
			initialised = true;
		}

		const result = await esbuild.build({
			bundle: true,
			write: false,
			stdin: {
				contents: code,
				sourcefile: "index.ts",
			},
			format: "esm",
			target: "es2022",
			loader: {
				".ts": "ts",
			},
		});

		const output = result.outputFiles[0].text;

		return new Response(output, {
			headers: {
				"Content-Type": "application/javascript",
			},
		});
	}
}
```

Now, when we visit our worker, we should see the following code, having been bundled and minified:

```js
console.log(1);
```

### How do we make this more useful?

While the above code _works_ - we can make this more useful by adding a few more features. Let's start by implementing a file system in our worker. You can use something more complicated for resolving files if you wish, for example dealing with node modules, URL imports etc. - but for the purposes of this demo, we'll just use a simple object.

```ts
type File = { content: string };

const fileTree: Record<string, File> = {
	'index.ts': {
		content: `import { a } from "./a.ts";
		console.log(a);`,
	},
	'./a.ts': {
		content: `export const a: number = 1;`,
	},
};
```

Now, we need to make an esbuild plugin for accessing this file tree that we've created. This might look something like the following:

```ts
const fileTreePlugin: esbuild.Plugin = {
	name: 'file-tree',
	setup(build) {
		build.onResolve({ filter: /.*/ }, (args) => {
			return { path: args.path, namespace: 'file-tree' };
		});
		build.onLoad({ filter: /.*/, namespace: 'file-tree' }, (args) => {
			const file = fileTree[args.path];
			if (!file) throw new Error(`File not found: ${args.path}`);
			return {
				contents: file.content,
				loader: 'ts',
			};
		});
	},
};
```

In this code snippet, we create a plugin which has 2 handlers. The first handler is called when esbuild is resolving a file. We return the path and an arbitrary namespace, to tell esbuild to use our `onLoad` handler to process it. The second handler is called when esbuild is loading a file. We check if the file exists in our file tree, and if it does, we return the contents of the file, and the loader to use. If the file doesn't exist, we throw an error.

Now, we can modify our fetch handler to use this plugin:

```ts
const result = await esbuild.build({
	bundle: true,
	write: false,
	stdin: {
		contents: fileTree['index.ts'].content,
		sourcefile: 'index.ts',
	},
	format: 'esm',
	target: 'es2022',
	loader: {
		'.ts': 'ts',
	},
	plugins: [fileTreePlugin],
});
```

Key changes here: For this demo we've set the contents of stdin to the contents of the index.ts file from our file tree, and we've added the plugin to the plugins array.

If you run your worker again when visiting it, you should see something like the following output:

```js
// file-tree:./a
var a = 1;

// index.ts
console.log(a);
```

### How do we make this even more useful?

Possible improvements to this worker could include:

- Adding a cache for the bundled code, so that if the same file is requested twice, it doesn't need to be bundled again. Something like Cloudflare's KV could be used for this.
- Adding a way to specify the entrypoint of the application, rather than hardcoding it to index.ts
- Using user code as the entrypoint, rather than your own file tree (this has potential with [Workers for Platforms](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/), for example)

### Notes

This is mostly intended as a proof of concept, as there are a few caveats to using this approach:

- This requires an Unbound worker, as it uses around 20-30 ms of CPU time to run this demo, however spikes up to around 70ms on the 99th percentile. There's very little that can be done to reduce this, as esbuild-wasm is significantly less performant than esbuild's native binary.
- This demo has extremely limited functionality, and is not intended to be used in production. It's only intended to be used as a proof of concept, and to show that it is possible to run esbuild in a worker.
- Your worker is unlikely to receive the full benefits of running on Cloudflare's edge, due to the fact that scripts which are over 1MiB in size (~2.8MiB in this case) are likely to be evicted from Cloudflare Colos when not recently requested, and will need to be re-fetched from the Worker's Core Colos in this event. It may be more performant to instead run Esbuild on your own server.

There are some benefits to using this approach, however:

- You can run esbuild on Cloudflare's edge, which means that you can bundle your code without having to send it to a third party server.
- You can use Cloudflare's KV to cache the bundled code, so that if the same file is requested twice, it doesn't need to be bundled again.
- For those not willing to deal with hosting a server, this alternative is a good option.
- This approach does not have any access to File Systems, so there is no risk of a malicious script managing to escape Esbuild's bundling, and accessing any sensitive content.

### Conclusion

This demo is just showcasing some of the potential of esbuild, and how it can be used to bundle code in a worker. It's not meant to be a production-ready solution, but rather a proof of concept. I hope you've enjoyed this article, and I hope you've learned something new. If you have any questions, feel free to reach out to me on the [Interactions.Rest Discord](https://discord.gg/j22RkU7eBr).

The code for this demo can be found [here](https://github.com/Interactions-as-a-Service/esbuild-worker-demo).

Thanks for reading!
