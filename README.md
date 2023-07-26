# synapse-log-analyzer

This is a command-line script that parses [request log files][log_format]
produced by the Matrix Synapse server and can be used as a smarter `grep`.

## Filtering

If you were interested in requests that got a 403 error (permission denied),
the naive approach might be:

```
grep 403 /var/log/matrix-synapse/homeserver.log
```

However this could produce false positives where the characters "403" happened
to occur in some other field on the line.  Instead, you can use this script to
apply a filter match to only the HTTP response code field:

```
synapse-log-analyzer --filter 'resp_code = 403' /var/log/matrix-synapse/homeserver.log
```

You can use multiple filters to see only the lines which match all filters:

```
synapse-log-analyzer --filter 'resp_code = 403' --filter 'user = @bob:example.com' /var/log/matrix-synapse/homeserver.log
```

### Available filter types:

- String equality
  ```
  --filter 'field_name = value'
  ```

- String inequality
  ```
  --filter 'field_name != value'
  ```

- Regular expression match
  ```
  --filter 'field_name ~ pattern'
  ```

- Regular expression mismatch (i.e. include only lines which do not match the pattern)
  ```
  --filter 'field_name !~ pattern'
  ```

For more complex filtering, you may wish to use JSON output (details below) and
pipe the output into a tool like `jq`.

## Configure default log paths

Instead of typing the full pathname of the log file (or files) every time, you
can create a config file at `~/.synapse-log-analyzer.json` and define which
directory and which files you want to search by default:

```
{
    "log_dir": "/var/log/matrix-synapse",
    "log_files": [
        "homeserver.log",
        "generic1.log",
        "generic2.log",
        "generic3.log"
    ]
}
```

Now you can simply omit the filename and the default file(s) will be searched:

```
synapse-log-analyzer --filter 'resp_code = 403'
```


## Output formats

When a log line matches the filters, the default behaviour is to print the
whole line on STDOUT, but there are a number of alternatives:

- List only the fields you want:
  ```
  --fields 'user_local,resp_code,req_method,req_path'
  ```

- Define templated output to combine field values with text
  ```
  --output 'User=${user_local} Path=${req_path} Code=${resp_code}'
  ```

- Request JSON output. Either all available fields, or user-specified with via
  the `--fields` option
  ```
  --json --fields 'user_local,resp_code,req_method,req_path'
  ```

## Reusing filters

Option "bundles" allow you to configure a set of options (typically filters) that
you use regularly and give them a name.  First you would add a `"bundles"` section
to your config file:

```
    "bundles": {
        "permission_denied": [
            "--filter", "resp_code = 403",
            "--filter", "req_path !~ ^/_matrix/media"
        ],
        "json_brief": [
            "--json",
            "--fields", "user_local,resp_code,req_method,req_path"
        ]
    }
```

Then you'd use the `--bundle` (can be abbreviated to `-b`) to include one or more
bundles:

```
synapse-log-analyzer -b permission_denied -b json_brief
```

## Tallying output

The `--tally` option allows you to report a count of unique output lines rather
then reporting every line.  For example:

```
synapse-log-analyzer --filter 'req_method = POST' --filter 'req_path = /_matrix/media/r0/upload' --fields 'user' --tally
  1  @andrea:example.com
  9  @philip:example.com
 16  @megynn:example.com
```

The tallied lines are sorted from least to most numerous.

## More info

Use the `--help` option for more information about available options.

## Installation

Most servers probably have Perl installed already.  The only non-core dependency
is `JSON::MaybeXS` and its associated parser module (e.g.: `Cpanel::JSON::XS`).
On a Debian/Ubuntu server you can install these dependencies (along with the
`perl-doc` package - needed for the `--help` option) with:

```
sudo apt install libjson-maybexs-perl perl-doc
```

Alternatively the modules can be installed via the `cpan` shell, however you'll
need a C compiler installed for that.

## Performance

On a busy Synapse server the log files can be huge and parsing them using an
interpreted language (Perl in this case) can be slow.  If this is a problem in
your situation, a workaround is to use `grep` directly on the log files and
then pipe the output through `synapse-log-analyzer` to filter out false
positives and to make use of output formatting.

Alternatively, you can do essentially the same thing using the `--regex` option
to "pre-filter" lines and only do the more expensive field-by-field parsing and
filteringof the lines that match the supplied Perl regular expression.

Another approach would be to reimplement the tool in a lower-level language like
Rust. If you do that, please let me know.


[log_format]: https://matrix-org.github.io/synapse/latest/usage/administration/request_log.html
