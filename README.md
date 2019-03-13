# inkdb

An authenticated, realtime, key-value microstorage that's synced across devices.

Backend-agnostic.  Supports multiple backends (firebase, dynamodb, hasura/postgres, etc).

Provides two data enpoints:
- `inkdb.common` Anonymous, public, read-only.
- `inkdb.user` Authenticated, encrypted, read-write.  Can be zero-knowledge encrypted.


### Stupid simple syntax.

```js
inkdb.common.foo
"bar"

inkdb.common.ab_tests.button_color
"#454545"
```

```js
inkdb.user.bar
{'error': 'User not authenticated'}

// Authenticates with server.  If user encryption is server-managed, securely fetches user's encryption key, and decrypts.
inkdb.user.authenticate('myservermanagedusername', 'mypassword')

inkdb.user

{
    "foo": "Hi, I'm data."
}

inkdb.user.bar.set("ack");

inkdb.user

{
    "foo": "Hi, I'm data.",
    "bar": "ack",
}
```


`inkdb.user.register('mynewusername', 'mypassword')`


```js 
// Zero-knowledge encryption.

// Authenticates with a server.  If user encryption user-managed, simply fetches the encrypted data.
inkdb.user.authenticate('myservermanagedusername', 'mypassword')

inkdb.user.bar
{"error": "Zero-knowledge encryption enabled, and user vault is not decrypted."}

// Decrypt zero-knowledge encrypted data
inkdb.user.decrypt(prompt("What is your vault encryption key"?))

inkdb.user.bar
'ack'

// Enable zero-knowledge
inkdb.user.enable_zero_knowledge_encryption('myzeroknowledgeencryptionkey')

// Disable zero-knowledge
inkdb.user.disable_zero_knowledge_encryption('myzeroknowledgeencryptionkey')
```




## Data flow and security

Given:
1. The server has a public/private key pair.
2. The client is given the server's public key on page load.

Create:
1. Create account on server for user from username/pass
2. Generate pub/priv pair for user on server.  Encrypt with server's public key, store in db.
3. Return pub/priv via SSL.
4. Store pub/priv in browser local storage

Data access/reads and writes:
1. Access user data on data store at /publickey
2. Bind data to local var
3. Watch var for changes.
4. On changes (and first run), recursively decrypt the tree with the user's private key.
5. On user data add, save to local sync'd object, see in watch, encrypt with the public key and put into data store.


Using a second device/logging in:
1. Authenticate with the server over SSL.
2. Recieve pub/priv pair via SSL.
3. Follow 4-8 above.

Zero-knowledge path:
1. Follow 1-3 from the above.
2. Generate local zero-knowledge zeropub/zeropriv
3. Ask the user to provide a zero-knowledge password.
4. Encrypt local zeropub/zeropriv pair using the zero-knowledge-pass.
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

In the database/and in database backups:
- Each client's default public/private pair, encrypted with the server's private key
- Zero-knowledge clients' zeropub/zeropair, encrypted with a passphrase that the server doesn't know.

In a regular client's browser memory:
- The server's public key
- The client's default public/private pair

In a regular client's local storage:
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
- https://www.w3.org/TR/WebCryptoAPI/
- https://caniuse.com/#search=web%20crypto
- https://github.com/diafygi/webcrypto-examples
- https://blog.engelke.com/2015/02/14/deriving-keys-from-passwords-with-webcrypto/
- https://github.com/brix/crypto-js
- https://github.com/melanke/Watch.JS/
