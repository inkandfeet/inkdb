# inkdb

An authenticated, realtime, offline-first microstorage that's synced across devices.

Built-in support for most common data-sharing patterns, that makes securely managing your user's and app's data simple.

Provides multiple data patterns, built-in:

- `inkdb.public` Anonymous, public, read-only.
- `inkdb.user.app.readonly` User-specific data the app backend has read-write access to. User read-only. Authenticated, encrypted.
- `inkdb.user.app.readwrite`  User-specific data the app backend has read-write access to. User read-write. Authenticated, encrypted.  (Needs new name?)
- `inkdb.user.private` User-specific, authenticated, encrypted, read-write data the app backend promises not to access.  Can be zero-knowledge encrypted to make sure that holds true.
- `inkdb.device_only` Encrypted, read-write storage that never leaves the device. Unlike the above, it is never sync'd.


Key features:
- Everything encrypted.  Use whatever backend you'd like.  Your user's data is encrypted.
- ~Agnostic to your realtime backend.  Supports multiple backends (Kinto, Firestore, DynamoDB, Hasura/Postgres, etc).~
- Built on Kinto.
- Bundled authentication backend is Python/Django, but you can override with your own, and it'd be simple to port this to Rails, etc.


### Stupid simple syntax.


#### Get
```js
inkdb.public.foo
"bar"

inkdb.public.theme.daily_button_color
"#454545"

> inkdb.user.app.readwrite.foo
{'error': 'User not authenticated'}

// Authenticates with server.  If user encryption is server-managed,
// securely fetches user's encryption key, and decrypts.
> inkdb.authenticate('myusername', 'mypassword')

> inkdb.user.app.readwrite.foo

{
    "foo": "Hi, I'm data."
}


> inkdb.user.app.readonly.purchases

{
    "product-id-1": {
        purchased_on: 1928312981023,
        title: "My Great Product",
        url: "https://mycompany.com/products/great-product-1",
    }
}


```

#### Set

```js
> inkdb.public.foo = "I want to override this"
{'error': 'Public is read-only.'}

> inkdb.public.foo
"bar"

> inkdb.user.app.readwrite.bar = "ack";
{'error': 'User not authenticated'}

> inkdb.authenticate('myusername', 'mypassword')
> inkdb.user.app.readwrite.bar = "ack";

> inkdb.user.app.readwrite
{
    "foo": "Hi, I'm data.",
    "bar": "ack",
}

> inkdb.user.app.readonly.purchases.product2.purchased = True
{'error': 'Vault is read-only'}

> inkdb.user.app.readwrite.favorite_color = "#454545";

> inkdb.user.app.readwrite.favorite_color
"#454545";


```

#### User Registration

```js
inkdb.register('mynewusername', 'mynewpassword')
inkdb.authenticate('mynewusername', 'mynewpassword')
```

#### Change Password

```js
inkdb.user.change_password('mynewpassword')  // Only works if authenticated.
```

#### Reset Password

```js
inkdb.reset_password('mynewpassword')  
// Starts email-based password recovery flow.  Won't help with zero-knowledge encrypted private data.
```



#### Zero-knowledge encryption.
```js 
// Authenticates with a server.  If user encryption user-managed, simply fetches the encrypted data.
> inkdb.authenticate('myusername', 'mypassword')

> inkdb.private.bar
{"error": "Zero-knowledge encryption enabled, and user vault is not decrypted."}

// Decrypt zero-knowledge encrypted data
> inkdb.private.decrypt(prompt("What is your vault encryption key"?))

> inkdb.private.bar
'ack'

// Enable zero-knowledge
> inkdb.private.enable_zero_knowledge_encryption('myzeroknowledgeencryptionkey')

// Disable zero-knowledge
> inkdb.private.disable_zero_knowledge_encryption('myzeroknowledgeencryptionkey')
```




## Data flow and security

### Open Questions

- Do we need to even do pub/private pairs?  We only have one real party who needs access, symmetric encryption might be a better fit.
- Is it worth bothering to use the server's key pair to verify, since we implicitly trust SSL?  Protection against heartbleed-type attacks, sure.  But worth the cost?


### Data flow

Given:
- The server has a secure public/private key pair.
- The client is passed the server's public key via SSL

Account Creation:
1. Create account on server for user from username/pass
2. Generate pub/priv pair for user on server.  Encrypt with server's private key, store in db.
3. Return pub/priv via SSL.  Verify using server's public key (needed since we trust SSL?).
4. Store pub/priv in browser local storage

Using a second device/logging in:
1. Authenticate with the server over SSL.
2. Recieve pub/priv pair via SSL and session token (that stays for the time you set.)
3. Follow 1-5 above.

Data access/reads and writes:
1. Access user data on data store at /publickey
2. Bind data to local var
3. Watch var for changes.
4. On changes (and first run), recursively decrypt the tree with the user's private key.
5. On user data add, save to local sync'd object, see in watch, encrypt with the public key and put into data store.

Zero-knowledge path:
1. Follow 1-4 from the standard account list.
2. Generate local zero-knowledge zeropub/zeropriv
3. Ask the user to provide a zero-knowledge password.
4. Encrypt local zeropub/zeropriv pair using the PBKDF(zero-knowledge-pass).
5. Send encrypted zeropair to server, save.
6. (Optional) Save working zero-knowledge pair into localstorage (If user chooses when entering password. Default to not save.)
7. Use the zero-knowledge pair instead of passed pair to encrypt/decrypt.


Zero-knowledge second device:
1. Authenticate with the server over SSL.
2. Recieve pub/priv and zeropub/zeropriv over SSL.
3. Provide zeropub/zeropriv decryption key.
4. Unlock zeropub/zeropriv keys.
5. Follow 5-6 above.

Zero-knowledge upgrade:
1. Use current pub/priv pair to recursively decrypt data
2. Use new zeropub/zeropriv pair to recursively encrypt data
3. Force-write encrypted data to root node.




### As a check to the above, the following data is available.  Is this ok?

In the server's environment:
- The server's public/private keys

In the database and in database backups:
- Each client's default public/private pair, encrypted with the server's private key
- Zero-knowledge clients' zeropub/zeropair, encrypted with a passphrase that the server doesn't know.

In a regular client's browser memory:
- The server's public key
- The client's default public/private pair

In a regular client's local storage:
- A session token
- The client's default public/private pair

In a zero-knowledge client's browser memory:
- The server's public key
- The client's default public/private pair
- The client's zeropub/zeropriv

In a zero-knowledge client's local storage:
- Nothing

In transit, over TLS/SSL:
- The client's username and password
- The server's public key
- The client's default public/private pair
- The client's encrypted zeropub/zeropriv

In transit, over insecured HTTP:
- Nothing

In transit between server and database, over TLS/SSL:
- Each client's default public/private pair, encrypted with the server's private key
- Zero-knowledge clients' zeropub/zeropair, encrypted with a passphrase that the server doesn't know.



Tech notes:
- https://kinto.readthedocs.io/en/latest/tutorials/client-side-encryption.html
- https://michielbdejong.github.io/kinto-encryption-example/
- https://github.com/Kinto/kinto-http.py

- https://www.w3.org/TR/WebCryptoAPI/
- https://caniuse.com/#search=web%20crypto
- https://github.com/diafygi/webcrypto-examples
- https://blog.engelke.com/2015/02/14/deriving-keys-from-passwords-with-webcrypto/
- https://github.com/brix/crypto-js
- https://github.com/melanke/Watch.JS/

