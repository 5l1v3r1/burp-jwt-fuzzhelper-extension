# JWT FuzzHelper for Burp

## Purpose

JSON Web Token (JWT) support for Burp Intruder. This extension adds a payload processor for fuzzing JWT claims.

## Comparison

[JOSEPH](https://github.com/portswigger/json-web-token-attacker) and [JSON Web Tokens](https://github.com/portswigger/json-web-tokens) are two extensions that automate some common attacks and provide various views for JWTs. This  extension complements those by providing an Intruder hook for more targeted fuzzing and on-the-fly manipulation of JWTs.

## Use Cases

Example use cases may include:
1. Inserting atypical values for common claims
2. Inserting a new claim that may be processed by the application before signature validation (e.g. [CVE-2018-0114](https://www.exploit-db.com/exploits/44324/))
3. Easily iterating over a large set of payload claim values if, for example, one has obtained a signing key
4. Inserting bogus or unusually encoded strings or bad inputs. For example, those in the [Big List of Naughty Strings](https://github.com/minimaxir/big-list-of-naughty-strings)
5. Manipulation of timestamps or expirations in `iat`, `exp`, etc...
6. Fuzzing site specific implementations of `kid`.
7. Testing for denial of service conditions.
8. Classic attacks like testing for `none` type signatures, algorithmic substitution, etc...

This extension will also process JWT tokens that do not have JSON encoded payloads, which, while uncommon, is something other extensions have may have overlooked.

## Dependencies

This extension requires you to have Jython installed.

The HS* class of signature algorithms (ie. HS256, HS384, and HS512) are implemented using native Python libraries. The RS*, ES*, PS*, and None class of signatures are generated via the [pyjwt](https://pyjwt.readthedocs.io/en/latest/) and [rsa](https://pypi.python.org/pypi/rsa) libraries. Since pyjwt relies on Python `cryptography` libs and these libs cannot be installed via Jython, you will need to specify a folder for loading additional Python modules in Extender -> Options -> Python Environment. If you are not planning on making use of ES*, RS*, or PS* algorithms, you do not need `pyjwt` or `rsa`. You can find the location of your libraries with the command `python -c "import sys; print sys.path;"`.

## Installation

#### Install Python dependencies

```bash
$ pip install -r requirements.txt
```

#### Install the extension.

You can do this in the extender pane.

Extender -> Extensions -> Add -> Type: Python -> Load `burp-jwt-fuzzhelper.py`

#### Add your external modules path

The `pyjwt` libs rely on C extensions that cannot be installed via Jython. Users need to specify an external path for loading modules.

After finding your environment import paths with `python -c "import sys; print sys.path;"`, locate pyjwt. On Kali this is usually in 
some folder named `dist-packages`, on Macs in `site-packages`. You can confirm by grepping for the module (e.g. `ls /usr/lib/python2.7/dist-packages/ | grep -i pyjwt`.

Refer to [this issue](https://github.com/pinnace/burp-jwt-fuzzhelper-extension/issues/3) if you are having problems.

## Usage

### **Important**

1. You must **disable** payload encoding for the `.` character in Intruder options, or the JWT delimiters will be URL encoded.

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/payload_encoding.png" width="75%" height="75%">

### Calling the extension

You can invoke the extension in the Intruder tab via payload processor pane

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/payload_processing.png" width="65%" height="65%">

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/payload_processing_rule.png" width="65%" height="65%">

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/processing_rule.png" width="65%" height="65%">

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/invoke_processor.png" width="65%" height="65%">



## Configuration

This fuzzer uses [jq's Object Identifier-Index](https://stedolan.github.io/jq/manual/#Basicfilters) or a regular expression to select fields for fuzzing.

### Options

1. `Target Selection`: Select either the Header or the Payload portion of a JWT to fuzz
2. `JSON Selector`: Specify a filter using [jq's Object Identifier-Index](https://stedolan.github.io/jq/manual/#Basicfilters) (e.g. `.user.role`) or a regex depending on whether `Use regex as JSON selector` is checked.
  - For Object Identifier-Index selectors, a single `.` is an empty selector. If the selector is not empty and this claim does not exist, it will be created.
  - For regular expressions, the regex is passed to [`re.sub`](https://docs.python.org/2/library/re.html#re.sub). An empty selector is no character.
3. `Use regex as JSON selector`: As stated, optionally use a regex.
4. `Generate Signature`: Whether or not to generate a signature
5. `Signature Algorithm`: If `Generate Signature` is True, then use this algorithm
6. `Signing Key` : Optional signing key to paste. If using RS, ES, or PS family of algorithms, this key must be a valid signing key.
7. `Signing Key From File`: Optionally load key from file. If selected, option `Path to Signing Key` will appear. Useful if key is raw bytes and generally more reliable. Recommended.
8. `Signing Key From Command`: Optionally obtain key from external comand. If selected, option `Path to Signing Cmd` will appear. Useful if signing process is not standard or not supported. The command should expect only `encoded(header).encoded(payload)` string as an argument and print base64-encoded key to standard output.
9. `Path to Signing Key`: Path to file with the signing key. If using RS, ES, or PS family of algorithms, this key must be a valid signing key.

#### Selector Example: Selecting `alg`

If you wanted to fuzz the `alg` field, you would use "Header" for your target selection and `.alg` as your selector

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/alg_selector.png" width="55%" height="55%">

#### Selector Example: Selecting a nested claim

Given the claim:

```json
"user" : { 
       "username" : "john.doe", 
       "role" : "admin" 
    } 
```

Say you want to fuzz _role_. You would use `.user.role` as your selector. If you were using a regex, you might just use `admin`.



## Fuzzing examples

### Example 1: Fuzzing for `None`

Say you want to test if an application can be tricked into accepting `none` as a valid hashing algorithm. This vulnerability was originally discussed [here](https://auth0.com/blog/2015/03/31/critical-vulnerabilities-in-json-web-token-libraries/). You may want to try various permutations of none (e.g. `NoNe`, `nOne`, `noNe`, etc). Note that this is not the same as selecting 'None' as the Signature Algorithm, which will simply use `pyjwt`.

1. Use `.alg` as your selector
2. Strip signature from your token

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/intruder_none_censored.png" width="55%" height="55%">

3. Add your payload list to Intruder

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/none_payload.png" width="55%" height="55%">

4. Run Intruder. One can see the [JSON Web Tokens](https://github.com/portswigger/json-web-tokens) extension is also handy here

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/none_intruder.png" width="55%" height="55%">


### Example 2: Algorithmic substitution

Say you want to test if an application can be tricked into using a public key as an HMAC key.

1. Use an empty selector `.`, or try fuzzing another claim (e.g. Payload -> `.user.name`) to see if your attack has been successful.
2. Set `Generate Signature` to True
3. Select `HS256` as your signature algorithm
4. Specify the path to the public key, or paste the key in the text box (be careful with `\n`s)

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/algorithmic_confusion.png" width="55%" height="55%">


### Example 3: `kid` claim fuzzing

[Bitcoin CTF](https://bitcoinctf.com) had a challenge last year involving an improperly handled `kid` field. Here's how this extension could help you attack that.

Looking at [RFC7515](https://tools.ietf.org/html/rfc7515#section-4.1.4), we can see that the `kid` (key id) value is an optional claim field in the header section of a JWT token providing a 'hint' to the operator as to which key was used to sign the token. This is useful if multiple keys are used. Implementation itself is unspecified and up to the operator. Since the `kid` parameter is parsed before verifying the signature and implementation is up to the operator, this field presents a promising attack vector.

In the Bitcoin CTF, the `kid` field turned out to be a filename under control of the user. By specifying a CSS or JS file with known contents and manipulating the algorithm, one could generate a valid token. To test this with this fuzzer, one could do the following:

To exploit this using the fuzzer you would do the following:

1. Select the **Header** as your target and `.kid` as your selector
2. Set **Generate Signature?** to "True"
3. Select the signature algorithm, in this case HS256
4. Dump the known file contents into the **Signing Key** text field
5. Hit save

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/kid_config.png" width="75%" height="75%">

Or

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/kid_config2.png" width="75%" height="75%">


6. Add your fuzz list

<img src="https://github.com/pinnace/burp-jwt-extension-images/blob/master/kid_payload.png" width="75%" height="75%">

7. Run Intruder
8. Victory dance

## Issues or feature requests

PRs welcome. Please open an issue if you have encountered a bug or want to see additional features added. Currently, the fuzzer only supports fuzzing a single claim at a time. If there is interest in supporting functions like [Pitchfork](https://portswigger.net/burp/help/intruder_positions) intruder positions, I may consider adding. I am also exploring adding an IScannerInsertionPoint for the active scanner.

This extension has been submitted to Portswigger for inclusion in the BApp store pending review.
