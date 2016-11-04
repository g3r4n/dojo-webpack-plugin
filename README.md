# Introduction

The DojoAMDPlugin is a [Webpack](https://webpack.github.io/) plugin that supports using Webpack to build Dojo 1.x applications (tested with version 1.10).  Features include:

* Support for Dojo loader config properties, including `paths`, `packages`, `map` and `aliases`
* Support for client-side synchronous and asynchronous `require()` calls for packed modules.
* Webpack loader implementations of standard Dojo loaders (e.g. `dojo/i18n`).
* Limited support for client side execution of some Dojo loaders.

# The Dojo loader

The DojoAMDPlugin uses the Dojo loader (dojo.js) at build time to resolve modules based on the properties specified in the Dojo loader config.  In addition, a stripped-down build of the loader, as well as the loader config, are embedded in the packed application to enable client-side resolution of modules by the `require()` function.  Client-side `require()` is needed for calls that cannot be transformed by Webpack at build time because of module identifiers that cannot be statically evaluated.  Although the client-side `require()` function may be called to obtain a reference to a module which is included in the packed assets, it cannot be used to load non-packed modules.  However, the `require.toAbsMid()` and `require.toUrl()` functions may be called on the client to resolve module names and module URLs. 

This package does not include the Dojo loader.  The loader will be built by Webpack based on the location of Dojo specified in the 
Dojo loader config (see below).  The built loader is packaged as a CommonJS module so that it may be more easily consumed by Webpack.  The build also specifies has.js features which exclude unneeded code (e.g. for loading modules) so that the loader embedded into the client is as small as possible (~4KB after uglify and gzip). 

The Dojo loader builder assumes that the Dojo `util` directory is a sibling of the `dojo` directory.  If you do not want to build the Dojo loader every time Webpack is run, then you can build it manually and specify the location of the built loader using the `loader` option.  You can produce a manual build of the loader by running the build script in the buildDojo directory.

        node buildDojo/build.js ../../dojo/dojo ../release

The example above will build the loader and place it in the `../release` directory, relative to the current directory.  Again, the Dojo util directory must be located at `../../util` in order for the build to succeed.

To have Webpack use the built loader, specify the location of the loader in the plugin options as follows:

        plugins: [
            new DojoAMDPlugin({
                loaderConfig: require("./loaderConfig"),
                locales: ["en"],
                loader: path.join(__directory, "../release/dojo/dojo.js")
            }),
        ]

# The Dojo loader config

The loader config defines properties used in resolving module identifiers as described in [Configuring Dojo with dojoConfig](https://dojotoolkit.org/documentation/tutorials/1.7/dojo_config/).  Note that not all properties in the loader config are used by Webpack.  Only properties relating to module name/path resolution are used.  These include `baseUrl`, `packages`, `paths`, `map` and `aliases`.  The loader config may also specify a `has` map of feature-name/value pairs. The `has` features are used in resolving `dojo/has` loader conditionals at build time, and to provide the initial values for the run-time has.js feature detection functionality provided by `dojo/has`.  The loader config is specified by the `loaderConfig` options property and is mixed into the global `dojoConfig` property on the client by the Webpack bootstrap code.

Because the loader config is used to resolve module paths both at build time, and on the client, you may need to conditionally specify some properties, such as `baseUrl`, depending on whether the current environment is node or a browser.  This may be necessary if you need `require.toUrl()` to return a valid URLs on the client.

# Dojo loader extensions

Loader extensions are used to provide special processing when loading modules.  Loader extensions prefix the module being loaded, separated by the `!` character.  Both Dojo and Webpack have the concept of loader extensions, but the implementation are very different, and they use conflicting terminology.  Dojo refers to them as plugins and Webpack refers to them as loaders.  To avoid confusion, we refer to them both in this document as loader extensions. 

Dojo loader extensions generally cannot be used with Webpack.  There are several approaches to dealing with Dojo loader extensions.

* Replace the Dojo loader extension with a compatible Webpack extension.  For example, the `dojo/text` loader extension can be replaced with the Webpack `raw` loader extension.  This can be done with code similar to the following in your `webpack.config.js`.

        plugins: {
            new DojoAMDPlugin({...}),
            new webpack.NormalModuleReplacementPlugin(/^dojo\/text!/, function(data) {
                data.request = data.request.replace(/^dojo\/text!/, "raw!");
            })
        }
    This replacement (among others) is automatically configured for you, so you don't need to include this in your webpack.config.js.  It is provided here as an example of what you could do with other loader extensions. 


* Replace the entire module expression with the desired module.  Some Dojo loader extensions are used to dynamically load one module or another based on runtime conditions.  An example is the gfx loader, which loads the rendering engine supported by the client.  Since all modern browsers support the `canvas` rendering engine, you can replace the module expression that includes the loader with the module expression for the target module.

        new NormalModuleReplacementPlugin(/^dojox\/gfx\/renderer!/, "dojox/gfx/canvas")

* Implement the Dojo loader extension as a Webpack loader extension.  This is what is done with the `dojo/i18n` loader extension.

* Use the `dojo/loaderProxy` Webpack loader extension provided by this package to proxy Dojo loader extensions on the client.  More information on this is provided in the following section.

The DojoAMDPlugin defines the following loader extension replacements:

          new webpack.NormalModuleReplacementPlugin(/^dojo\/selector\/_loader!/, "dojo/selector/lite"),
          new webpack.NormalModuleReplacementPlugin(/^dojo\/request\/default!/, "dojo/request/xhr"),
          new webpack.NormalModuleReplacementPlugin(/^dojo\/text!/, function(data) {
              data.request = data.request.replace(/^dojo\/text!/, "raw!");
          })

You can override these replacements by specifying your own replacements in the `plugins` property of your `webpack.config.js` file immediately following the registration of the DojoAMDPlugin.

# The loaderProxy loader extension

`dojo/loaderProxy` is a Webpack loader extension that enables Dojo loader extensions to run on the client.  Not all Dojo loader extension may be used this way.  The basic requirement is that the Dojo loader extension's `load` method invokes its callback in-line, before returning from the `load` method.  The most common use cases are loader extensions that delegate to `dojo/text` or another supported loader extension to load the resource before doing some processing on the result.

Consider a simple svg loader extension that loads the specified svg file and fixes up the contents by removing the xml header in the content.  The implementation of the load method might look like this:

        load: function (name, req, callback) {
          req(["dojo/text!" + name], function(text) {
            callback(stripHeader(text).trim());
          });
        }

Here, the load method delegates to a loader extension that is supported by Webpack to load the resource.  If the resource is included in the packed modules, then the `req` callback will be invoked in-line, and thus the `load` method's callback will be invoke in-line.  If the `load` method's callback is not invoked before the `load` method returns, then an exception will be thrown.

You can use the `dojo/loaderProxy` with the Webpack NormalModuleReplacementPlugin as follows:

        new webpack.NormalModuleReplacementPlugin(
       	    /^svg!/, function(data) {
        	        var match = /^svg!(.*)$/.exec(data.request);
        	        data.request = "dojo/loaderProxy?loader=svg&deps=dojo/text%21" + match[1] + "!" + match[1]);
            }
        )

The general syntax for the `dojo/loaderProxy` loader extension is `dojo/loaderProxy?loader=<loader>&deps=<dependencies>!<resource>` where *loader* specifies the Dojo loader extension to run on the client and *dependencies* specifies a comma separated list of module dependencies to add to the packed resources.  In the example above, if the client code specifies the module as `svg!closeBtn.svg`, then the translated module will be `dojo/loaderProxy?loader=svg&deps=dojo/text%21closeBtn.svg!closeBtn.svg`.  Note the need to URL encode the `!` character so as not to trip up parsing.

Specifying `dojo/text!closeBtn.svg` as a dependency ensures that when it is required by the `svg` loader extension's load method on the client, then the dependency will be resolved in-line and the `load` method's callback will be invoked in-line as required.

# Miscellanious Notes

When using Webpack's NormalModuleReplacementPlugin, the order of the plugin registration relative to the DojoAMDPlugin's registration is significant.  The DojoAMDPlugin resolves `dojo/has` loader extension conditionals in module expressions, and converts the module expression to an absMid (relative paths resolved, maps and aliases applied), so if the NormalModuleReplacementPlugin is registered after the DojoAMDPlugin, then `data.request` will contain the resolved absMid for the module and `data.originalRequest` will contain the original module expression before transformation by the DojoAMDPlugin.  If the NormalModuleReplacementPlugin is registered before the DojoAMDPlugin, then the NormalModuleReplacementPlugin will get to modify the request before before the DojoAMDPlugin applies its transformations.

# Sample application

See the sample application located in the [sample](https://git.swg.usma.ibm.com/chuckd/dojo-webpack-plugin/tree/master/sample) folder.  It can be built by running the build.js with node.

https://git.swg.usma.ibm.com/pages/chuckd/dojo-webpack-plugin/sample/test.html