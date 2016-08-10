# NAME

Plack::Middleware::ReviseEnv - Revise request environment at will

# VERSION

This document describes Plack::Middleware::ReviseEnv version {{\[ version
\]}}.

<div>
    <a href="https://travis-ci.org/polettix/Plack-Middleware-ReviseEnv">
    <img alt="Build Status" src="https://travis-ci.org/polettix/Plack-Middleware-ReviseEnv.svg?branch=master">
    </a>

    <a href="https://www.perl.org/">
    <img alt="Perl Version" src="https://img.shields.io/badge/perl-5.8+-brightgreen.svg">
    </a>

    <a href="https://badge.fury.io/pl/Plack-Middleware-ReviseEnv">
    <img alt="Current CPAN version" src="https://badge.fury.io/pl/Plack-Middleware-ReviseEnv.svg">
    </a>

    <a href="http://cpants.cpanauthors.org/dist/Plack-Middleware-ReviseEnv">
    <img alt="Kwalitee" src="http://cpants.cpanauthors.org/dist/Plack-Middleware-ReviseEnv.png">
    </a>

    <a href="http://www.cpantesters.org/distro/P/Plack-Middleware-ReviseEnv.html?distmat=1">
    <img alt="CPAN Testers" src="https://img.shields.io/badge/cpan-testers-blue.svg">
    </a>

    <a href="http://matrix.cpantesters.org/?dist=Plack-Middleware-ReviseEnv">
    <img alt="CPAN Testers Matrix" src="https://img.shields.io/badge/matrix-@testers-blue.svg">
    </a>
</div>

# SYNOPSIS

    use Plack::Middleware::ReviseEnv;

    my $mw = Plack::Middleware::ReviseEnv->new(

       # straight value
       var1 => 'a simple, overriding value',

       # value from %ENV
       var2 => '[% ENV:USER %]',

       # value from other element in $env
       var3 => '[% env:foobar %]',

       # mix and match, values are templates actually
       var4 => 'Hey [% ENV:USER %] this is [% env:var1 %]',

       # to delete an element just "undef" it
       X_REMOVE_ME => undef,

       # overriding is the default behaviour, but you can disable it
       X_FOO => {
          value => 'Get this by default',
          override => 0,
       },

       # the key is a template too!
       '[% ENV:USER %]' => '[% ENV:HOME %]',

       # the "key" can be specified inside, ignoring the "outer" one
       IGNORED_KEY => {
          key => 'THIS IS THE KEY!',
          value => 'whatever',
       },

       # more examples and features below in example with array ref

    );

    # you can also pass the key/value pairs as a hash reference
    # associated to a key named 'revisors'. This is necessary e.g. if
    # you want to set a variable in $env with name 'app', 'opts' or
    # 'revisors'
    my $mw2 = Plack::Middleware::ReviseEnv->new(revisors => \%revisors);

    # when evaluation order or repetition is important... use an array
    # reference for 'revisors'. You can also avoid passing the external
    # key here, and just provide a sequence of hash definitions
    my $mw3 = Plack::Middleware::ReviseEnv->new(
       revisors => [
          KEY => { ... specification ... },

          # by default inexistent/undef inputs are expanded as empty
          # strings.
          {
             key => 'weird',
             value => '[% ENV:HOST %]:[% ENV:UNDEFINED %]',

             # %ENV = (HOST => 'www.example.com'); # no UNDEFINED
             # # weird => 'www.example.com:' # note trailing colon...
          },

          # you can "fail" generating a variable if something is missing,
          # so you can avoid the trailing colon above in two steps:
          {
             key => 'correct_port_spec',
             value => ':[% ENV:PORT %]',
             require_all => 1,

             # %ENV = (); # no PORT
             # # -> no "correct_port_spec" is generated in $env

             # %ENV = (PORT => 8080);
             # # correct_port_spec => ':8080'
          },
          {
             key => 'host_and_port',
             value => '[% ENV:HOST %][% env:correct_port_spec %]',

             # %ENV = (HOST => 'www.example.com'); # no PORT
             # # host_and_port => 'www.example.com'

             # %ENV = (HOST => 'www.example.com', PORT => 8080);
             # # host_and_port => 'www.example.com:8080'
          }

          # the default value is "undef" which ultimately means "do not
          # generate" or "delete if existent". You can set a different
          # one for the key and the value separately
          {
             key         => '[% ENV:USER %]',
             default_key => 'nobody',

             value         => '[% ENV:HOME %]',
             default_value => '/tmp',
          },

          # the default is applied only when the outcome is "undef", but
          # you can extend it to the empty string too. This is useful to
          # obtain the same effect of shell's test [ -z "$VAR" ] which is
          # true both for missing and empty values
          {
             key         => '[% ENV:USER %]',
             default_key => 'nobody',

             value         => '[% ENV:HOME %]',
             default_value => '/tmp',

             empty_as_default => 1,
          }

          # We can revisit the example on host/port and set defaults for
          # the missing variables, using two temporary variables that
          # will be cleared afterwards
          {
             key => '_host',
             value => '[% ENV:HOST %]',
             default_value => 'www.example.com',
             empty_as_default => 1,
          },
          {
             key => '_port',
             value => '[% ENV:PORT %]',
             default_value => '8080',
             empty_as_default => 1,
          },
          host_and_port => '[% env:_host %]:[% env:_port %]',
          _host => undef, # clear temporary variable
          _port => undef, # ditto
       ]
    );

