Public Key Cryptography in the Browser
======================================

This web page with associated JavaScript creates a random RSA
key pair, and allows a user to:

* Encrypt local files with the public key, and save the encrypted version
* Decrypt previously encrypted files with the matching private key, and save the decrypted version

This example is provided to illustrate how to use the W3C
[Web Cryptography API](http://www.w3.org/TR/WebCryptoAPI/ "API Draft")
to perform public key encryption inside a web browser. It is based
on the [working draft](http://www.w3.org/TR/2014/WD-WebCryptoAPI-20140325/ "Dated Working Draft")
of the standard available when this example was created.

Using this example requires a web browser that implements a compatible version
of the Web Cryptography API. When the example was created, current versions of
the Google Chrome browser with the optional "Enable Experimental Web Platform
features" flag enabled and recent nightly builds of the Firefox browser could
run the example.

This example uses RSA-OAEP to encrypt a random 128-bit AES key, then uses
AES in CBC mode with the random key and a 16 byte random initialization
vector. The encrypted AES key, random initialization vector, and AES-CBC
encrypted results are packaged into a single encrypted file.
Modifying it to use a different supported public-key algorithm, or
different size AES keys or different AES modes would be simple for a developer.

**This is not intended to be a production tool.** Rather, it may
be helpful to developers who intend to create their own tools using
the Web Cryptography API.

Copyright (c) 2014 Info Tech, Inc.
Provided under the MIT license.
See LICENSE file for details.
