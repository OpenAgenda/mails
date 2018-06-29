# @openagenda/mails

Build and send responsive e-mails from Node.js.

MJML + EJS + Nodemailer = :heart:

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes.

This project also allows you to create templates with preview and refresh in real time.

### Installing

```bash
yarn add @openagenda/mails

# or `npm i @openagenda/mails`
```

### Initializing

Before using it you must initialize the service, the configuration needs to know where to find the templates, how to send them, then optionally the default values for each send (for example: *domain*, *lang*) and the translations of your templates.

```js
const mails = require( '@openagenda/mails' );

/* Default configuration */

const config = {
  // Templating
  templatesDir: process.env.MAILS_TEMPLATES_DIR || path.join( __dirname, 'templates' ),

  // Mailing
  transport: {
    pool: true,
    host: '127.0.0.1',
    port: '1025', // Mailcatcher port
    maxMessages: Infinity,
    maxConnections: 20,
    rateLimit: 14, // 14 emails/second max
    rateDelta: 1000
  },
  defaults: {},

  // Localization
  translations: {
    labels: {},
    makeLabelGetter
  },

  // Queuing
  redis: {
    host: 'localhost',
    port: 6379
  },
  queueName: 'mails',
  disableVerify: false
};

mails.init( config )
  .then( () =>
     console.log( 'Service mails initialized' )
   )
  .catch( error =>
    console.log( 'Error on initializing service mails', error )
  );
```

