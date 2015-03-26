#Public Key Cryptography in the Browser

It's been a month since I published posts on <a title="Intro post on symmetric cryptography in browser" href="http://blog.engelke.com/2014/06/19/exploring-the-new-web-cryptography-api/" target="_blank">how to</a> <a title="Part 2 of symmetric cryptography series" href="http://blog.engelke.com/2014/07/05/symmetric-cryptography-in-the-browser-part-2/" target="_blank">perform</a> <a title="Part 3 of symmetric cryptography series" href="http://blog.engelke.com/2014/07/13/symmetric-cryptography-in-the-browser-part-3/" target="_blank">symmetric</a> <a title="Conclusion of asymmetric cryptography series" href="http://blog.engelke.com/2014/07/16/symmetric-cryptography-in-the-browser-conclusion/" target="_blank">cryptography</a> in web browsers using the <a title="Draft specification" href="http://www.w3.org/TR/WebCryptoAPI/" target="_blank">Web Cryptography API</a>. Now we'll move on to performing asymmetric, or public-key, cryptography in the browser. The goal of this post is to write the simplest possible in-browser code to encrypt and decrypt files using public-key cryptography. The result will not be a useful tool at all, but should be a good first step toward a useful tool.

Even though this is just about the simplest possible example, _*this is still a very long post!*_ Sorry about that, but it will be clearer this way than spread out over several posts. The final example is available on <a title="Github repository for this example" href="https://github.com/infotechinc/public-key-encryption-in-browser" target="_blank">Github</a>, and a <a title="Live demonstration page" href="http://infotechinc.github.io/public-key-encryption-in-browser/" target="_blank">live demonstration page</a> is available, too.

#The API

The <a title="Draft specification" href="http://www.w3.org/TR/WebCryptoAPI/" target="_blank">Web Cryptography API</a> is available through the *window.crypto* object in the browser. Most of the functionality is in the *window.crypto.subtle* object. It is still under development but already widely available. Chrome browsers currently turn it off unless the flag _Enable Experimental Web Platform Features_ is set, but <a title="Google doc about Web Crypto API in Chromium" href="https://docs.google.com/document/d/184AgXzLAoUjQjrtNdbimceyXVYzrn3tGpf3xQGCN10g/edit" target="_blank">Google has announced</a> that it will be on by default starting with Chrome 37, due out in a few weeks. Firefox Nightly builds have it, and it's expected to move to the Aurora releases on September 2, then step by step to beta and general release. The Opera Developer release now supports it, but doesn't seem to support public-key algorithms yet and I don't when it will be complete and move to stable. I haven't been able to find any information on if and when Safari will support it. Internet Explorer 11 general release supports a prefixed version at *window.msCrypto*. Unfortunately, that's based on an earlier version of the API. The functionality is there, but the behavior of the methods is a bit different from what we will use here.

Those are all desktop browsers, but mobile versions of them seem to be working to implement this API, too. They are lagging a bit behind the desktops, but making progress.

Any of the browser versions supporting *window.crypto.subtle* should eventually be able to run the code in this post. It was developed with Chrome 36 and the web platform flag enabled, and tested in in Firefox Nightly. As of this writing, it does not yet work in Opera Developer.

#JavaScript Promises

<a title="HTML version of latest specification" href="https://people.mozilla.org/~jorendorff/es6-draft.html#sec-promise-objects" target="_blank">Promise objects</a> offer a way to perform asynchronous operations without using callbacks. They can't do anything more than callbacks can, but they can be more convenient to use, especially when you have to chain aynchronous operations together. The API returns a *Promise* for almost every operation.

A Promise has two important methods: *then* and *catch*. They each take a function as a parameter, and each return another Promise. If the operation of the Promise succeeds, the function provided to the *then* method is executed and passed the result of the operation as its sole parameter. If it fails or has an exception, the function provided to the *catch* method is run and usually provided with an Error object as its sole parameter.

The result of the then or catch method is another promise, which provides its return value as the parameter to that new Promise's then handler (or Error object to the catch handler). Finally, instead of separate then and catch methods, you can just pass two functions to the then handler which will execute the first one on success, or the second one on failure.

The code in this post often chains Promises together to force asynchronous operations to happen sequentially. Once you get comfortable with them, you'll likely find this pretty clear to follow.

#About Public-Key Cryptography

