# Derivation Outputs and Types of Derivations

As stated on the [main pages on derivations](./index.md#store-derivation),
a derivation produces [store objects], which are known as the *outputs* of the derivation.
Indeed, the entire point of derivations is to produce these outputs, and to reliably and reproducably produce these derivations each time the derivation is run.

One of the parts of a derivation is its *outputs specification*, which specifies certain information about the outputs the derivation produces when run.
The outputs specification is a map, from names to specifications for individual outputs.
The topics covered in those specifications are:

- whether the output is [content-addressed](@docroot@/store/store-object/content-address.md) [input-addressed](#TODO)

- if the content is content-addressed, how is it content addressed

- if the content is content-addressed, what is its content address (what is its [store path])

## Output Names {#outputs}

Output names can be any string which is also a valid [store path] name.
The name mapped to each output specification is not actually the name of the output.
In the general case, the output store object has name `derivationName + "-" + outputSpecName`, not any other metadata about it.
However, an output spec named "out" describes and output store object whose name is just the derivation name.

> **Example**
>
> A derivation is named `hello`, and has two outputs, `out`, and `dev`
>
> - The derivation's path will be: `/nix/store/<hash>-hello.drv`.
>
> - The store path of `out` will be: `/nix/store/<hash>-hello`.
>
> - The store path of `dev` will be: `/nix/store/<hash>-hello-dev`.

The outputs are the derivations are the [store objects][store object] it is obligated to produce.

> **Note**
>
> The formal terminology here is somewhat at adds with everyday communication in the Nix community today.
> "output" in casual usage tends to refer to either to the actual output store object, or the notional output spec, depending on context.
>
> For example "hello's `dev` output" means the store object referred to by the store path `/nix/store/<hash>-hello-dev`.
> It is unusual to call this the "`hello-dev` output", even though `hello-dev` is the actual name of that store object.

## Content-addressing derivation outputs

The content-addressing of an output is the same as the content-addressing of any store object, and thus is as already described [earlier in the store chapter](@docroot@/store/store-object/content-address.md).
It only depends on the store-object itself, not any other metadata about it.
In other words, a store object will be content-addressed the same way regardless of whether it was manually inserted into the store, outputted by some derivation, or outputted by a some other derivation.

### Fixed-output content-addressing

These attributes declare that the derivation is a so-called *fixed-output derivation* (FOD), which means that a cryptographic hash of the output is already known in advance.

As opposed to regular derivations, the [`builder`] executable of a fixed-output derivation has access to the network.
Nix computes a cryptographic hash of its output and compares that to the hash declared with these attributes.
If there is a mismatch, the derivation fails.

The rationale for fixed-output derivations is derivations such as
those produced by the `fetchurl` function. This function downloads a
file from a given URL. To ensure that the downloaded file has not
been modified, the caller must also specify a cryptographic hash of
the file. For example,

```nix
fetchurl {
  url = "http://ftp.gnu.org/pub/gnu/hello/hello-2.1.1.tar.gz";
  sha256 = "1md7jsfd8pa45z73bz1kszpp01yw6x5ljkjk2hx7wl800any6465";
}
```

It sometimes happens that the URL of the file changes, e.g., because
servers are reorganised or no longer available. We then must update
the call to `fetchurl`, e.g.,

```nix
fetchurl {
  url = "ftp://ftp.nluug.nl/pub/gnu/hello/hello-2.1.1.tar.gz";
  sha256 = "1md7jsfd8pa45z73bz1kszpp01yw6x5ljkjk2hx7wl800any6465";
}
```

If a `fetchurl` derivation was treated like a normal derivation, the
output paths of the derivation and *all derivations depending on it*
would change. For instance, if we were to change the URL of the
Glibc source distribution in Nixpkgs (a package on which almost all
other packages depend) massive rebuilds would be needed. This is
unfortunate for a change which we know cannot have a real effect as
it propagates upwards through the dependency graph.

For fixed-output derivations, on the other hand, the name of the
output path only depends on the `outputHash*` and `name` attributes,
while all other attributes are ignored for the purpose of computing
the output path. (The `name` attribute is included because it is
part of the path.)

As an example, here is the (simplified) Nix expression for
`fetchurl`:

```nix
{ stdenv, curl }: # The curl program is used for downloading.

{ url, sha256 }:

stdenv.mkDerivation {
  name = baseNameOf (toString url);
  builder = ./builder.sh;
  buildInputs = [ curl ];

  # This is a fixed-output derivation; the output must be a regular
  # file with SHA256 hash sha256.
  outputHashMode = "flat";
  outputHashAlgo = "sha256";
  outputHash = sha256;

  inherit url;
}
```

If a fetchurl derivation followed the normal translation scheme, the output paths of the derivation and *all derivations depending on it* would change.
For instance, if we were to change the URL of the Glibc source distribution—a component on which almost all other components depend—massive rebuilds will ensue.
This is unfortunate for a change which we know cannot have a real effect as it propagates upwards through the dependency graph.

Fixed-output derivations solve this problem by allowing a derivation to state to Nix that its output will hash to a specific value.
When Nix builds the derivation, it will hash the output and check that the hash corresponds to the declared value.
If there is a hash mismatch, the build fails and the output is not registered as valid.
For fixed-output derivations, the computation of the output path only depends on the declared hash and hash algorithm, *not* on any other attributes of the derivation.

### (Floating) Content-Addressing

> **Warning**
> This is part of an [experimental feature](@docroot@/development/experimental-features.md).
>
> To use this type of output addressing, you must enable the
> [`ca-derivations`][xp-feature-ca-derivations] experimental feature.
> For example, in [nix.conf](@docroot@/command-ref/conf-file.md) you could add:
>
> ```
> extra-experimental-features = ca-derivations
> ```

It also implicitly requires that the machine to build the derivation must have the `ca-derivations` [system feature](@docroot@/command-ref/conf-file.md#conf-system-features).

## Input-addressing derivation outputs

"Input addressing", however, means the address the store object by the *way it was made* rather than *what it is*.
That is to say, with input addressing the outputs store path is a function not of the output itself, but the derivation that produced it.
Even if two store paths have the same contents, if they are produced in different ways, and one is input-addressed, then they will have different store paths, and thus guaranteed to not be the same store object.

### Modulo fixed-output derivations

**TODO hash derivation modulo.**

So how do we compute the hash part of the output path of a derivation?
This is done by the function `hashDrv`, shown in Figure 5.10.
It distinguishes between two cases.
If the derivation is a fixed-output derivation, then it computes a hash over just the `outputHash` attributes.

If the derivation is not a fixed-output derivation, we replace each element in the derivation’s inputDrvs with the result of a call to `hashDrv` for that element.
(The derivation at each store path in `inputDrvs` is converted from its on-disk ATerm representation back to a `StoreDrv` by the function `parseDrv`.) In essence, `hashDrv` partitions store derivations into equivalence classes, and for hashing purpose it replaces each store path in a derivation graph with its equivalence class.

The recursion in Figure 5.10 is inefficient:
it will call itself once for each path by which a subderivation can be reached, i.e., `O(V k)` times for a derivation graph with `V` derivations and with out-degree of at most `k`.
In the actual implementation, memoisation is used to reduce this to `O(V + E)` complexity for a graph with E edges.

[xp-feature-ca-derivations]: @docroot@/development/experimental-features.md#xp-feature-ca-derivations
[xp-feature-dynamic-derivations]: @docroot@/development/experimental-features.md#xp-feature-dynamic-derivations
[xp-feature-git-hashing]: @docroot@/development/experimental-features.md#xp-feature-git-hashing
