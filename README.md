# http-digest-nv

HTTP digest access authentication lets a client prove it knows a password without sending it. The server answers a request with `401 Unauthorized` and a `WWW-Authenticate: Digest` header holding a challenge: a realm, a nonce and the algorithm to use. The client sends the request again with an `Authorization: Digest` header holding a hash of the user name, the password, the nonce and the request. It is specified in RFC 7616, which replaces RFC 2617 and extends RFC 2069. 3D printer firmware such as PrusaLink and many network cameras and routers use it.

This package reads a challenge and computes the `Authorization` header that answers it. It sends nothing: the caller's HTTP client makes both requests.

## What it is

The response is a hash, written as lower-case hexadecimal. With H the hash of the challenge's algorithm:

| Value | Definition |
| --- | --- |
| HA1 | H(username `:` realm `:` password). For an algorithm that ends in `-sess`, H(H(username `:` realm `:` password) `:` nonce `:` cnonce). |
| HA2 | H(method `:` uri) for `qop=auth`. H(method `:` uri `:` H(body)) for `qop=auth-int`. |
| response | H(HA1 `:` nonce `:` nc `:` cnonce `:` qop `:` HA2). With no `qop`, H(HA1 `:` nonce `:` HA2), the form of RFC 2069. |

`nc` is the number of requests the client has sent under the server's nonce, as eight hexadecimal digits. `cnonce` is a string the client chooses for each answer.

## Install

```
novo pkg add http-digest-nv
```

## Example

```novo
use std.bytes
use digauth
use digchallenge

fn main() [io]
    let header = "Digest realm=\"http-auth@example.org\", qop=\"auth, auth-int\", algorithm=SHA-256, nonce=\"7ypf/xlj9XXwfDPEoM4URrv/xwf94BcCAzFZH4GiTo0v\", opaque=\"FQhe/qaU925kfnzjCev0ciny7QMkPqMAFRtzCUYo5tdS\""
    match digchallenge.parse(header)
        Err(e) => println(e.message())
        Ok(c)  =>
            let request = digauth.DigestRequest {
                username: "Mufasa",
                password: "Circle of Life",
                method: "GET",
                uri: "/dir/index.html",
                cnonce: "f2/wE4q74E6zIJEtWaHKaf5wv/H5QzzpXusqGemxURZJ",
                nc: 1,
                body: bytes.zeros(0) }
            println(digauth.authorization(c, request))
```

This is the example of RFC 7616 section 3.9.1. The program prints the `Authorization` header value, whose `response` parameter is `753927fa0e85d155564e2e272a28d1802ca10daf4496794697cf8db5856cb6c1`.

## Using it with an HTTP client

1. Send the request without credentials. The server answers `401`.
2. Collect every `WWW-Authenticate` header value of the answer and call `digchallenge.best_of`. A server may send one challenge per algorithm, and `best_of` takes the strongest it can answer.
3. Make a client nonce from random bytes, set `nc` to 1, and call `digauth.authorization`.
4. Send the request again with the value as its `Authorization` header.
5. For the next request under the same nonce, add 1 to `nc` and compute the header again with the same challenge. When the server answers `401` with `stale=true` in a new challenge, the credentials were right and the nonce has expired: parse the new challenge and start again at step 3.

## What the package contains

| Module | Contents |
| --- | --- |
| `digchallenge` | `parse` reads one `Digest` challenge; `best_of` chooses among several. |
| `digauth` | `authorization` builds the header value; `response_of` computes the `response` parameter alone; `DigestRequest` carries the request and the credentials; `nc_text` and `quote` are the two small formats the header needs. |
| `digerr` | `DigestFault`, with a message and a code. |

## The rules a user needs

1. The algorithms are `MD5`, `MD5-sess`, `SHA-256` and `SHA-256-sess`, in any letter case in the challenge. The header names the algorithm in the spelling above. A challenge that names another algorithm, `SHA-512-256` included, is a fault that names it.
2. `qop` is read as a list. The answer uses `auth` when the server offers it, `auth-int` when that is all it offers, and neither when the challenge has no `qop`. A `qop` with neither value is a fault.
3. A challenge that names no algorithm is answered with no `algorithm` parameter, as the servers of RFC 2069 and RFC 2617 expect.
4. `uri` is the request target as it is written in the request line, with its query.
5. `body` is hashed for `auth-int` and ignored for `auth`. It is the entity body of the request, without any transfer encoding.
6. When the challenge has `userhash=true`, the header carries H(username `:` realm) in place of the name, and `userhash=true`. The password digest still uses the plain name.
7. A user name or a value written in quotes has each quote and backslash escaped with a backslash. A user name outside ASCII is written as it is, without the `username*` form of RFC 7616 section 3.4.
8. The parameters are written in one fixed order: `username`, `realm`, `nonce`, `uri`, `algorithm`, `response`, `opaque`, `qop`, `nc`, `cnonce`, `userhash`. A server reads them in any order.
9. `cnonce` is sent in the header and hashed as given. It is the caller's, and a client nonce from random bytes is written as hexadecimal.

## Design

This package depends on [crypto-nv](https://novo-lang.org/packages/crypto-nv) for MD5 and SHA-256. It reads no random source and no clock: the client nonce and the request count arrive as parameters, so every function is a function of its arguments.

The standard library's HTTP client, `std.http.client`, belongs to the compiler and not to a package, so this package cannot extend it. A caller sets the header with `HttpClient.with_header`, or on one request.

Two choices have another reasonable answer.

- **`auth` before `auth-int`.** RFC 7616 lets the client choose among the values the server offers. `auth-int` protects the body and costs a hash of it. A caller that wants it for every request has a server that offers only it.
- **The header is built as text.** A struct of parameters would let a caller reorder or leave out a parameter. The text is the interface a client sets as a header.

## What is not included

- **Sending a request, and reading the response of one.** The caller's HTTP client does both.
- **A server.** Checking a client's response, keeping nonces and refusing a replayed `nc` are not here.
- **`SHA-512-256`.** RFC 7616 lists it, and crypto-nv has no SHA-512/256.
- **`username*`**, the RFC 5987 form of a user name outside ASCII, and the `charset` and `domain` parameters of a challenge.
- **Basic authentication.** It is the header `Authorization: Basic` and the base64 of `user:password`.

## Tests

```bash
novo test tests/digest_tests.nv         # RFC 7616 and RFC 2617 examples, and the forms they do not reach
novo test tests/digest_edge_tests.nv    # the grammar of a challenge, its refusals, the choice among several
bash tests/coverage.sh                  # line coverage over src/, merged across the suites
```

Three response values are the RFC's own: MD5 and SHA-256 for the user Mufasa of RFC 7616 section 3.9.1, and the MD5 response of RFC 2617 section 3.5. The values for the `-sess` forms, `auth-int`, the form of RFC 2069 and a hashed user name were computed with Python's `hashlib` from the formulas of RFC 7616 section 3.4, apart from this package.

## Licence

Apache-2.0. See `LICENSE`.