More details on the options in the [API section](#API).

### Example

```js
const { results, errors } = await mails( {
  template: 'helloWorld',
  to: {
    address: 'kevin.bertho@gmail.com',
    data: { username: 'bertho', lang: 'fr' }
  }
} );
```

### Building templates

The templates can come from an independent folder by setting the environment variable `MAILS_TEMPLATES_DIR` or setting `templatesDir` at initialization.

Then run `yarn start` and navigate to [http://localhost:3000](http://localhost:3000).
The home page is the list of templates available in the chosen folder (`templates` by default), once on the template to edit you just have to save your changes to see the changes in your browser. 

Each template has a folder with its name, in there must be at least one file `index.mjml` and `fixtures.js`.

**index.mjml** is the entry point of your template, it can be split into different partials (see [`mj-include`](https://mjml.io/documentation/#mj-include)).
**fixtures.js** are data that are used in the template to preview as in production.

The structure of your templates folder can look like this:
```
/templates
  /helloWorld
    fixtures.js
    index.mjml
  /accountActivation
    fixtures.js
    index.mjml
 ```

## API

### Configuration

#### `init( options )`

Returns a Promise.

**Usage**
```js
const mails = require( '@openagenda/mails' );

/* Dafault values */

await mails.init( {
  // Templating
  templatesDir: process.env.MAILS_TEMPLATES_DIR || path.join( __dirname, 'templates' ),

  // Mailing
  transport: {
    pool: true,
    host: '127.0.0.1',
    port: '1025',
    maxMessages: Infinity,
    maxConnections: 20,
    rateLimit: 14, // 14 emails/second max
    rateDelta: 1000
  },
  defaults: {},

  // Localization
  translations: {
    labels: {},
    makeLabelGetter
  },

  // Queuing
  redis: {
    host: 'localhost',
    port: 6379
  },
  queueName: 'mails'
} );
```

**Arguments**
- `{Object} options`

Name | Type | Description
---|:---:|---|---
`options` | `Object` | The options to initializing the service. 

**options**
Value | Required | Description
|---:|:---:|---
|`templatesDir` | * | The folder path containing your templates.
|`transport` | * | An object that defines connection data, it's the first argument of `nodemailer.createTransport` ([SMTP](https://nodemailer.com/smtp/) or [other](https://nodemailer.com/transports/)).
|`defaults` |  | An object that is going to be merged into every message object. This allows you to specify shared options, for example to set the same _from_ address for every message. It's the second argument of `nodemailer.createTransport`.
|`translations` |  | An object containing `labels` and |`makeLabelGetter` keys. <br />- `labels` is an object of labels, one key per template. <br />- `makeLabelGetter( labels, defaultLang )` is a function that returns a function that can be called in templates with `__`. <br />By default the `__` signature is `( name, values, lang )` and the values in the label are replaced when they are surrounded by `%`, for example a label like `Hello %username%` hope to receive `{ username }`
|`redis` | * | An object with your Redis connection data, which will be used to stack your mails in a queue. <br />`{ host, port }` ([@openagenda/queues](https://github.com/Oagenda/queues))
|`queueName` | * | A string that is the name of your Redis queue. 
|`disableVerify` |  | A Boolean that allows to disable the verification of the transporter connection, it is done in the init.
|`logger` |  | An object for the method `setModuleConfig` of https://github.com/Oagenda/logs

During initialization a `queue` and a `transporter` are added to the config, you can use them raw from anywhere with a require of `@openagenda/mails/config`.

### Mailing

#### `sendMail( options )`

This is the main method, the one exported by default.
This function returns a Promise with the value:

- an array of Redis IDs if the queue is activated
- an array of nodemailer `sendMail` results if the queue is disabled

This is a nodemailer `sendMail` overload with some notable differences:

- You can use a template.
- The email addresses are validated before sending.
- The sending of emails is never grouped, the recipients of the messages are always separated, which makes it possible to attach data by recipient.
- Emails can be stored in an external queue while waiting for their turn.

**Usage**
```js
const mails = require( '@openagenda/mails' );

await mails( {
  template: 'helloWorld',
  to: {
    address: 'kevin.bertho@gmail.com',
    data: { username: 'bertho', lang: 'fr' }
  },
  queue: false
} );
```

**Arguments**
- `{Object} options`

Name | Type | Description
---|:---:|---|---
`options` | `Object` | The options to sending email(s).

**Options**
| Value | Required | Description |
|--:|:--:|--|
| template |  | A string that is the name of the template, is equal to the folder name. |
| lang |  | A string that defines the default language that will be applied to all recipients without lang. |
| to | * | A recipient or array of recipients. |
| queue |  | A Boolean, if false do not queue job and execute directly.
| **...** |  | **All other nodemailer properties are normally handled by nodemailer, see the other options here (https://nodemailer.com/message/).**



***Error handling***
`sendMail` does not throw an error in case of problem, it returns an object `{ results, errors }`.
It allows not to block the sending of emails for all when there is only a malformed email address in the batch, for example.

***Recipients***
You will find more information on the nodemailer documention (https://nodemailer.com/message/addresses/).
The main difference is that the email is sent separately to each recipient, one mail/one recipient.
If you want to add specific data to a recipient for the template (for example: its name, age, role, etc.) you must use an object with the data key, the language of the recipient can be in the data key:
```js
{
  address: 'kevin.bertho@gmail.com',
  data: { lang: 'fr', username: 'bertho' }
}
```

***Defaults***
It's an object that is going to be merged into every message object. This allows you to specify shared options, for example to set the same _from_ address for every message.
Defaults can not be overloaded, except for `data`, `headers` and `lang`.

***Data order***
The data come from several sources, they are `Object.assign`ed in this order:

 - `data` from the `sendMail` options
 - `data` from the current recipient (`recipient.data`)
 - `data` from `defaults.data` lastly for conserve values like *domain*, etc

***Language***
As for data, the language can be overloaded in several places, in this order:

 - `{ lang }` from `defaults`.
 - `lang` from the `sendMail` options
 - `lang` from the data of `sendMail` options
 - `lang` from the current recipient (`recipient.data.lang`)

The `__` and `lang` values are passed to the template.

#### `task()`

If you can send a lot of messages it is better to use the Redis queue rather than the memory.

To use a `rateLimit` you will need to boot a transport with the `pool: true` option. Learn more at [Delivering bulk mail](https://nodemailer.com/usage/bulk-mail/) and [Pooled SMTP](https://nodemailer.com/smtp/pooled/)

Make sure to run the task before sending any email, just after the initialization looks correct.

`task` returns a promise that should **not** be waited.

**Usage**

```js
task();
```

> **ProTip**: You can disable the queue for all email sends by setting `{ defaults: { queue: false } }` to initialization.

### Templating

These methods allow you to use your [MJML](https://mjml.io/) templates, coupled with [ejs](http://ejs.co/) for replacing variables and loops, among others.

Only `render` and `compile` methods add `__` method in the data for use the translations in the templates, the labels are found with the `templateName` argument.
You can pass your own translation method or overload the existing one with the data.

All the methods below return an HTML string.

The `opts` argument corresponds to the EJS argument described [here](http://ejs.co/#docs).

#### `render( templateName [, data = {}, opts = {}] )`

#### `renderFromFile( filePath [, data = {}, opts = {}] )`

#### `renderFromMjml( mjml [, data = {}, opts = {}] )`

#### `compile( templateName [, opts = {}] )`

#### `compileFromFile( filePath [, opts = {}] )`

#### `compileFromMjml( mjml [, opts = {}] )`

## Testing

### Running the tests

For a single run of all suites of tests:

```
yarn test
```
You can add the `--watch` option to watch the tests related to the files you modify, or `--watchAll` to run all tests with each change.

`--coverage` option is available to indicates that test coverage information should be collected and reported in the output.

These options are the most common, but you can use other [Jest CLI options](https://jestjs.io/docs/en/cli.html).

### Adding tests

If you want to create your own tests, you can refer to the [Testing SMTP](https://nodemailer.com/smtp/testing) section on the nodemailer documentation.