# Inkdb

An authenticated, realtime key-value storage synced across devices.

Encrypted by default with a shared key, can be zero-knowledge encrypted by the user.

Defaults to two data connections: 
    - site (common to all users, read-only),
    - user (private, read-write).

Supports multiple backends (firebase, etc)