Symmetric cryptography, as used in the earlier blog posts, uses a single secret key to encrypt and decrypt data. Which means that any two parties (call them Alice and Bob) that want to communicate securely have to first find a secure way to share that key. And if three or more parties want to communicate securely they either have to accept that everyone in the group can read everything, or create and share separate keys for every pair of people in the group. And a secret shared by a group isn't likely to remain secret. As Ben Franklin said, <a title="Benjamin Franklin quote" href="http://www.brainyquote.com/quotes/quotes/b/benjaminfr162078.html" target="_blank">three can keep a secret if two of them are dead</a>.

Public key cryptography solves these problems, though it adds complexity to do so. It can be hard to get your head around it, which isn't surprising considering that symmetric cryptography has been around for millennia but public key cryptography wasn't <a title="Wikipedia article on the history of PKC" href="http://en.wikipedia.org/wiki/Public-key_cryptography#History" target="_blank">invented until the 1970s</a>. The core idea is that each party owns a pair of keys, called the _public_ and _private_ keys, that are related in such a way that anything encrypted with one of those keys can only be decrypted with the other one. The difference between the keys in a key pair is just that one is arbitrarily called public and shared with anyone at all, while the other one is called private and kept under the sole control of the key pair owner.

So even if Alice and Bob have never met or had a secure way to communicate before, they can use public key cryptography to securely share information. Alice can send Bob a secret message by encrypting it with Bob's public key, and then only Bob can decrypt it because he has sole control of his private key. And if you have a large group, each member would have a separate key pair and anyone could communicate securely with any other specific member using that member's public key.

It's important to remember that this provides secure communication with the owner of a key pair, who may or may not be the actual person you think it is. Reliably authenticating the owner of a key pair is a separate problem that public key cryptography can help with, but it isn't addressed in this post.

#The Web Page

The page is just going to allow the user to select a file and then click a button to encrypt or decrypt it. Those operations will use a key pair that is randomly generated when the page is created. That's right. Once the page is closed, there's no way to decrypt anything it encrypted because the key pair is gone for good. Persisting, exporting, and importing keys are left out of this example.

The HTML is really simple:

    <!DOCTYPE html>
    <html>
    <head>
        <title>Public-Key Encryption</title>
        <script src="pkcrypto.js"></script>
    </head>
    <body>
        <h1>Public-Key Encryption</h1>
        <section id="encrypt-and-decrypt">
            <input type="file" id="source-file"/>
            <button id="encrypt">Encrypt File</button>
            <button id="decrypt">Decrypt File</button>
        </section>
        <section id="results">
            Download results:
            <ul id="download-links">
            </ul>
        </section>
    </body>
    </html>