# DESCRIPTION

This module allows you to reshape [Plack](https://metacpan.org/pod/Plack)'s `$env` that is passed along
to the sequence of _app_s, taking values from an interpolation of items
in `%ENV` and `$env`.

At the most basic level, it allows you to get selected values from the
environment and override some values in `$env` accordingly. For
example, if you want to use environment variables to configure a reverse
proxy setup, you can use the following revisor definitions:

    ...
    'psgi.url_scheme' => '[% ENV:RP_SCHEME %]',
    'HTTP_HOST'       => '[% ENV:RP_HOST   %]',
    'SCRIPT_NAME'     => '[% ENV:RP_PATH   %]',
    ...

This would basically implement the functionality provided by
[Dancer::Middleware::Rebase](https://metacpan.org/pod/Dancer::Middleware::Rebase) (without the `strip` capabilities).

Value definitions are actually templates with normal text and variables
expansions between delimiters. So, the following definition does what
you think:

    salutation => 'Hello, [% ENV:USER %], welcome [% ENV:HOME %]',

You are not limited to taking values from the environment and peek into
`$env` too:

    ...
    bar => 'baz', # no expansion in this template, just returns 'bar'
    foo => '[% env:bar %]',
    ...

As you can understand, if you want to peek at other values in `$env`
and these values are generated too, order matters! Take a look at
["Ordering Revisors"](#ordering-revisors) to avoid being biten by this, but the bottom line
is: use the array-reference form and put revisors in the order you want
them evaluated.

## Defining Revisors

There are multiple ways you can provide the definition of a revisor.
Before explaining the details, it's useful to notice that you can invoke
the constructor for `Plack::Middleware::ReviseEnv` in different ways:

    # the "hash" way, where %hash MUST NOT contain the "revisors" key
    my $mwh  = Plack::Middleware::ReviseEnv->new(%hash);

    # the "hash reference" way
    my $mwhr = Plack::Middleware::ReviseEnv->new(revisors => \%hash);

    # the "array reference" way
    my $mwar = Plack::Middleware::ReviseEnv->new(revisors => \@array);

The first two will be eventually turned into the last one by means of
["normalize\_input\_structure"](#normalize_input_structure) by simply putting the sequence of
key/value pairs in the array, _ordered by key_.

In the _array reference_ form, for each revisor you can provide:

- a single hash reference with the details on the revisor (see below for
the explaination), OR
- a string key (that we will call _external key_) and a hash reference.
If the hash reference contains the key `key` (sorry!) then the
`external key` will be ignored, otherwise it will be set corresponding
to key `key`. Example:

        foo => { value => 'ciao' }

    is interpreted as:

        { key => 'foo', value => 'ciao' }

    while:

        foo => { key => 'bar', value => 'baz' }

    is interpreted as:

        { key => 'bar', value => 'baz' }

    This is useful when you start from the _hash_ or _hash-reference_
    forms, because the _external key_ will be used for ordering revisors
    only (see ["Ordering Revisors"](#ordering-revisors));

- two strings, one for the key and one for the value. Example:

        foo => 'bar'

    is interpreted as:

        { key => 'foo', value => 'bar' }

While the normal key/value pairs should be sufficient in the general
case, to trigger more advanced features you have to pass the whole hash
reference definition for a revisors.  The hash can contain the following
keys:

- `default_key`
- `default_value`

    when the computed value for either the key or the value are undefined,
    the corresponding key is deleted. If you set a defined value, this will
    be used instead.

    Setting a default value makes sense only if either `empty_as_default`
    or `require_all` are set too; otherwise, whatever expansion will always
    yield a defined value (possibly empty).

- `empty_as_default`

    when the computed value is empty, treat it as it were undefined. This is
    a single setting for both key and value.

    It is useful if you suspect that your environment might actually contain
    a variable, but with an empty value that you want to override with a
    default.

- `esc`

    the escape character to use when parsing templates. It defaults to a
    single backslash, but you can override this with a different string as
    long as it's not empty, it does not start with a space and is different
    from both `start` and `stop` (see below) values. This might come handy
    in the (unlikely) case that you must use lots of backslashes.

- `key`

    the key that will be set in `$env`. It is a template itself, so it
    is subject to expansion and other rules explained here.

    If you set the revisor with the key/value pair style, the key will be
    used as the default value here; if you just provide a specification
    revisor via a hash reference, you MUST provide a key though.

- `override`

    boolean flag to indicate that you want to overwrite any previously
    existing value in `$env` for a specific computed key.

    It defaults to _true_, but you can set it to e.g. `0` to disable
    overriding and set the value in `$env` only if there's nothing there
    already.

- `require_all`

    boolean flag that makes an expansion _fail_ (returning `undef`) if any
    component is missing. Defaults to a false value, meaning that missing
    values are expanded as empty (but defined!) strings.

    For example, consider the following revisors:

        ...
        inexistent => undef, # this removes inexistent from $env

        set_but_awww => 'Foo: [% env:inexistent %]',

        not_set_at_all => {
           value => 'Foo: [% env:inexistent %]',
           require_all => 1,
        },
        ...

    As a final result, `$env->{set_but_empty}` ends up being
    present with value `Foo: `, while `$env->{not_set_at_all}` is
    not set or deleted if present.

    This can be also combined with `default_key` or `default_value`.

- `start`
- `stop`

    the delimiters for the expansion sections, defaulting to `[%` and `%]`
    respectively (or whatever option was set in `opts` at object creation).
    You can override them with any non-empty string.

- `value`

    the template for the value.

## Templates

Both the key and the value of a revisor are _templates_. They are
initially _parsed_ (during `prepare_app`) and later _expanded_ when
needed (i.e. during `call`).

The parsing verifies that the template adheres to the
["Template rules"](#template-rules); the expansion is explained in section
["Expansion"](#expansion).

### Template rules

Templates are a sequence of plain text and variable expansion sections.
The latter ones are delimited by a _start_ and _stop_ character
sequence. So, for example, with the default start and stop markers the
following text:

    Foo [% ENV:BAR %] baz

is interpreted as:

    plain text        'Foo '
    expansion section ' ENV:BAR '
    plain text        ' baz'

Plain text sections can contain whatever character sequences, except
(unescaped) start for a variable expansion section. If you want to
include a start sequence, prepend it with an _escape_ sequence
(defaulting to a single backslash), like this:

    Foo \[% ENV:BAR %] baz

is interpreted as:

    plain text 'Foo \[% ENV:BAR %] baz'

The _escape_ just makes the character immediately following it be
ignored during parsing, which happens in the expansion sections too. So,
suppose that you have a variable whose name contains the end sequence,
you can still use it like this:

    Foo [% env:bar \%] %] baz

is interpreted as:

    plain text        'Foo '
    expansion section ' env:bar \%] '
    plain text        ' baz'

After dividing the input template into sections, the plain text sections
are just unescaped, while the expansion sections are futher analyzed:

- the section string is trimmed while still honoring escape characters
(i.e. escaped trailing spaces are _kept_, even if it can sound crazy);
- then it is unescaped;
- then it is split into two components separated by a colon, checking that
the first part is either `ENV` or `env` (the source for the expansion)
and the second is the name of the item inside the source.

Example:

                      ' ENV:FOO\ \  '
     trimmed to   --> 'ENV:FOO\ \ '
     unescaped to --> 'ENV:FOO  '
     split to     --> 'ENV', 'FOO  '

In the example, the expansion section will be used to get the value of
item `FOO  ` (with two trailing spaces) from `%ENV`.

You can set different _start_, _stop_ and _escape_ sequences by:

- setting options `start`, `stop` and `esc` (respectively) in
configuration hash `opts` in the constructor, or
- setting options `start`, `stop` and `esc` (respectively) in the
revisor definition (this takes precedence with respect to the ones in
the `opts` for the object, of course).

### Expansion

While parsing happens once, expansion of templates happens every time
the `call` method is invoked, which should mean at each request hitting
Plack.

During expansion, text parts are passed verbatim, while expansion
sections take the value from either `%ENV` or `$env` depending on the
expansion section itself. If the corresponding value is not present or
is `undef`:

- by default the empty string is used
- if option `require_all` in the revisor definition is set to a (Perl)
true value, the whole expansion _fails_ and returns `undef` or
whatever default value has been set in `default_key` or
`default_value` for keys and values respectively.

If the expansion above yields `undef`:

- if it's the expansion of a key, it is skipped;
- if it's the expansion of a value, ["Removing Variables"](#removing-variables) applies (i.e.
the variable is not set and removed if present).

## Removing Variables

In addition to setting values, you can also remove them (e.g. suppose
that you are getting some headers and you want to silence them (e.g. for
debugging purposes). To do this, just set the corresponding key to
`undef`:

    ...
    remove_me => undef,
    ...

This actually works whenever the expanded value returns `undef`,
although this never happens by default because `undef` values in the
expansion are turned into empty strings:

    ...
    will_be_empty => '[% ENV:inexistent_value %]',
    ...

See ["Expansion"](#expansion) for making the above return `undef` (via
`require_all`) and trigger the removal of `will_be_empty`.

## Ordering Revisors

If you plan using intermediate variables for building up complex values,
you might want to switch to the _array reference_ form of the revisor
definition (see ["Defining Revisors"](#defining-revisors)), because the hash-based
alternatives require more care.

As an example, the following will NOT do what you think:

    # using plain hash way... and being BITEN HARD!
    my $me = Plack::Middleware::ReviseEnv->new(
       foo => 'FOO',
       bar => 'Hey [% env:foo %]',
    );

This is because the following array-based rendition will be used:

    [
       bar => 'Hey [% env:foo %]',
       foo => 'FOO',
    ]

i.e. `bar` will be eventually expanded _before_ `foo`. This is
because keys are used for ordering revisors when transforming to the
array-based form.

The ordering part is actually there to help you, because by default Perl
does not guarantee _any kind of order_ when you expand a hash to the
list of key/value pairs. So, at least, in this case you have some
guarantees!

So what can you do? You can take advantage of the full form for defining
a revisor, like this:

    # using plain hash way... more verbose but correct now
    my $me = Plack::Middleware::ReviseEnv->new(
       '1' => { key => foo => value => 'FOO' },
       '2' => { key => bar => value => 'Hey [% env:foo %]'},
    );

The hash keys `1` and `2` will be used to order revisors, so they are
set correctly now:

    [
       '1' => { key => foo => value => 'FOO' },
       '2' => { key => bar => value => 'Hey [% env:foo %]'},
    ]

Note that the revisor definitions already contain a `key` field, so
neither `1` nor `2` will be used to override this field, which is the
same as the following array form:

    [
       { key => foo => value => 'FOO' },
       { key => bar => value => 'Hey [% env:foo %]'},
    ]

i.e. what you were after in the first place.

Takeaway: if you can, always use the array-based form!

# METHODS

The following methods are implemented as part of the interface for a
Plack middleware. Although you can override them... there's probably
little sense in doing this!

- call
- prepare\_app

Methods described in the following subsections can be overridden or used
in derived classes, with the exception of ["new"](#new).

## **escaped\_index**

    my $i = $obj->escaped_index($template, $str, $esc, $pos);

Low-level method for finding the first unescaped occurrence of `$str`
inside `$template`, starting from `$pos` and considering `$esc` as
the escape sequence. Returns -1 if the search is unsuccessful, otherwise
the index value in the string (indexes start from 0), exactly as
`CORE::index`.

## **escaped\_trim**

    my $trimmed = $obj->escaped_trim($str, $esc);

Low-level function to trim away spaces from an escaped string. It takes
care to remove all leading spaces, and all unescaped trailing spaces
(there can be no "escaped leading spaces" because escape sequences
cannot start with a space).

Note that trimming targets only plain horizontal spaces (ASCII 0x20).

## **generate\_revisor**

    my $revisor = $obj->generate_revisor($input_definition);

Generate a revisor from an input definition.

The input definition MUST be a hash reference with fields explained in
section ["Defining Revisors"](#defining-revisors).

Returns a new revisor.

It is used by ["prepare\_app"](#prepare_app) to set the list of revisors that will be
used during expansion.

## **new**

    # Alternative 1, when %hash DOES NOT contain a "revisors" key
    my $mw_h = Plack::Middleware::ReviseEnv->new(%hash);

    # Alternative 2, "revisors" points to a hash ref
    my $mw_r = Plack::Middleware::ReviseEnv->new(
       revisors => $hash_or_array_ref, # array ref is PREFERRED
       opts     => \%hash_with_options
    )

You are not supposed to use this method directly, although these are
exactly the same parameters that you are supposed to pass e.g. to
`builder` in [Plack::Builder](https://metacpan.org/pod/Plack::Builder).

The first form is _quick and dirty_ and should be fine in most of the
simple cases, like if you just want to set a few variables taking them
from the environment (`%ENV`) and you're fine with the default options.

The second form allows you to pass options, e.g. to change the
delimiters for expansion sections, and also to define the sequence of
revisors as an array reference, which is quite important if you are
going to do fancy things (see ["Ordering Revisors"](#ordering-revisors) for example).

Available opts are:

- `esc`

    the escape sequence to use when parsing a template (see
    ["Template rules"](#template-rules)). This can be overridden on a per-revisor basis.

    Defaults to a single backslash `\`.

- `start`

    the start sequence to use when parsing a template (see
    ["Template rules"](#template-rules)). This can be overridden on a per-revisor basis.

    Defaults to string `[%`, in [Template::Toolkit](https://metacpan.org/pod/Template::Toolkit) spirit.

- `stop`

    the stop sequence to use when parsing a template (see
    ["Template rules"](#template-rules)). This can be overridden on a per-revisor basis.

    Defaults to string `%]`, in [Template::Toolkit](https://metacpan.org/pod/Template::Toolkit) spirit.

## **normalize\_input\_structure**

    my $normal = $obj->normalize_input_structure($source, $defaults);

normalizes the object internally landing you with the following fields:

- `app`
- `revisors`
- `opts`

where `revisors` is in the array form and `opts` has any missing item
initialised to the corresponding default (if not already present).

## **parse\_template**

    my $expandable = $obj->parse_template($template, $start, $stop, $esc);

applies the parsing explained in section ["Template rules"](#template-rules) and returns
an array reference of sequences of either plain text chunks, or hash
references each containing:

- `src`

    either `env` or `ENV`

- `key`

    the key to use inside the `src` for expanding a variable.

This method is quite low-level and you have to explicitly pass the
start, stop and escaping sequences, making sure they don't tread on each
other.

Used by ["generate\_revisor"](#generate_revisor).

## **unescape**

    my $text = $obj->unescape($escaped_text, $esc);

Removes the escaping sequence `$esc` from `$escaped_text`.

# BUGS AND LIMITATIONS

Report bugs either through RT or GitHub (patches welcome).

# SEE ALSO

[Plack](https://metacpan.org/pod/Plack), [Plack::Middleware::ForceEnv](https://metacpan.org/pod/Plack::Middleware::ForceEnv),
[Plack::Middleware::ReverseProxy](https://metacpan.org/pod/Plack::Middleware::ReverseProxy),
[Plack::Middleware::SetEnvFromHeader](https://metacpan.org/pod/Plack::Middleware::SetEnvFromHeader),
[Plack::Middleware::SetLocalEnv](https://metacpan.org/pod/Plack::Middleware::SetLocalEnv).

# AUTHOR

Flavio Poletti <polettix@cpan.org>

# COPYRIGHT AND LICENSE

Copyright (C) 2016 by Flavio Poletti <polettix@cpan.org>

This module is free software. You can redistribute it and/or modify it
under the terms of the Artistic License 2.0.

This program is distributed in the hope that it will be useful, but
without any warranty; without even the implied warranty of
merchantability or fitness for a particular purpose.
