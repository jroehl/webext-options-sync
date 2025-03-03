# webext-options-sync [![Travis build status](https://api.travis-ci.org/fregante/webext-options-sync.svg?branch=master)](https://travis-ci.org/fregante/webext-options-sync) [![npm version](https://img.shields.io/npm/v/webext-options-sync.svg)](https://www.npmjs.com/package/webext-options-sync)

> Helps you manage and autosave your extension's options.

Main features:

* Define your default options
* Add autoload and autosave to your options `<form>`
* Run migrations on update

## Install

You can just download the [standalone bundle](https://packd.fregante.now.sh/webext-options-sync@latest?name=OptionsSync) (it might take a minute to download) and include the file in your `manifest.json`, or:

```sh
npm install webext-options-sync
```

```js
import OptionsSync from 'webext-options-sync';
```

The [browser-extension-template](https://github.com/notlmn/browser-extension-template) repo includes a complete setup with ES Modules, based on the advanced usage below.

## Usage

This module requires the `storage` permission in `manifest.json`:

```json
{
	"name": "My Cool Extension",
	"permissions": [
		"storage"
 	]
}
```

### Simple usage

You can set and get your options from any context (background, content script, etc):

```js
/* global OptionsSync */
const optionsStorage = new OptionsSync();

await optionsStorage.set({showStars: 10});

const options = await optionsStorage.getAll();
// {showStars: 10}
```

### Advanced usage

It's suggested to create an `options-storage.js` file with your defaults and possible migrations, and import it where needed:

```js
/* global OptionsSync */
window.optionsStorage = new OptionsSync({
	defaults: {
		colorString: 'green',
		anyBooleans: true,
		numbersAreFine: 9001,
	},

	// List of functions that are called when the extension is updated
	migrations: [
		(savedOptions, currentDefaults) => {
			// Perhaps it was renamed
			if (savedOptions.colour) {
				savedOptions.color = savedOptions.colour;
				delete savedOptions.colour;
			}
		},

		// Integrated utility that drops any properties that don't appear in the defaults
		OptionsSync.migrations.removeUnused
	]
});
```

Include this file as a background script: it's where the `defaults` are set for the first time and where the `migrations` are run. This example also includes it in the content script, if you need it there:

```json
{
	"background": {
		"scripts": [
			"webext-options-sync.js",
			"options-storage.js",
			"background.js"
		]
	},
	"content_scripts": [
		{
			"matches": [
				"https://www.google.com/*",
			],
			"js": [
				"webext-options-sync.js",
				"options-storage.js",
				"content.js"
			]
		}
	]
}
```

Then you can use it this way from the `background` or `content.js`:

```js
/* global optionsStorage */
async function init () {
	const {colorString} = await optionsStorage.getAll();
	document.body.style.background = colorString;
}

init();
```

And also enable autosaving in your options page:

```html
<!-- Your options.html -->
<form>
	<label>Color: <input name="colorString"/></label><br>
	<label>Show: <input type="checkbox" name="anyBooleans"/></label><br>
	<label>Stars: <input name="numbersAreFine"/></label><br>
</form>

<script src="webext-options-sync.js"></script>
<script src="options-storage.js"></script>
<script src="options.js"></script>
```

```js
// Your options.js file
/* global optionsStorage */

optionsStorage.syncForm(document.querySelector('form'));
```

### Form autosave and autoload

When using the `syncForm` method, `OptionsSync` will serialize the form using [dom-form-serializer](https://github.com/jefersondaniel/dom-form-serializer), which uses the `name` attribute as `key` for your options. Refer to its readme for more info on the structure of the data.

Any changes to the form are immediately saved into `chrome.storage.sync` via `input` events debounced by 600ms.

#### Input validation

If your form fields have any [validation attributes](https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/HTML5/Constraint_validation#Validation-related_attributes) they will not be saved until they become valid.

<details>

Since autosave and validation is silent, you should inform the user of invalid fields, possibly via CSS by using the `:invalid` selector:

``` css
/* Style the element */
input:invalid {
	color: red;
	border: 1px solid red;
}

/* Or display a custom error message */
input:invalid ~ .error-message {
	display: block;
}
```

</details>

## API

#### const optionsStorage = new OptionsSync([setup])

##### setup

Type: `object`

Optional. It should follow this format:

```js
{
	defaults: { // recommended
		color: 'blue'
	},
	migrations: [ // optional
		savedOptions => {
			if(savedOptions.oldStuff) {
				delete savedOptions.oldStuff
			}
		}
	],
}
```

Returns an instance linked to the chosen storage.

###### defaults

Type: `object`

A map of default options as strings or booleans. The keys will have to match the options form fields' `name` attributes.

###### migrations

Type: `array`

A list of functions to run in the `background` when the extension is updated. Example:

```js
{
	migrations: [
		(savedOptions, defaults) => {
			// Change the `savedOptions`
			if(savedOptions.oldStuff) {
				delete savedOptions.oldStuff
			}

			// No return needed
		},

		// Integrated utility that drops any properties that don't appear in the defaults
		OptionsSync.migrations.removeUnused
	],
}
```

###### storageName

Type: `string`
Default: `'options'`

The key used to store data in `chrome.storage.sync`

###### logging

Type: `boolean`
Default: `true`

Whether info and warnings (on sync, updating form, etc.) should be logged to the console or not.

#### opts.set(options)

This will merge the existing options with the object provided.

**Note:** Any values specified in `default` are not saved into the storage, to save space, but they will still appear when using `getAll`. You just have to make sure to always specify the same `defaults` object in every context (this happens automatically in the [Advanced usage](#advanced-usage) above.)

##### options

Type: `object`
Default: `{}`
Example: `{color: red}`

A map of default options as strings, booleans, numbers and anything accepted by [dom-form-serializer](https://github.com/jefersondaniel/dom-form-serializer)’s `deserialize` function.

#### opts.setAll(options)

This will override **all** the options stored with your `options`.

#### opts.getAll()

This returns a Promise that will resolve with all the options.

#### opts.syncForm(form)

Any defaults or saved options will be loaded into the `<form>` and any change will automatically be saved via `chrome.storage.sync`

##### form

Type: `HTMLFormElement`, `string`

It's the `<form>` that needs to be synchronized or a CSS selector (one element). The form fields' `name` attributes will have to match the option names.

#### opts.stopSyncForm()

Removes any listeners added by `syncForm`.

## Related

* [webext-storage-cache](https://github.com/fregante/webext-storage-cache) - Map-like promised cache storage with expiration.
* [webext-domain-permission-toggle](https://github.com/fregante/webext-domain-permission-toggle) - Browser-action context menu to request permission for the current tab.
* [webext-dynamic-content-scripts](https://github.com/fregante/webext-dynamic-content-scripts) - Automatically inject your `content_scripts` on custom domains.
* [webext-detect-page](https://github.com/fregante/webext-detect-page) - Detects where the current browser extension code is being run.
* [webext-content-script-ping](https://github.com/fregante/webext-content-script-ping) - One-file interface to detect whether your content script have loaded.
* [`Awesome WebExtensions`](https://github.com/fregante/Awesome-WebExtensions): A curated list of awesome resources for Web Extensions development

## License

MIT © [Federico Brigante](https://bfred.it)
