---
title: "Github GPG Keys"
date: 2023-06-27T17:31:27+08:00
Description: ""
Tags: [github, gpg, git, signature, signingkey]
Categories: [GitHub, GPG, Security, Git]
DisableComments: false
toc: false
---

## GitHub Web-flow GPG Key

GitHub sets the committer for all commits made using their web interface to the user [web-flow](https://github.com/web-flow).

For any given GitHub account, you can add `.gpg` to its URL to get its public key — so for web-flow, you can find it at <https://github.com/web-flow.gpg>:

```text
-----BEGIN PGP PUBLIC KEY BLOCK-----

xsBNBFmUaEEBCACzXTDt6ZnyaVtueZASBzgnAmK13q9Urgch+sKYeIhdymjuMQta
x15OklctmrZtqre5kwPUosG3/B2/ikuPYElcHgGPL4uL5Em6S5C/oozfkYzhwRrT
SQzvYjsE4I34To4UdE9KA97wrQjGoz2Bx72WDLyWwctD3DKQtYeHXswXXtXwKfjQ
7Fy4+Bf5IPh76dA8NJ6UtjjLIDlKqdxLW4atHe6xWFaJ+XdLUtsAroZcXBeWDCPa
buXCDscJcLJRKZVc62gOZXXtPfoHqvUPp3nuLA4YjH9bphbrMWMf810Wxz9JTd3v
yWgGqNY0zbBqeZoGv+TuExlRHT8ASGFS9SVDABEBAAHNNUdpdEh1YiAod2ViLWZs
b3cgY29tbWl0IHNpZ25pbmcpIDxub3JlcGx5QGdpdGh1Yi5jb20+wsBiBBMBCAAW
BQJZlGhBCRBK7hj4Ov3rIwIbAwIZAQAAmQEIACATWFmi2oxlBh3wAsySNCNV4IPf
DDMeh6j80WT7cgoX7V7xqJOxrfrqPEthQ3hgHIm7b5MPQlUr2q+UPL22t/I+ESF6
9b0QWLFSMJbMSk+BXkvSjH9q8jAO0986/pShPV5DU2sMxnx4LfLfHNhTzjXKokws
+8ptJ8uhMNIDXfXuzkZHIxoXk3rNcjDN5c5X+sK8UBRH092BIJWCOfaQt7v7wig5
4Ra28pM9GbHKXVNxmdLpCFyzvyMuCmINYYADsC848QQFFwnd4EQnupo6QvhEVx1O
j7wDwvuH5dCrLuLwtwXaQh0onG4583p0LGms2Mf5F+Ick6o/4peOlBoZz48=
=HXDP
-----END PGP PUBLIC KEY BLOCK-----
```

You can then import and trust that public key.

As shown in this thread:

```sh
$ curl https://github.com/web-flow.gpg | gpg --import
$ gpg --edit-key noreply@github.com
gpg> trust
gpg> save
$ gpg --lsign-key noreply@github.com
```

Ref: <https://stackoverflow.com/questions/60482588/what-is-githubs-public-gpg-key>

## GitHub SSH Key Fingerprints

Ref: <https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints>

## Transfer GPG Keys

We will need to transfer the keys. Mac and Linux work the same, storing the keys in `~/.gnupg`.
