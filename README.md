# Inkdb

An authenticated, realtime, key-value microstorage that's synced across devices.

User data is encrypted by default with a server-managed key, and can be zero-knowledge encrypted by the user.

Site data is completely anonymous, and read-only.

Defaults to two data connections: 

- `inkdb.common` (anonymous, read-only,)
- `inkdb.user` (authenticated, private, read-write).

Stupid simple syntax.

```js
inkdb.common.foo
"bar"

inkdb.common.ab_tests.button_color
"#454545"
```

```js
inkdb.user.bar
{'error': 'User not authenticated'}
```


```js
inkdb.user.authenticate('myservermanagedusername', 'mypassword')
# Authenticates with server.  If user encryption is server-managed, securely fetches user's encryption key, and decrypts.
```


```js
inkdb.user

{
    'foo': 'Hi, I'm data.'
}

inkdb.user.bar.set('ack');

inkdb.user
{
    'foo': 'Hi, I'm data.',
    'bar': 'ack',
}
```




`inkdb.user.register('mynewusername', 'mypassword')`


```js 
# Zero-knowledge encryption.
inkdb.user.authenticate('myservermanagedusername', 'mypassword')

# Authenticates with a server.  If user encryption user-managed, simply fetches the encrypted data.
inkdb.user.bar
{"error": "Zero-knowledge encryption enabled, and user vault is not decrypted."}


# Decrypt zero-knowledge encrypted data
`inkdb.user.decrypt('myzeroknowledgeencryptionkey')
inkdb.user.bar
'ack'

# Enable zero-knowledge
`inkdb.user.enable_zero_knowledge_encryption('myzeroknowledgeencryptionkey')`

# Disable zero-knowledge
`inkdb.user.disable_zero_knowledge_encryption('myzeroknowledgeencryptionkey')
```

Backend-agnostic.  Supports multiple backends (django-channels, firebase, dynamodb, hasura/postgres, etc).


Tech-spec:

Authentication:
- Python django app

Realtime data sync
 and encryption:
- username/password used to generate a public/private key pair.
- 

Tech notes:
https://www.w3.org/TR/WebCryptoAPI/
https://caniuse.com/#search=web%20crypto
https://github.com/diafygi/webcrypto-examples
https://blog.engelke.com/2015/02/14/deriving-keys-from-passwords-with-webcrypto/
