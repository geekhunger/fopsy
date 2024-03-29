# Readme

> **IMPORTANT:** Changes since version 1.0.0
>
> - Fopsy now an ESM module! (Breaking change!)
> - New methods: `mkgitignore`, `mkscript`, `runscript`

# synchronous file operations

Lightweight wrappers around **synchchronous** file operations in NodeJS.


CommonJS (before version 1.0.0):

```js
const fopsy = require("fopsy") // all exports
const {exec} = require("fopsy") // selected expors
```

ESM (since version 1.0.0):

```js
import fopsy from "fopsy" // all exports
import {mkfile, rmfile} from "fopsy" // selected exports
```


<br>

- **`mkfolder(path)`**

Create a folder recursevly at given `path`.


<br>

- **`mkfile(path, content, action = "w", mode = 0o744)`**

Create a file at given `path`. Any folders are created recursevly along the way.

The `action` argument sets the file interaction flag like `'a'` as in 'append'. (Default is `'w'` as in 'write'.) [See this list](https://nodejs.org/api/fs.html#file-system-flags) for all supported flags.

The `mode` argument sets the file permissions (`chmod`).


<br>


- **`mkscript(filepath, contents, environment)`**

Create a shell script file at given `filepath` with given `contents`.

The file gets only created if `environment` argument is either missing, or, if its value equals to `NODE_ENV`. Meaning, you can generate scripts based on the current Node environment.

For example, if your environment is `NODE_ENV=production` and you create your script like this `mkscript("/folder/file.sh", "echo 'test'", "development")` then the script will not be created because the script is meant to be created only for "development" environments.

This is useful if you have a preflight routine that generates scripts for your and you have multiple application running the same code. Based on the environment argument of the function, you could easily run the same code on "production" and "development" and only "production" would create the required file from the example above.


<br>


- **`mkgitignore(path, rules)`**

Generates a `.gitignore` file with given `rules`.

The neat thing about this function is that if there already IS a gitignore file, then it will extend it with given `rules`, without duplicated entries!

Since the contents of the '.gitignore' file are auto-generated, the function call will also automatically untrack itself from Git, by adding a `.gitignore` rule to the generated file. - This does not apply if a gitignore file already existed and had a `*` (ignore all) rule, because in this case everything is untracked anyways.


<br>

- **`rmfolder(path)` and `rmfile(path)`**

Delete a folder or a file recursevly at given `path`.


<br>

- **`catfolder(path, encoding)`**

Read file(s) at given `path` (`path` can be a string or an array of strings).

The return value is an array of objects. Each object represents one file that has been read. For example:

```js
[
    {
        content: <Buffer 68 65 6c 6c 6f 20 77 6f 72 6c 64 0a>,
        encoding: undefined,
        mime: 'text/plain',
        size: {
            value: 12,
            byte: [Getter/Setter],
            kilobyte: [Getter/Setter],
            megabyte: [Getter/Setter],
            gigabyte: [Getter/Setter]
        }
    }
]
```

Without an `encoding` argument the `content` property will be a `Buffer`! - With an `encoding` argument, the `content` will become a string (encoded by `encoding` format) and the `encoding` property will be present. You can then use the `content` string as you like, or use buffers yourself to convert it into a different format. [(See this documentation on encoding formats)](https://nodejs.org/docs/latest/api/buffer.html#buffers-and-character-encodings)

The `size` propery is an object with getters and setters and helps converting the file size into different units, like byte into megabyte and vice versa.

> NOTE: If a file is missing then its `content` will equal `null` and its `size` will equal `0`!


<br>

- **`catfile(path, encoding)`**

Read file(s) at given `path`. - Technically, this function is just an alias to `catfolder()`, *but* instead of always returning an array of fetched files, this function returns an array only when `path` is passed as an array too! (If `path` is a string but it still points to a directory (potentially with many files in it), then the return value will also remain an array!) - Only when `path` is a string that points to a file, this function returns a single object. - So, this is basically a convenience function wrapper for `.catfile()`.


<br>

- **`sizeunit(byte)`**

This is a helper... but it is available too!

Calling it with an argument will set the default value in byte! For example: `sizeunit(16)` - From there on, you can convert the value into `byte`, `kilobyte`, `megabyte` or `gigabyte`, for example: `console.log( sizeunit(16).megabyte )`

You can also directly set a value in `megabyte` if you want, and then convert it back into `byte`. For example:

```js
const mb = sizeunit()
console.log(mb) // default byte value is 0
mb.megabyte = 3 // setter will convert the value into byte internally
console.log(mb.byte)
console.log(mb.megabyte)
```


<br>

- **`exec(command [, options])`**

This is a very light wrapper around the NodeJS [execSync](https://nodejs.org/api/child_process.html#child_processexecsynccommand-options) function and it lets you run shell commands. The function call accepts the exact same arguments as the original function, but it has a purpose and some additional value...

`exec` is able catch all of the errors and outputs from STDOUT, that your `command` produces during execution! The return value is an object with a boolean `success` (true, false) and a string `stdout` (the response received from STDOUT).

This way you can tell check if the command execution was sucessfully or not. And we can also parse the response of the command and react to any events accordingly.

Here are some use-case examples:

```js
import {exec} from "fopsy"

const response = exec(`certbot certonly --webroot --non-interactive --agree-tos -m ${EMAIL} -w '${CHALLENGE_SAVEDIR}' -d ${DOMAIN} --work-dir '${ROOTPATH}' --deploy-hook '${RENEW_CALLBACK}'`)

if(response.success) {
    console.info(`Certbot requested a certificate renewal for '${DOMAIN}'. Server will be reloaded after the cert update.`)
    console.log(response.stdout)
} else {
    console.error(response.stdout)
    console.warn(`Certbot could not auto-renew the SSL certificate for '${DOMAIN}'! Please inspect the server logs and resolve this issues manually!`)
}
```

As you've noticed, the `exec` will not throw any errors or break your application. But sometimes `throw new Error("command execution failed")` is exactly what you need:

```js
const response = exec(`certbot certonly...`)
if(!response.success) throw new Error("command failed with: ${response.stdout}")
```

If you want a more elegant and flexible way of asserting, feel free to use the `assert` function from my other package on NPM, called [type-approve](https://www.npmjs.com/package/type-approve)!

```js
import {assert} from "type-approve"
import {exec} from "fopsy"
const run = cmd => assert(...Object.values(exec(cmd))) // just a custom shortcut to calling assert with condition and message arguments

try {
    const response = run(`certbot certonly...`)
    console.log("Success: " + response.stdout)
} catch(error) {
    console.error(error)
}
```


<br>


- **`runscript(cmd)`**

Runs a command line call, for example: `runscript("echo 'hello world'")`. The great thing is that the call is running through an `assert` call, meaning it will throw an error if the script exists with an stderr.

Basically the implimentation is as simple as `assert(...Object.values(exec(cmd)))`. Just a light wrapper around `exec()`.
