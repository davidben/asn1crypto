# asn1crypto

A fast, pure Python library for parsing and serializing ASN.1 structures.

 - [Features](#features)
 - [Why Another Python ASN.1 Library?](#why-another-python-asn1-library)
 - [Related Crypto Libraries](#related-crypto-libraries)
 - [Current Release](#current-release)
 - [Dependencies](#dependencies)
 - [Installation](#installation)
 - [License](#license)
 - [Documentation](#documentation)
 - [Continuous Integration](#continuous-integration)
 - [Testing](#testing)
 - [Development](#development)
 - [CI Tasks](#ci-tasks)

[![GitHub Actions CI](https://github.com/wbond/asn1crypto/workflows/CI/badge.svg)](https://github.com/wbond/asn1crypto/actions?workflow=CI)
[![CircleCI](https://circleci.com/gh/wbond/asn1crypto.svg?style=shield)](https://circleci.com/gh/wbond/asn1crypto)
[![PyPI](https://img.shields.io/pypi/v/asn1crypto.svg)](https://pypi.org/project/asn1crypto/)

## Features

In addition to an ASN.1 BER/DER decoder and DER serializer, the project includes
a bunch of ASN.1 structures for use with various common cryptography standards:

| Standard               | Module                                      | Source                                                                                                                 |
| ---------------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| X.509                  | [`asn1crypto.x509`](asn1crypto/x509.py)     | [RFC 5280](https://tools.ietf.org/html/rfc5280)                                                                        |
| CRL                    | [`asn1crypto.crl`](asn1crypto/crl.py)       | [RFC 5280](https://tools.ietf.org/html/rfc5280)                                                                        |
| CSR                    | [`asn1crypto.csr`](asn1crypto/csr.py)       | [RFC 2986](https://tools.ietf.org/html/rfc2986), [RFC 2985](https://tools.ietf.org/html/rfc2985)                       |
| OCSP                   | [`asn1crypto.ocsp`](asn1crypto/ocsp.py)     | [RFC 6960](https://tools.ietf.org/html/rfc6960)                                                                        |
| PKCS#12                | [`asn1crypto.pkcs12`](asn1crypto/pkcs12.py) | [RFC 7292](https://tools.ietf.org/html/rfc7292)                                                                        |
| PKCS#8                 | [`asn1crypto.keys`](asn1crypto/keys.py)     | [RFC 5208](https://tools.ietf.org/html/rfc5208)                                                                        |
| PKCS#1 v2.1 (RSA keys) | [`asn1crypto.keys`](asn1crypto/keys.py)     | [RFC 3447](https://tools.ietf.org/html/rfc3447)                                                                        |
| DSA keys               | [`asn1crypto.keys`](asn1crypto/keys.py)     | [RFC 3279](https://tools.ietf.org/html/rfc3279)                                                                        |
| Elliptic curve keys    | [`asn1crypto.keys`](asn1crypto/keys.py)     | [SECG SEC1 V2](http://www.secg.org/sec1-v2.pdf)                                                                        |
| PKCS#3 v1.4            | [`asn1crypto.algos`](asn1crypto/algos.py)   | [PKCS#3 v1.4](ftp://ftp.rsasecurity.com/pub/pkcs/ascii/pkcs-3.asc)                                                        |
| PKCS#5 v2.1            | [`asn1crypto.algos`](asn1crypto/algos.py)   | [PKCS#5 v2.1](http://www.emc.com/collateral/white-papers/h11302-pkcs5v2-1-password-based-cryptography-standard-wp.pdf) |
| CMS (and PKCS#7)       | [`asn1crypto.cms`](asn1crypto/cms.py)       | [RFC 5652](https://tools.ietf.org/html/rfc5652), [RFC 2315](https://tools.ietf.org/html/rfc2315)                       |
| TSP                    | [`asn1crypto.tsp`](asn1crypto/tsp.py)       | [RFC 3161](https://tools.ietf.org/html/rfc3161)                                                                        |
| PDF signatures         | [`asn1crypto.pdf`](asn1crypto/pdf.py)       | [PDF 1.7](http://wwwimages.adobe.com/content/dam/Adobe/en/devnet/pdf/pdfs/PDF32000_2008.pdf)                           |

## Why Another Python ASN.1 Library?

Python has long had the [pyasn1](https://pypi.org/project/pyasn1/) and
[pyasn1_modules](https://pypi.org/project/pyasn1-modules/) available for
parsing and serializing ASN.1 structures. While the project does include a
comprehensive set of tools for parsing and serializing, the performance of the
library can be very poor, especially when dealing with bit fields and parsing
large structures such as CRLs.

After spending extensive time using *pyasn1*, the following issues were
identified:

 1. Poor performance
 2. Verbose, non-pythonic API
 3. Out-dated and incomplete definitions in *pyasn1-modules*
 4. No simple way to map data to native Python data structures
 5. No mechanism for overridden universal ASN.1 types

The *pyasn1* API is largely method driven, and uses extensive configuration
objects and lowerCamelCase names. There were no consistent options for
converting types of native Python data structures. Since the project supports
out-dated versions of Python, many newer language features are unavailable
for use.

Time was spent trying to profile issues with the performance, however the
architecture made it hard to pin down the primary source of the poor
performance. Attempts were made to improve performance by utilizing unreleased
patches and delaying parsing using the `Any` type. Even with such changes, the
performance was still unacceptably slow.

Finally, a number of structures in the cryptographic space use universal data
types such as `BitString` and `OctetString`, but interpret the data as other
types. For instance, signatures are really byte strings, but are encoded as
`BitString`. Elliptic curve keys use both `BitString` and `OctetString` to
represent integers. Parsing these structures as the base universal types and
then re-interpreting them wastes computation.

*asn1crypto* uses the following techniques to improve performance, especially
when extracting one or two fields from large, complex structures:

 - Delayed parsing of byte string values
 - Persistence of original ASN.1 encoded data until a value is changed
 - Lazy loading of child fields
 - Utilization of high-level Python stdlib modules

While there is no extensive performance test suite, the
`CRLTests.test_parse_crl` test case was used to parse a 21MB CRL file on a
late 2013 rMBP. *asn1crypto* parsed the certificate serial numbers in just
under 8 seconds. With *pyasn1*, using definitions from *pyasn1-modules*, the
same parsing took over 4,100 seconds.

For smaller structures the performance difference can range from a few times
faster to an order of magnitude or more.

## Related Crypto Libraries

*asn1crypto* is part of the modularcrypto family of Python packages:

 - [asn1crypto](https://github.com/wbond/asn1crypto)
 - [oscrypto](https://github.com/wbond/oscrypto)
 - [csrbuilder](https://github.com/wbond/csrbuilder)
 - [certbuilder](https://github.com/wbond/certbuilder)
 - [crlbuilder](https://github.com/wbond/crlbuilder)
 - [ocspbuilder](https://github.com/wbond/ocspbuilder)
 - [certvalidator](https://github.com/wbond/certvalidator)

## Current Release

1.4.0 - [changelog](changelog.md)

## Dependencies

Python 2.6, 2.7, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9 or pypy. *No third-party
packages required.*

## Installation

```bash
pip install asn1crypto
```

## License

*asn1crypto* is licensed under the terms of the MIT license. See the
[LICENSE](LICENSE) file for the exact license text.

## Documentation

The documentation for *asn1crypto* is composed of tutorials on basic usage and
links to the source for the various pre-defined type classes.

### Tutorials

 - [Universal Types with BER/DER Decoder and DER Encoder](docs/universal_types.md)
 - [PEM Encoder and Decoder](docs/pem.md)

### Reference

 - [Universal types](asn1crypto/core.py), `asn1crypto.core`
 - [Digest, HMAC, signed digest and encryption algorithms](asn1crypto/algos.py), `asn1crypto.algos`
 - [Private and public keys](asn1crypto/keys.py), `asn1crypto.keys`
 - [X509 certificates](asn1crypto/x509.py), `asn1crypto.x509`
 - [Certificate revocation lists (CRLs)](asn1crypto/crl.py), `asn1crypto.crl`
 - [Online certificate status protocol (OCSP)](asn1crypto/ocsp.py), `asn1crypto.ocsp`
 - [Certificate signing requests (CSRs)](asn1crypto/csr.py), `asn1crypto.csr`
 - [Private key/certificate containers (PKCS#12)](asn1crypto/pkcs12.py), `asn1crypto.pkcs12`
 - [Cryptographic message syntax (CMS, PKCS#7)](asn1crypto/cms.py), `asn1crypto.cms`
 - [Time stamp protocol (TSP)](asn1crypto/tsp.py), `asn1crypto.tsp`
 - [PDF signatures](asn1crypto/pdf.py), `asn1crypto.pdf`

## Continuous Integration

Various combinations of platforms and versions of Python are tested via:

 - [macOS, Linux, Windows](https://github.com/wbond/asn1crypto/actions/workflows/ci.yml) via GitHub Actions
 - [arm64](https://circleci.com/gh/wbond/asn1crypto) via CircleCI

## Testing

Tests are written using `unittest` and require no third-party packages.

Depending on what type of source is available for the package, the following
commands can be used to run the test suite.

### Git Repository

When working within a Git working copy, or an archive of the Git repository,
the full test suite is run via:

```bash
python run.py tests
```

To run only some tests, pass a regular expression as a parameter to `tests`.

```bash
python run.py tests ocsp
```

### PyPi Source Distribution

When working within an extracted source distribution (aka `.tar.gz`) from
PyPi, the full test suite is run via:

```bash
python setup.py test
```

### Package

When the package has been installed via pip (or another method), the package
`asn1crypto_tests` may be installed and invoked to run the full test suite:

```bash
pip install asn1crypto_tests
python -m asn1crypto_tests
```

## Development

To install the package used for linting, execute:

```bash
pip install --user -r requires/lint
```

The following command will run the linter:

```bash
python run.py lint
```

Support for code coverage can be installed via:

```bash
pip install --user -r requires/coverage
```

Coverage is measured by running:

```bash
python run.py coverage
```

To change the version number of the package, run:

```bash
python run.py version {pep440_version}
```

To install the necessary packages for releasing a new version on PyPI, run:

```bash
pip install --user -r requires/release
```

Releases are created by:

 - Making a git tag in [PEP 440](https://www.python.org/dev/peps/pep-0440/#examples-of-compliant-version-schemes) format
 - Running the command:

   ```bash
   python run.py release
   ```

Existing releases can be found at https://pypi.org/project/asn1crypto/.

## CI Tasks

A task named `deps` exists to download and stage all necessary testing
dependencies. On posix platforms, `curl` is used for downloads and on Windows
PowerShell with `Net.WebClient` is used. This configuration sidesteps issues
related to getting pip to work properly and messing with `site-packages` for
the version of Python being used.

The `ci` task runs `lint` (if flake8 is available for the version of Python) and
`coverage` (or `tests` if coverage is not available for the version of Python).
If the current directory is a clean git working copy, the coverage data is
submitted to codecov.io.

```bash
python run.py deps
python run.py ci
```