The skeleton of the JavaScript is a lot like the symmetric cryptography code. The main difference is that the user doesn't control the key pair; it's automatically created when the page loads:

    document.addEventListener("DOMContentLoaded", function() {
        "use strict";

        if (!window.crypto || !window.crypto.subtle) {
            alert("Your current browser does not support the Web Cryptography API! This page will not work.");
            return;
        }

        var keyPair;
        createAndSaveAKeyPair().
        then(function() {
            // Only enable the cryptographic operation buttons if a key pair can be created
            document.getElementById("encrypt").addEventListener("click", encryptTheFile);
            document.getElementById("decrypt").addEventListener("click", decryptTheFile);
        }).
        catch(function(err) {
            alert("Could not create a keyPair or enable buttons: " + err.message);
        });

        // More code to come
    }

The *createAndSaveAKeyPair* function puts its result in the *keyPair* variable instead of returning it because it runs asynchronously. It returns before it has created the pair. However, the following *then* clause does not run until the *keyPair* has been successfully created and saved.

#Creating a Key Pair

We used the API's *generateKey* method to create a key for symmetric cryptography in the earlier posts, though any bit sequence of the right length would have worked. With public key cryptography you must use a sophisticated algorithm to create a key pair, because the keys must have necessary mathematical properties that anything encrypted with one key can only be decrypted with the other. It is also important that it must not be feasible to derive one member of the key pair from the other. The *generateKey* method can do all that. It's the same method signature as before:

    window.crypto.subtle.generateKey(algorithmIdentifier, extractableFlag, keyUsagesList).
    then(successHandler).
    catch(failureHandler);

So the first step in creating a key pair is to figure out the algorithm that will use it and the parameters to provide for it. The draft has a table of <a title="Registered algorithms table" href="http://www.w3.org/TR/WebCryptoAPI/#algorithms-index" target="_blank">registered algorithms</a> listed with their possible usages. Right now there are only two public key algorithms listed that can be used to encrypt and decrypt: RSAES-PKCS1-v1\_5 and RSA-OAEP, both specified in <a title="RFC 3447 specifying PKCS1 algorithms" href="http://www.ietf.org/rfc/rfc3447" target="_blank">RFC 3447</a>. Both use the RSA algorithm as their basis. RSA-OAEP seems to be the newer, preferred, choice, but it's not currently supported in Chrome, so this code uses RSAES-PKCS1-v1_5, which both Chrome and Firefox can already support.

The algorithmIdentifier object has to include the name of the algorithm and the <a title="RSA Key Generation parameters" href="http://www.w3.org/TR/WebCryptoAPI/#dfn-RsaKeyGenParams" target="_blank">RsaKeyGenParams</a> _modulusLength_ and _publicExponent_. The _modulusLength_ is generally known as the key length, and has to be much larger than an AES key of similar security. The most common choice at this time is 2048 bits for the modulusLength, considered secure enough but not too large to work with easily. The publicExponent is a different story, and picking a good choice requires a pretty deep understanding of the RSA algorithm. But good news: Chrome currently <a title="Google doc describing current limitations on Web Cryptography API implementation" href="https://docs.google.com/document/d/184AgXzLAoUjQjrtNdbimceyXVYzrn3tGpf3xQGCN10g/edit" target="_blank">only supports the values 3 and 65537</a> (2^16 + 1) for this, so we don't have to think much about it. We'll use 65537 (0x101).

The following code returns a Promise that generates one key pair and saves it in the variable *keyPair*. You can just call the function and expect that the value of keyPair will eventually be updated, or you can use the then clause of the returned Promise to run code that should only occur after the value has been updated. Since the function provided to the then method returns the keyPair, that value will be provided as the input parameter to the next then clause in a chain.

    function createAndSaveAKeyPair() {
        return window.crypto.subtle.generateKey(
            {
                name: "RSAES-PKCS1-v1_5",
                modulusLength: 2048,
                publicExponent: new Uint8Array([1, 0, 1])   // 24 bit representation of 65537
            },
            true,   // can extract it later if we want
            ["encrypt", "decrypt"]).
        then(function(key) {
            keyPair = key;
            return key;
        });
    }

If this works, *keyPair* will be an object with fields of type Key named _privateKey_ and _publicKey_. We're going to encrypt a file with the _publicKey_, and then later decrypt it with the matching _privateKey_.

#Encryption

This should be easy. The earlier series of posts encrypted and decrypted a file, so this should be pretty much the same, right? Unfortunately, wrong. The RSA algorithm can only encrypt data somewhat smaller than the key's modulusLength, which is only 2048 bits (256 bytes). That's pretty limiting. And even if it could encrypt more data, we wouldn't want to do it. RSA is extremely slow. *Really, really, slow.*

So how can you share a file secretly using public key cryptography? By creating a random symmetric key (say, a 128 bit AES key) and encrypting the file with that key. Then encrypt that key (known as the session key) using public key cryptography with the public key. Give the recipient the encrypted file and the encrypted session key so they can first decrypt the session key with their private key, then decrypt the file with that session key.

Having to use a session key adds more work, but it has a benefit. You can encrypt a file to multiple recipients by using a single session key, and then separately encrypt the session key to each recipient. The resulting secret message is much smaller than it would be if you had to encrypt the actual message with multiple keys.

So the steps we need to follow are:

1. Create a random session key.
2. Encrypt the plaintext (original file) with the session key.
3. Export the session key.
4. Encrypt the session key with the recipient's public key.
5. Package the encrypted session key and encrypted file together for delivery.

These steps don't have to be done strictly in sequence. For instance, step 2 could be done in parallel with steps 3 and 4. Since all the steps are done with Promises, that's possible, but to keep this example as simple as possible we'll just do the steps in order. Here's the skeleton of the needed code:

    function encrypt(plaintext, publicKey) {
        // Returns a Promise that provides its then handler
        // a Blob representing the encrypted data.
        var sessionKey, encryptedFile; // Used in two steps, so saved here for passing

        return window.crypto.subtle.generateKey(
            {name: "AES-CBC", length: 128},
            true,
            ["encrypt", "decrypt"]).
        then(saveSessionKey).     // Need this in a later (not just the next) step
        then(encryptPlaintext).
        then(saveEncryptedFile).  // Need this result in a later step
        then(exportSessionKey).
        then(encryptSessionKey).
        then(packageResults);
    }

Note the lack of a catch method. Since this returns a promise, it can defer handling errors to a catch method on the return value or even a later step in a chain. The relatively global variables sessionKey and encryptedFile are holders for intermediate values needed in later (not just immediately following) steps:

* saveSessionKey saves the session key in the variable sessionKey, and also passes it to the next step.
* encryptPlaintext takes the session key as a parameter and encrypts plaintext with that session key and a random initialization vector, passing the resulting iv and ciphertext in an array to the next step.
* saveEncryptedFile gets an array of the iv and ciphertext as a parameter, and saves them in the variable encryptedFile. It doesn't return anything.
* exportSessionKey ignores the parameters it is given, and exports the saved sessionKey to an ArrayBuffer, passing the result to the next step.
* encryptSessionKey will get the exported session key as a parameter, and encrypt it with publicKey, passing the encrypted key result to the next step.
* packageResults will use the encrypted session key it is passed as a parameter and the saved encryptedFile to produce a Blob holding all the encrypted data, and pass the Blob to the next step (the then method handler of the Promise being returned).

Now that it's broken down into parts, it isn't too complicated to build. encryptPlaintext and exportSessionKey are each essentially operations that were shown in earlier posts:

    function encryptPlaintext(sessionKey) {
        // The plaintext is in an enclosing scope, called plaintext
        var iv = window.crypto.getRandomValues(new Uint8Array(16));
        return window.crypto.subtle.encrypt({name: "AES-CBC", iv: iv}, sessionKey, plaintext).
        then(function(ciphertext) {
            return [iv, new Uint8Array(ciphertext)];
        });
    }

    function exportSessionKey() {
        // Exports the sessionKey from the enclosing scope.
        return window.crypto.subtle.exportKey('raw', sessionKey);
    }

The functions to save intermediate values in enclosing scopes are both pretty trivial:

    function saveSessionKey(key) {
        sessionKey = key;
        return key;
    }

    function saveEncryptedFile(ivAndCiphertext) {
        encryptedFile = ivAndCiphertext;
    }

Which brings us to the two meaty new parts, one dealing with crypto, one wrangling and packaging multiple pieces of data into a Blob. Encrypting the session key turns out to be pretty easy:

    function encryptSessionKey(exportedKey) {
        // Encrypts the exportedKey with the publicKey found in the enclosing scope.
        return window.crypto.subtle.encrypt({name: "RSAES-PKCS1-v1_5"}, publicKey, exportedKey);
    }

Now for the nasty part: packaging this up to provide to the recipient, who will eventually decrypt it all. At a minimum, the package has to contain the encrypted session key, iv, and ciphertext. But it should also have a lot more in it:

* The symmetric algorithm that was used to create the ciphertext.
* The public key algorithm used to create the encrypted session key.
* Some kind of identifier of the public key the session key was encrypted for, so that recipients can know which private key they need to decrypt it.
* A way to indicate which bytes of the file represent the different pieces of the package.

There are two widely used message formats that address all these issues and more: <a title="OpenPGP message format RFC 4880" href="http://tools.ietf.org/html/rfc4880" target="_blank">OpenPGP</a> and <a title="CMS message format RFC 5652" href="http://tools.ietf.org/html/rfc5652" target="_blank">Cryptographic Message Syntax (CMS)</a>. They're both pretty complex, so this post won't cover them. After all, the key pair being used is stored only as a JavaScript variable, so it's going to go away as soon as the browser is closed and any encrypted messages built with this page will then be forever inaccessible. So keep it as simple as possible: the package format will contain the encrypted session key followed immediately by the iv and ciphertext. Since it's not clear that the encrypted session key will always be the same size, it will be preceded by a 16 bit integer giving its length:

    2-byte-key-length:key-length-byte-encrypted-session-key:16-byte-iv:ciphertext

We end up with the following:

    function packageResults(encryptedKey) {
        var length = new Uint16Array([encryptedKey.byteLength]);
        return new Blob(
            [
                length,             // Always a 2 byte unsigned integer
                encryptedKey,       // "length" bytes long
                encryptedFile[0],   // 16 bytes long initialization vector
                encryptedFile[1]    // Remainder is the ciphertext
            ],
            {type: "application/octet-stream"}
        );
    }

Note that we get the 16 bit length by creating a one element array of unsigned 16 bit integers.

#Creating and Saving the Encrypted Results

The symmetric encryption example in the previous posts got their input file from an element in the page with the id *source-file*, and put a link to the result at the end of an unordered list with id *download-links*. The page containing all this code follows the same pattern.

    function encryptTheFile() {
        var sourceFile = document.getElementById("source-file").files[0];

        var reader = new FileReader();
        reader.onload = processTheFile;
        reader.readAsArrayBuffer(sourceFile);

        function processTheFile() {
            var reader = this;  // Was invoked by the reader object
            var plaintext = reader.result;
            encrypt(plaintext, keyPair.publicKey). // keyPair defined in enclosing scope
            then(function(blob) {
                var url = URL.createObjectURL(blob);
                document.getElementById("download-links").insertAdjacentHTML(
                    'beforeEnd',
                    '<li><a href="' + blobUrl + '">Encrypted file</a></li>');
            }).
            catch(function(err) {
                alert("Something went wrong encrypting: " + err.message + "\n" + err.stack);
            });
        }
    }

#Decryption

Decryption is similar to encryption, though actually a bit easier. Encryption was described more or less bottom-up; decryption will be described top-down. The click handler is identical, except that the processTheFile step is different:

    function decryptTheFile() {
        var sourceFile = document.getElementById("source-file").files[0];

        var reader = new FileReader();
        reader.onload = processTheFile;
        reader.readAsArrayBuffer(sourceFile);

        function processTheFile() {
            var reader = this;              // Invoked by the reader object
            var data = reader.result;

            var keyLength       = new Uint16Array(data, 0, 2)[0];   // First 16 bit integer
            var encryptedKey    = new Uint8Array( data, 2,              keyLength);
            var iv              = new Uint8Array( data, 2 + keyLength,  16);
            var ciphertext      = new Uint8Array( data, 2 + keyLength + 16);

            decrypt(ciphertext, iv, encryptedKey, keyPair.privateKey).
            then(function(blob) {
                var url = URL.createObjectURL(blob);
                document.getElementById("download-links").insertAdjacentHTML(
                    'beforeEnd',
                    '<li><a href="' + url + '">Decrypted file</a></li>');
            }).
            catch(function(err) {
                alert("Something went wrong decrypting: " + err.message + "\n" + err.stack);
            });
        }
    }

The first part of processing is to pull out the four parts of the file: the *keyLength*, needed only to then get the right number of bytes for the *encryptedKey*, then the *initialization vector*, and finally the *ciphertext* itself. These are all passed to the decrypt function, which returns a promise that yields a Blob to the then method handler. That handler creates a URL for the blob and puts it in the page.

The actual decryption takes only three steps:

    function decrypt(ciphertext, iv, encryptedSessionKey, privateKey) {
        return decryptKey(encryptedSessionKey, privateKey).
        then(importSessionKey).
        then(decryptCiphertext);
    }

And the three steps are themselves pretty simple, given all the background covered so far:

    function decryptKey(encryptedKey, privateKey) {
        return window.crypto.subtle.decrypt({name: "RSAES-PKCS1-v1_5"}, privateKey, encryptedKey);
    }

    function importSessionKey(keyBytes) {
        return window.crypto.subtle.importKey(
            "raw",
            keyBytes,
            {name: "AES-CBC", length: 128},
            true,
            ["encrypt", "decrypt"]
        );
    }

    function decryptCiphertext(sessionKey) {
        return window.crypto.subtle.decrypt({name: "AES-CBC", iv: iv}, sessionKey, ciphertext).
        then(function(plaintext) {
            return new Blob([new Uint8Array(plaintext)], {type: "application/octet-stream"});
        });
    }

#Summing Up

This was a long post, but it's as concise as possible. It illustrates how to use public-key cryptography to encrypt and decrypt files with the Web Cryptography API. And it is of no practical use. The key pair used goes away as soon as the page is closed, lost forever. And even if that problem were solved, this page could not interoperate with any other software. More code is needed to import and export key pairs in standard formats used by other systems, and build an encrypted file in a standard format. I hope to deal with those challenges in a future post.

The next post in this series, though, will continue to be completely useless as a practical matter. It will demonstrate digitally signing files and then verifying those digital signatures.
