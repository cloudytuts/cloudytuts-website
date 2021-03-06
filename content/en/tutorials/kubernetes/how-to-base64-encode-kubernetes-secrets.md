---
title: "How to Base64 Encode Kubernetes Secrets"
date: 2020-08-12T21:01:26-04:00
draft: false
author: serainville
tags:
  - kubernetes
description: |
    Learn how to encode and decode Kubernetes secrets using the base64 command in Linux and OSX.
---

Kubernetes secrets allow us to segregate our secret and sensitive information from our resources. Instead of storing the data as clear text inside of, for example, a Pod manifest we can add a place holder that is replaced by Kubernetes when the Pod is created.

Kubernetes stores secrets as base64 encoded strings and encrypts the data on disk. In order to save a secret in Kubernetes it must be converted to a base64 string.

As an example, if the following string were the password for our database
```text
super-secret-password
```

It would look like the following when base64 encoded:
```text
c3VwZXItc2VjcmV0LXBhc3N3b3Jk
```

## Secrets Manifest
Secrets are populated in Kubernetes using Secrets resources. The following is an example of a Secrets manifest.

```yaml
apiVersion: v1
kind: Secrets
metadata:
  name: example-secrets
data:
  DB_PASSWORD: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
  DB_USER: ZGVtby1hcHAx
```

Notice the values for `DB_PASSWORD` and `DB_USER`. The values are actually base64 encoded strings, which is how Kubernetes stores secrets in its database. When creating a Secrets manifest you must base64 encode your string values.

## Base64 Encode Secrets
Base64 encoding a string in **OSX** and **Linux** can be done from the shell. Both operating systems typically come bundled with the `base64` command-line tool.

In order to convert a string into a valid base64 encoded string using the base64 command, we echo the string and pipe the output to the base64 command. The `-n` flag set for `echo` ensures only the characters within the commas will be encoded.

```shell
echo -n 'super-secret-password' | base64
```
    | Output
```shell
c3VwZXItc2VjcmV0LXBhc3N3b3Jk
```

{{< warning >}}
Always remember to set the `-n` flag for `echo` when encoding secrets. The flag prevents trailing newline characters from being encoded. Hidden characters in base64 encoded secrets will result in improperly formed strings, which will cause you grief with Kubernetes.
{{< /warning >}}

## Using stringData
For those who wish not to encode values as base64 encoded string first, an alternative is to use the `stringData` key instead of the `data` key in your manifest. The `stringData` key allows us to store our secrets as plain text in the file.

```yaml
apiVersion: v1
kind: Secrets
metadata:
  name: example-secrets
stringData:
  DB_PASSWORD: "super-secret-password"
  DB_USER: "demo-app1"
```

## Base64 Encoding Files
The base64 command on OSX and Linux is capable of more than just encoding strings. We can also encode the contents of an entire file, which is useful when we need to encode a configuration file or a certificate for a Kubernetes service.

Files are encoded using the `--input` flag with `base64`.

```shell
base64 --input cert.pem
```

## Decode Base64 Encodings
So far we've discussed base64 encoding your strings, but there are times when we need to do the reverse. You may need to decode a base64 string just to verify it was encoded correctly.

The base64 command has a `--decode` flag that will decode a given string.

```shell
echo -n 'c3VwZXItc2VjcmV0LXBhc3N3b3Jk' | base64 --decode
```

If you are decoding a file you can set the `-o <output-file>` flag. This is more useful when we are decoding a value that is known to be a file, such as a PEM file.

```shell
echo -n 'yBQYWdlIENvbnRlbnQgVHlwZXMKQ2xvdWR5VHV0cyBwb3N0cyBmb2xsb3dzIHNldmVyYWwgdHlwZXMgb2YgcGFnZSBjb250ZW50LgoqIFR1dG9yaWFscwoqIEd1aWRlcwoqIE5ld3MKKiBBdXRob3JzCgpUaGUgaW50ZW50aW9uIG9mIHRoaXMgZ3VpZGUgaXMgdG8gaGVscCBuZXcgY29udHJpYnV0b3JzIHN1Ym1pdCBuZXcgY29udGVudCB0aGF0IG1hdGNoZXMgQ2xvdWR5VHV0cyBleHBlY3RhdGlvbnMuCgojIyBUdXRvcmlhbHMKQSB0dXRvcmlhbCBpcyBhIHBvc3Qgd2l0aCBhIGxpbWl0ZWQgc2NvcGUgdGh' | base64 --decode -o test.md
```


