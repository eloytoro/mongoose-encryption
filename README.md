_This is a fork from [mongoose-encryption](https://github.com/joegoldbeck/mongoose-encryption)_

mongoose-encryption
==================
Simple encryption and authentication for mongoose documents. Relies on the Node `crypto` module. Encryption and decryption happen transparently during save and find. Rather than encrypting fields individually, this plugin takes advantage of the BSON nature of mongoDB documents to encrypt multiple fields at once.


## How it Works

Encryption is performed using `AES-256-CBC` with a random, unique initialization vector for each operation. Authentication is performed using `HMAC-SHA-512`.

To encrypt, the relevant fields are removed from the document, converted to JSON, enciphered in `Buffer` format with the IV and plugin version prepended, and inserted into the `_ct` field of the document. Mongoose converts the `_ct` field to `Binary` when sending to mongo.
However this version of mongoose-encryption uses a different key for each document, virtually stored in the `_key` property, if not present the document cannot be decrypted, **it must be defined programmatically using the methods described below**

To decrypt, the `_ct` field is deciphered, the JSON is parsed, and the individual fields are inserted back into the document as their original data types.

To sign, the relevant fields (which necessarily include `_id` and `_ct`)  are stably stringified and signed along with the list of signed fields, the collection name, and the plugin version. This signature is stored in `Buffer` format in the `_ac` field with the plugin version prepended and the list of signed fields appended. Mongoose converts the field to `Binary` when sending to mongo.

To authenticate, a signature is generated in the same fashion as above, and compared to the `_ac` field on the document. If the signatures are equal, authentication succeeds. If they are not, or if `_ac` is missing from the document, authentication fails and an error is passed to the callback.

During `save`, documents are encrypted and then signed. During `find`, documents are authenticated and then decrypted

## Installation

`npm install eloytoro/mongoose-encryption`


## Usage
Generate and store keys separately. They should probably live in environment variables, but be sure not to lose them. You can either use a single `secret` string of any length; or a pair of base64 strings (a 32-byte `encryptionKey` and a 64-byte `signingKey`).

A great way to securely generate this pair of keys is `openssl rand -base64 32; openssl rand -base64 64;`

### Basic

By default, all fields are encrypted except for `_id`, `__v`, and fields with indexes

```javascript
//user.js

var mongoose = require('mongoose');
var encrypt = require('mongoose-encryption');

var userSchema = new mongoose.Schema({
    name: String,
    age: { type: Number, encrypt: true }
    // whatever else
});

// Add any other plugins or middleware here. For example, middleware for hashing passwords

userSchema.plugin(encrypt);
// This adds _ct and _ac fields to the schema, as well as pre 'init' and pre 'save' middleware,
// and encrypt, decrypt, sign, and authenticate instance methods

User = mongoose.model('User', userSchema);

// config.js

encryption.config({
    signingKey: process.env.SOME_64BYTE_BASE64_STRING
});
```

And you're all set. You should be able to `find` and make `New` documents as normal, but you should not use the `lean` option on a `find` if you want the document to be authenticated. `findOne`, `findById`, etc..., as well as `save` and `create` also all work as normal. `update` will work fine on unencrypted and unauthenticated fields, but will not work correctly if encrypted or authenticated fields are involved.

But in order for the document to be decrypted you must use the `registerKey(base64Key)` method, which will decrypt the document using the given key, after doing so you dont have to worry about the document's key anymore.


### Authenticate Additional Fields
By default, the encrypted parts of documents are authenticated along with the `_id` to prevent copy/paste attacks by an attacker with database write access. If you use one of the above options such that only part of your document is encrypted, you might want to authenticate the fields kept in cleartext to prevent tampering. In particular, consider authenticating any fields used for authorization, such as `email`, `isAdmin`, or `password` (though password should probably be in the encrypted block). You can do this with the `additionalAuthenticatedFields` option.
```
// keep isAdmin in clear but pass error on find() if tampered with
userSchema.plugin(encrypt, {
    encryptionKey: encKey,
    signingKey: sigKey,
    additionalAuthenticatedFields: ['isAdmin']
});
```
Note that the most secure choice is to include all non-encrypted fields for authentication, as this prevents tampering with any part of the document.

### Renaming an Encrypted Collection

To guard against cross-collection attacks, the collection name is included in the signed block. This means that if you simply change the name of a collection in Mongo (and therefore update the model name in Mongoose), authentication would fail. To restore functionality, pass in the `collectionId` option with the old model name.
```
// used to be the `users` collection, now it's `powerusers`
poweruserSchema.plugin(encrypt, {
    collectionId: `User` // this corresponds to the old model name
});

PowerUser = mongoose.model('PowerUser', poweruserSchema);
```

### Save Behavior

By default, documents are decrypted after they are saved to the database, so that you can continue to work with them transparently.
```
joe = new User ({ name: 'Joe', age: 42 });
joe.save(function(err){ // encrypted when sent to the database
                        // decrypted in the callback
  console.log(joe.name); // Joe
  console.log(joe.age); // 42
  console.log(joe._ct); // undefined
});

```
You can turn off this behavior, and slightly improve performance, using the `decryptPostSave` option.
```
userSchema.plugin(encrypt, { ..., decryptPostSave: false });
...
joe = new User ({ name: 'Joe', age: 42 });
joe.save(function(err){
  console.log(joe.name); // undefined
  console.log(joe.age); // undefined
  console.log(joe._ct); // <Buffer 61 41 55 62 33 ...
});
```

### Changing Options
For the most part, you can seemlessly update the plugin options. This won't immediately change what is stored in the database, but it will change how documents are saved moving forwards.

However, you cannot change the following options once you've started using them for a collection:
- `secret`
- `collectionId`

### Instance Methods

You can also encrypt, decrypt, sign, and authenticate documents at will (as long as the model includes the plugin). `decrypt`, `sign`, and `authenticate` are all idempotent. `encrypt` is not.

```
joe = new User ({ name: 'Joe', age: 42 });
joe.encrypt(function(err){
  if (err) { return handleError(err); }
  console.log(joe.name); // undefined
  console.log(joe.age); // undefined
  console.log(joe._ct); // <Buffer 61 41 55 62 33 ...

  joe.decrypt(function(err){
    if (err) { return handleError(err); }
    console.log(joe.name); // Joe
    console.log(joe.age); // 42
    console.log(joe._ct); // undefined
  });
});

```

```
joe.age = 30

joe.sign(function(err){
  if (err) { return handleError(err); }
  console.log(joe.name); // Joe
  console.log(joe.age); // 30
  console.log(joe._ac); // <Buffer 61 fa 63 95 50

  joe.authenticate(function(err){
    if (err) { return handleError(err); }
    console.log(joe.name); // Joe
    console.log(joe.age); // 30
    console.log(joe._ac); // <Buffer 61 fa 63 95 50

    joe.age = 22

    joe.authenticate(function(err){ // authenticate without signing changes, error is passed to callback
    	if (err) { return handleError(err); } // this conditional is executed
    	console.log(joe.name); // this won't execute
  });
});
```
There are also `decryptSync` and `authenticateSync` functions, which execute synchronously and throw if an error is hit.

## They keygen method
This version also provides a way to generate secure keys for each document, `this.keygen(secret)` will generate a HMAC-512 base64 string from the given secret(which could very safely be the user's password). Later on using `this.registerKey(base64Key)` will succesfully decrypt this document.
**DISCLAIMER:** do **not** store the generated key in your databse, best practice is to give it to the front-end user, pherhaps via token authentication.

## Getting Started with an Existing Collection
If you are using mongoose-encryption on an empty collection, you can immediately begin to use it as above. To use it on an existing collection, you'll need to either run a migration or use less secure options.

### The Secure Way
To prevent tampering of the documents, each document is required by default to have a signature upon `find`. The class method `migrateToA()` encrypts and signs all documents in the collection. This should go without saying, but **backup your database** before running the migration below.
```
userSchema.plugin(encrypt.migrations, { .... });
User = mongoose.model('User', userSchema);
User.migrateToA(function(err){
    if (err){ throw err; }
    console.log('Migration successful');
});
```
Following the migration, you can use the plugin as above.

### The Quick Way
You can also start using the plugin on an existing collection without a migration, by allowing authentication to succeed on documents unsigned documents. This is less secure, but you can always switch to the more secure options later.
```
userSchema.plugin(encrypt, { requireAuthenticationCode: false, .... });
```
