# HUAWEI CloudIDE plugin Generator
This generator will help you create a HUAWEI CloudIDE plugin project.

## How to use
The CloudIDE plugin can be developed using the CloudIDE itself. 
You can obtain a CloudIDE instance from [Huawei CloudIDE official site](https://www.huaweicloud.com/product/cloudide.html). 

### Install Yeoman and CloudIDE plugin generator
```bash
npm install -g yo @cloudide/generator-plugin
```

### Run Generator using yo command
The generator will ask you a few questions and create the project as you choose. 
```bash
yo @cloudide/plugin
```

### Build Your Plugin
After run the following command 
```bash
npm i && npm run pack
```

## Plugin Development Guides
### Overview
This guide describes the basic running principles of plugin and describes the API design of the plugin core framework through examples. 
All of this article is based on a generic project that can be created by executing `yo @cloudide/plugin` and selecting the `generic` type plugin.  

### What's The Benefit
* The plugin frontend supports pure H5 implementation, no need to learn complex IDE extension points, and the cost of getting started is extremely low.
* The plugin supports directly calling the exposed methods between the frontend and backend, and returns Promise, without directly using postMessage for message sending.
* Multi-view support, support for dynamically creating and destroying webviews, and direct calling of methods exposed between them.
* Support for event subscription at the frontend.
* Support event broadcasting between plugins.

### Basic Concepts 
* **backend**: JS code running in the NodeJS environment. Actually, the backend of CloudIDE instance is a web server started with express. 
* **frontend**: JS code running in the Browser environment.
* **remote call**: Make a remote function call exposed between the frontend and backend.
* **scope**: When we exposed a function in the backend or frontend, the running environment is the scope of the function.
For the frontend, the scope is the webview page where the frontend class is located. The *viewType* of the webview is used to indicate the scope. 
For the backend, all backend classes are restricted to the same scope called *backend*.
When making a remote call to other scope, we should specify the scope before the identifier.
***If no scope is specified, the backend is called by default when the frontend calls the remote function,
and the plugin main page fronted is called by default when the backend calls the remote function.***
For more examples, see [Dynamic Webview](#dynamic-webview). 

### Directory Structures of Plugin Project
Once you have created a generic plugin project using the yo generator, the root directory of the plugin project contains the following directories and files: 
```bash
your-awesome-plugin
├── LICENSE
├── package.json
├── package-lock.json
├── resources
│   ├── icons
│   │   ├── logo.png
│   │   └── plugin.svg
│   └── page
│       ├── css
│       │   ├── default.css
│       │   └── main.css
│       ├── dynamic-webview.html
│       └── index.html
├── src
│   ├── browser
│   │   ├── dynamic-webview.ts
│   │   └── frontend.ts
│   ├── node
│   │   └── backend.ts
│   └── plugin.ts
├── tsconfig.json
├── tsfmt.json
└── webpack.config.js
```
* **plugin.ts**: Entry of plugin.
* **src/browser**: Stores code that needs to be run on the browser side(frontend).
* **src/node**: Stores code that needs to be run on the server side with NodeJS(backend).
* **resources/page/index.html**: Plugin main page, which will be loaded by default.
* **resources/page/dynamic-webview.html**: Dynamic webview can be loaded programmatically, you can add dynamic webview files as needed.
* **resources/page/css/default.css**: Basic style sheets, do not modify this file.
* **resources/page/css/main.css**: Style sheets for your plugin page, add your own style sheets to this file.
* **resources/icons/logo.png**: Plugin logo, which will be displayed on marketplace.
* **resources/icons/plugin.svg**: Plugin icon, which will be displayed on CloudIDE panel.

### Plugin Implementation
In this section, we will go through the code to illustrate how plugin is loaded.
After being started, CloudIDE instances load plugins in sequence based on the dependency. 

#### Entry File
The default entry file of plugin is **plugin.ts**, which export `start` and `stop` functions. `start` function is invoked by default
if activationEvents registered as `*`, `stop` function is called before the plugin is stopped. 
You can add your own clean-up actions to this method.
```typescript
/**
 * Plugin activation entry point, this function is called when plugin is loaded.
 */
export function start(context: cloudide.ExtensionContext) {

    const opts: WebviewOptions = {
        viewType: 'view_type_of_your_plugin_view',
        title: 'title_of_your_plugin_view',
        targetArea: 'right',
        iconPath: 'resources/icons/plugin.svg',
        viewUrl: 'local:resources/page/index.html',
        preserveFocus: true
    };

    /**
     * The backend class that needs to be loaded, the classes in the array must inherit AbstractBackend.
     * Usually you only need to add the methods you want to expose to the frontend in the backen.ts and implement the plugin function in the run method.
     * If you want to define the backend class yourself, just refer to the implementation of the Backend class and add the type definition to the array.
     */
    const backends = [Backend];

    /**
     * Register a startup command for the plugin.
     * After closing the plugin, you can restart the plugin by entering the command displayed by the label through the F1 command. 
     * command registeration can be removed as needed.
     */
    context.subscriptions.push(
        cloudide.commands.registerCommand(opts.title, () => {
            Plugin.createOrShow(context, opts, backends);
        })
    );

    Plugin.createOrShow(context, opts, backends);
}

/**
 * The method that is called when the plugin is stopped. 
 * If you need to customize the clean-up action that the plug-in stops, you can add it to the method.
 */
export function stop(context: cloudide.ExtensionContext) {
    Plugin.getInstance().stop();
}
```

In `start` function, we declare a variable `opts`, the type of this variable is WebviewOptions.
This variable defined the basic configuration of the plugin main page. 
You can change the properties of opts as needed. 
```typescript
export interface WebviewOptions {
    /**
     * The unique type identifier of the plugin view, which is determined by yourself.
     */
    viewType: string;
    /**
     * The title of the plugin page, which is displayed at the very top of the plugin.
     */
    title: string;
    /**
     * The default view area of the plugin view.
     * Supports left ('left'), right'right', main editing area ('main'), bottom ('bottom').
     */
    targetArea: string;
    /**
     * Plugin icon displayed on the panel.
     * The icon in svg format can automatically adapt to the theme color.
     */
    iconPath: {
        light: string;
        dark: string;
    } | string;
    /**
     * The path of the page to be displayed.
     * Local page resources are placed under "resources" by default, and starting with "local:".
     * Remote page cannot interact with the plugin backend.
     */
    viewUrl: string;
    /**
     * when true, on main area the webview will not take focus, on left and right panel the webview will not be expanded.
     */
    preserveFocus?: boolean;
    /**
     * Extra data passed to the view.
     * Getting extra data using 'plugin.cloudidePluginApi.getExtData()' in frontend.
     */
    extData?: any;
}
```

After completing the basic configuration, we defined an array named `backends` to hold the backend classes. 
All backend classes that need to expose methods to the frontend should inherit the backend abstract class named `AbstractBackend`. 
We have implemented a Backend class by default, and we usually just need to implement your backend program in this class.
If you want to implement your own backend class, you can refer to the implementation of the default `Backend` class and add it to the array.

#### Backend Class
The backend classes are the facades responsible for interacting with the IDE and OS.
If you want the frontend to call your backend methods easily, you can define a class that implements the AbstractBackend.
We add a `@exposable` decorator to the backend class indicate that the public member function of the class can be exposed to frontend using `@expose` decorator.
The following uses the default `Backend` class as an example to describe how to write a backend class.
```typescript
/**
 * Add your backend api in this class
 * Using '@expose' to expose your function to frontend
 */
@exposable
export class Backend extends AbstractBackend {

    /**
     * function call to the backend will wait until init() to be resolved
     */
    async init(): Promise<void> {

    }

    /**
     * Entry of your plugin backend
     * In this function you can call function exposed by frontend 
     */
    public async run(): Promise<void> {
        const retValue = await this.plugin.call('myplugin.page.myApi', 'this is a function call from backend');
        this.plugin.log(LogLevel.INFO, retValue);
    }

    public stop(): void {

    }

    /**
     * this function can be called from plugin frontend as below:
     * @example
     * ```
     * plugin.call('your_backend_function_identifier', 'world').then(ret => {
     *     console.log(ret);
     * });
     * 
     * ```
     */
    @expose('your_backend_function_identifier')
    public doSomething(name: string): boolean {
        cloudide.window.showInformationMessage(`hello ${name}!`);
        return true;
    }

}
```

* `async init(): Promise<void>` - The `init` function can do some initialization actions that will be called before the `run()` method is executed, 
and calls to the exposed backend function will wait until `init()` to be resolved.
* `public async run(): Promise<void>` - The `run` function is called after init, in this function you can call function exposed by frontend.
* `public stop(): void` - The `stop` function is called before plugin stop, you can do some clean up actions before plugin stopped.
* expose your own function to frontend - Just add `@expose` decorator to your funciton, and pass a parameter as unique identifier, which stand for this function when called using plugin.call.

#### Frontend Class
The design of the frontend class is similar to that of the backend. 
The frontend class should inherit an abstract class called AbstractFrontend and needs to be decorated with `@expose`, providing three methods: `init`, `run`, and `stop`.
The usage of the three methods are quite similar to those of the backend. The only difference is that they all run in the browser environment.
```typescript
/**
 * Adding your fronted api in this class
 * Using '@expose' to expose your function to frontend
 */
@exposable
class Frontend extends AbstractFrontend {

    /**
     * function call to the frontend will wait until init() to be resolved
     */
    async init(): Promise<void> {

    }

    /**
     * Entry of your plugin frontend
     * In this function your can call function exposed by frontend
     */
    run(): void {
        this.plugin.call('your_backend_function_identifier', 'world');
    }

    stop(): void {

    }

    /**
     * this function can be called from plugin backend as below:
     * @example
     * ```
     * plugin.call('myplugin.page.myApi', 'this is a function call from backend').then(ret => {
     *     console.log(ret);
     * });
     * 
     * ```
     */
    @expose('myplugin.page.myApi')
    public myApi(message: string): string {
        console.log(message);
        return 'this is a return value from frontend function';
    }

}
```

After defining the frontend class, you need to create the frontend class explicitly. 
You can initialize the frontend class after the DOM element is loaded on the browser page.
The following code shows how to load the defined frontend class explicitly. 
Multiple frontend classes need to be added to the array passed to `Plugin.create`. 
```typescript
document.addEventListener('DOMContentLoaded', function() {
    PluginPage.create([Frontend]);
});
```

#### Dynamic Webview
In some scenarios, we need to dynamically create views on the CloudIDE workbench. 
The feature of *dynamic webview* provides an approach to create webview programmatically in frontend.
The dynamic views, plugin backend, and plugin main view have the ability to expose function to each other.

Dynamic webview can be added in 5 steps:
* step 1: Create a dynamic webview html page in `resources/page/` just like the default dynamic-webivew.html. 
You can copy the dynamic-webview.html file and rename it as you like. For example, `your-dynamic-webviw.html`
* step 2: Create your frontend file in `src/browser/` just like the default dynamic-webview.ts.
You can copy the dynamic-webview.ts file and rename it as you like. For example, `your-dynamic-webview.ts`
* step 3: Add an entry to the `webpack.config.js`.
```javascript
module.exports = {
    entry: {
        'page/dist/index': './src/browser/frontend.ts',
        'page/dist/dynamic-webview-index': './src/browser/dynamic-webview.ts',
        'page/dist/your-dynamic-webview': './src/browser/your-dynamic-webviw.ts'
    },
    ...
};
```
* step 4: Change the src attribute to your compiled js entry.
```html
<head>
    ...
	<script type="text/javascript" src="dist/your-dynamic-webviw.js" crossorigin="anonymous"></script>
    ...
</head>
```
* setp 5: Call `Plugin.createDynamicWebview()` to create the dynamic webview, `opts` passed to the
create function is a type of `WebviewOptions`, you should change the properties to your dynamic view.
```typescript
    run(): void {
       ...
        const dynamicWebviewOpts = {
                viewType: 'your-dynamic-webview',
                title: 'your dynamic webview',
                targetArea: 'main',
                iconPath: 'resources/icons/plugin.svg',
                viewUrl: 'local:resources/page/your-dynamic-webview.html',
                preserveFocus: false,
                extData: { data: 'my "extra" data' }
        };
        this.plugin.createDynamicWebview(dynamicWebviewOpts, true);
       ...
    }
```
Once you've done the above, you can easily complete frontend development using HTML5 page,
and then we'll go into the details of how each webview and the backend call each other. 

As shown in the following figure, assume that there are three webviews, two of which are dynamic webviews. 
In each webview, a frontend class exposes a method. The method identifiers are *funcA*, *funcB*, and *funcC*. 
Three backend classes expose a method. The method identifiers are *funcD*, *funcE*, and *funcF*. 
```bash
(plugin main webview)                            (dynamic webview)              (dynamic webview)
(viewType: main)                                 (viewType: view1)              (viewType: view2)
(exposed function id: funcA)                     (exposed function id: funcB)   (exposed function id: funcC)
+------------------+                             +------------------+           +------------------+
| +----------+     | plugin.call('main::funcA')  | +----------+     |           | +----------+     |
| |          |     +<----------------------------+ |          |     |           | |          |     |
| | frontend | ... |                             | | frontend | ... |           | | frontend | ... |
| |          |     | plugin.call('view1::funcB') | |          |     |           | |          |     |
| +----------+     +---------------------------->+ +----------+     |           | +----------+     |
+------------------+                             +------------------+           +------------------+
  ^                                                |       ^                             |
  |                                                |       | plugin.call('view1::funcB') |
  |                                                |       +-----------------------------+
  |                                                |
  | plugin.call('main::funcA')                     | plugin.call('backend::funcD')
  |                                                v
+------------------------------------------------------------------------------------------------------+
| +----------------------------+ +----------------------------+ +----------------------------+         |
| | backend                    | | backend                    | | backend                    |         |
| |                            | |                            | |                            | ......  |
| | exposed function id: funcD | | exposed function id: funcE | | exposed function id: funcF |         |
| |                            | |                            | |                            |         |
| +----------------------------+ +----------------------------+ +----------------------------+         |
+------------------------------------------------------------------------------------------------------+
```
As mentioned on [Basic Concepts](#basic-concepts), each frontend or backend has its own scope. 
The scope of frontend is its viewType, while backend classes are all *backend*.
The remote call always return Promise, you can `await` or using `then` callback to wait the Promise to be resolved. 

## LICENSE
[MIT](LICENSE)
